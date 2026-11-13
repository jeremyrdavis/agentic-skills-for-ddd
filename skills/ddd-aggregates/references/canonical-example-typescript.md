# Canonical Example — `Order` Aggregate Root (TypeScript)

Load this file when creating or substantially reshaping an aggregate in
TypeScript. For placement and review decisions, the Core Rules and Quick
Reference tables in SKILL.md are sufficient on their own.

The `Order` aggregate lives in `orders/domain/order.ts`. **Plain object — zero
ORM/driver/framework imports.** Exercises every Core Rule; comments cite rule
numbers.

## The aggregate

```typescript
// orders/domain/order.ts
import { CustomerId } from "./customer-id";
import {
  DomainEvent,
  OrderCancelledEvent,
  OrderFulfilledEvent,
  OrderPlacedEvent,
} from "./events";
import { OrderCancellationNotAllowedError } from "./errors";
import { Money } from "./money";
import { OrderId } from "./order-id";
import { OrderLine } from "./order-line";

export type OrderStatus = "PLACED" | "FULFILLED" | "CANCELLED";

export class Order {
  private pendingEvents: DomainEvent[] = [];

  /** Private — used only by the factory and rehydrate (Rule 4). */
  private constructor(
    private readonly _id: string,              // identity (Rule 9); raw UUID string internally
    private _version: number,                  // optimistic-lock cursor (Rule 10) — plain number
    private readonly _customerId: CustomerId,  // reference by ID, not object (Rules 8, 9)
    private readonly _lines: OrderLine[],
    private _status: OrderStatus,
    private _total: Money,                     // value object as a plain field (Rule 11)
    private readonly _placedAt: Date,          // the clock is passed in by the use case, never read here
  ) {}

  /**
   * Named factory — the only way domain code brings a new Order into existence.
   * Enforces creation invariants (Rule 5) and raises OrderPlacedEvent (Rule 12).
   */
  static place(id: OrderId, customerId: CustomerId, lines: OrderLine[], placedAt: Date): Order {
    if (lines.length === 0) {
      throw new Error("an order must have at least one line");
    }
    const currency = lines[0].unitPrice.currency;
    if (!lines.every((l) => l.unitPrice.currency === currency)) {
      throw new Error("all lines must share a currency");
    }

    const total = lines
      .map((l) => l.lineTotal())
      .reduce((sum, t) => sum.plus(t), Money.zero(currency));

    const order = new Order(id.value, 0, customerId, [...lines], "PLACED", total, placedAt);
    order.pendingEvents.push(new OrderPlacedEvent(id, customerId)); // Rule 12
    return order;
  }

  /**
   * Reconstruction path for the persistence layer (Rule 6). Trusts inputs that
   * already passed validation when first persisted; does NOT re-emit
   * OrderPlacedEvent. Only the repository implementation in
   * <bc>/infrastructure calls this (covered by ddd-persistence).
   */
  static rehydrate(id: string, customerId: CustomerId, lines: OrderLine[],
                   status: OrderStatus, total: Money, placedAt: Date, version: number): Order {
    return new Order(id, version, customerId, [...lines], status, total, placedAt);
  }

  /** Transition method — enforces the state-machine invariant (Rule 7). */
  cancel(): void {
    if (this._status === "FULFILLED") {
      throw new OrderCancellationNotAllowedError(new OrderId(this._id));
    }
    this._status = "CANCELLED";
    this.pendingEvents.push(new OrderCancelledEvent(new OrderId(this._id)));
  }

  markFulfilled(): void {
    if (this._status !== "PLACED") {
      throw new Error("only PLACED orders can be fulfilled");
    }
    this._status = "FULFILLED";
    this.pendingEvents.push(new OrderFulfilledEvent(new OrderId(this._id)));
  }

  // Read accessors — only what callers need (Rule 3). No setters (Rule 2).
  get id(): OrderId { return new OrderId(this._id); }
  get status(): OrderStatus { return this._status; }
  get total(): Money { return this._total; }
  get customerId(): CustomerId { return this._customerId; }
  get lines(): readonly OrderLine[] { return [...this._lines]; }  // defensive copy
  get placedAt(): Date { return this._placedAt; }
  get version(): number { return this._version; }

  /** Drained by the application service after persist; published after commit (Rule 12). */
  pullPendingEvents(): DomainEvent[] {
    const drained = [...this.pendingEvents];
    this.pendingEvents = [];
    return drained;
  }
}
```

## Companion types

Also plain, also persistence-free (full coverage in `ddd-value-objects` and
`ddd-domain-events`):

```typescript
// orders/domain/order-id.ts, customer-id.ts, order-line.ts, events.ts
export class OrderId {
  constructor(readonly value: string) {
    if (!value) throw new Error("OrderId requires a value");
  }
}

export class CustomerId {
  constructor(readonly value: string) {
    if (!value) throw new Error("CustomerId requires a value");
  }
}

export class OrderLine {
  constructor(
    readonly productSku: string,
    readonly quantity: number,
    readonly unitPrice: Money,
  ) {
    if (quantity <= 0) throw new Error("quantity must be positive");
  }
  lineTotal(): Money {
    return this.unitPrice.times(this.quantity);
  }
}

// Events: past tense, *Event suffix (per ddd-foundations). Every event carries
// eventId + occurredAt, assigned once when raised (ddd-domain-events Rule 3).
export interface DomainEvent {
  readonly eventId: string;
  readonly occurredAt: Date;       // when it happened — not an ordering key; order on version if a consumer needs order
}

abstract class BaseDomainEvent implements DomainEvent {
  readonly eventId = crypto.randomUUID();   // Web Crypto global (Node >= 19)
  readonly occurredAt = new Date();
}

export class OrderPlacedEvent extends BaseDomainEvent {
  constructor(readonly orderId: OrderId, readonly customerId: CustomerId) { super(); }
}
export class OrderCancelledEvent extends BaseDomainEvent {
  constructor(readonly orderId: OrderId) { super(); }
}
export class OrderFulfilledEvent extends BaseDomainEvent {
  constructor(readonly orderId: OrderId) { super(); }
}
```

## The persistence mirror (lives in `<bc>/infrastructure`)

The aggregate has no idea this schema exists. Sketch only — full coverage in
`ddd-persistence`:

```prisma
// prisma/schema.prisma — the conventional location; `prisma generate` makes the client
model Order {
  id            String  @id @db.Uuid
  version       Int                             // the lock column (Rule 10)
  customerId    String  @db.Uuid @map("customer_id")
  status        String
  totalAmount   Decimal @map("total_amount")    // Money split into columns — repository's call
  totalCurrency String  @map("total_currency")
  placedAt      DateTime @map("placed_at")

  @@map("orders")
}
```

```typescript
// orders/infrastructure/order-mapping.ts — the generated row type IS the mirror
import type { Order as OrderRow } from "@prisma/client";

export function toDomain(row: OrderRow, lines: OrderLine[]): Order {
  return Order.rehydrate(row.id, new CustomerId(row.customerId), lines,
    row.status as OrderStatus,
    new Money(Number(row.totalAmount), row.totalCurrency), row.placedAt, row.version);
}
```

**Optimistic locking mechanics:** Prisma has no built-in version annotation, so
the repository implementation does the compare-and-set explicitly —
`updateMany({ where: { id, version: loadedVersion }, data: { ...fields, version: { increment: 1 } } })`
— and if the returned `count` is 0, throws a domain-neutral
`ConcurrencyConflictError`; the api layer's error-mapping middleware turns it
into **409 Conflict**.

## What this demonstrates

- **Plain object (Rule 1):** imports only the BC's own domain types — no
  `@prisma/client`, `pg`, or `express`.
- **No setters, no public mutable fields (Rules 2, 3):** state is `private`
  (with `readonly` where immutable); read access via getters; only domain
  methods mutate.
- **Factory enforces creation invariants (Rules 4, 5):** empty lines and mixed
  currencies throw before the aggregate exists.
- **Rehydrate path (Rule 6):** the repository reconstructs via `rehydrate(...)`;
  no `OrderPlacedEvent` re-emitted; domain code never calls it.
- **Cross-aggregate reference by ID (Rule 8):** `CustomerId`, never a `Customer` field.
- **Typed identity (Rule 9):** `OrderId`/`CustomerId` classes at the API surface
  (nominal typing — a structural `{ value: string }` alias would let the two IDs
  swap silently); raw UUID string internally.
- **Plain `number` version (Rule 10):** the Prisma model carries the `version`
  column and the repository does the compare-and-set; the aggregate carries
  `_version`.
- **Value object as plain field (Rule 11):** `_total: Money`, no schema coupling.
- **Events (Rule 12):** `*Event` suffix, past tense, raised inside, drained via
  `pullPendingEvents()`, published after commit. Each carries its id and time, assigned once at the raising site (`ddd-domain-events` Rule 3).

## TypeScript idioms

- `private constructor` works in TypeScript — the compiler enforces Rule 4, same
  as Java.
- Use classes (not plain object literals) for typed IDs so `OrderId` and
  `CustomerId` are nominally distinct; with type aliases, structural typing
  makes them interchangeable and defeats Rule 9.
- Invariant unit tests are pure Node: no database, no Express app —
  `Order.place(...)` and `order.cancel()` are directly testable.
- Domain purity: no `@prisma/client`, `pg`, or `express` imports under
  `orders/domain/` — enforced by the ts-arch rules from `ddd-foundations`
  (the generated client is confined to `infrastructure/` by the same rules).
