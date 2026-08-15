# Canonical Example — Repository, Mirror, and Use-Case-Optimal Query (TypeScript)

Load this file when implementing a repository or adding a non-trivial query in
TypeScript. For placement and review decisions, the Core Rules and Quick
Reference tables in SKILL.md are sufficient on their own. Comments cite rule
numbers.

## 1. The interface — `<bc>/domain`, ubiquitous language only

```typescript
// orders/domain/order-repository.ts
import { Order } from "./order";
import { OrderId } from "./order-id";
import { OrderPage } from "./order-page";

export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;              // aggregate out (Rule 2)
  persist(order: Order): Promise<void>;                       // whole aggregate (Rule 6)
  remove(id: OrderId): Promise<void>;
  findRecent(page: number, size: number): Promise<OrderPage>; // use-case-optimal VO (Rules 9, 10)
}
```

## 2. The purpose-built query VOs — `<bc>/domain`, per `ddd-value-objects`

Domain-specific, immutable, equality by value — explicitly **not** DTOs (Rule 9).

```typescript
// orders/domain/order-page.ts
import { CustomerId } from "./customer-id";
import { Money } from "./money";
import { OrderStatus } from "./order";
import { OrderId } from "./order-id";

export class OrderSummary {
  constructor(
    readonly id: OrderId,
    readonly customerId: CustomerId,
    readonly total: Money,
    readonly status: OrderStatus,
  ) {}
}

export class OrderPage {
  readonly items: readonly OrderSummary[];
  constructor(items: OrderSummary[],
              readonly totalCount: number,
              readonly page: number,
              readonly size: number) {
    this.items = Object.freeze([...items]);   // defensive immutability (Rule 9)
  }
}
```

## 3. The implementation — `<bc>/infrastructure`, same name (Rule 3)

```typescript
// orders/infrastructure/order-repository.ts
import { PrismaClient } from "@prisma/client";   // generated client — ONLY under infrastructure/
import { ConcurrencyConflictError } from "../domain/errors";
import { Order } from "../domain/order";
import { OrderId } from "../domain/order-id";
import { OrderPage } from "../domain/order-page";
import { toDomain, toRow, toSummary } from "./order-mapping";

// Same name as the domain interface — the directory is the distinguisher.
// TypeScript is structural, so no `implements` (and no import collision) is
// needed: satisfaction is verified wherever the object meets a parameter
// typed as the domain interface. No technology name in the class name (Rule 3).
export class OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({ where: { id: id.value } });
    return row ? toDomain(row) : null;                       // rehydrate inside (Rule 5)
  }

  async persist(order: Order): Promise<void> {
    const data = toRow(order);
    const result = await this.prisma.order.updateMany({      // compare-and-set (Rule 7)
      where: { id: data.id, version: order.version },
      data: { ...data, version: { increment: 1 } },
    });
    if (result.count === 0) {
      const exists = await this.prisma.order.findUnique({ where: { id: data.id } });
      if (exists) throw new ConcurrencyConflictError(id);    // stale write → domain-neutral error
      await this.prisma.order.create({ data });              // first save
    }
  }

  async findRecent(page: number, size: number): Promise<OrderPage> {  // Vernon query (Rules 9, 10)
    const [rows, total] = await this.prisma.$transaction([
      this.prisma.order.findMany({
        orderBy: { placedAt: "desc" }, skip: page * size, take: size,
      }),
      this.prisma.order.count(),
    ]);
    return new OrderPage(rows.map(toSummary), total, page, size);
  }
}
```

The mirror is the **generated Prisma row types** plus the mapping module
(`order-mapping.ts`: `toDomain` / `toRow` / `toSummary`), per `ddd-aggregates`'
mirror table — nothing outside the implementation imports `@prisma/client`
(Rule 4).

**Conflict surfacing (Rule 7):** the zero-count compare-and-set result becomes
the domain-neutral `ConcurrencyConflictError`; the api layer's error-mapping
middleware turns it into **409 Conflict**. The repository never retries — the
use case decides what a conflict means.

## 4. An atomic operation — from the Positive Thoughts spec (Rule 8)

`thumbsUp` has no invariant, so load-mutate-persist would be a lost-update
hazard. The interface gets an intention-revealing method; the implementation is
one atomic update:

```typescript
// thoughts/domain — on ThoughtRepository:
incrementThumbsUp(id: ThoughtId): Promise<void>;

// thoughts/infrastructure — in the implementation:
async incrementThumbsUp(id: ThoughtId): Promise<void> {
  await this.prisma.thought.update({
    where: { id: id.value },
    data: { thumbsUp: { increment: 1 } },                    // atomic under concurrency
  });
}
```

## What this demonstrates

- **Interface in domain vocabulary (Rule 2):** typed IDs in, aggregates and
  domain VOs out; zero `@prisma/client` imports in `<bc>/domain`.
- **Same-name implementation (Rule 3):** the directory is the distinguisher;
  structural typing verifies satisfaction at the composition site.
- **Translation is the whole job (Rule 5):** `toDomain` calls
  `Order.rehydrate(...)`; VO constructor validation re-runs on the way in.
- **Locking (Rule 7):** explicit `updateMany` compare-and-set with a
  zero-count check → domain-neutral conflict → 409.
- **Atomic op (Rule 8):** Prisma's `{ increment: 1 }` behind a named method.
- **Vernon query (Rules 9, 10):** `OrderPage`/`OrderSummary` are frozen domain
  classes — not DTOs; the application service maps them onward if the api
  needs a different shape.

## TypeScript idioms

- `readonly OrderSummary[]` plus `Object.freeze` is the class-based spelling of
  an immutable collection field; `readonly` alone is compile-time only.
- `PrismaClient` arrives through the constructor from `createApp()` in
  `app.ts`; the repository never opens transactions — the interactive
  transaction belongs to the application service (per `ddd-services`).
- Migrations: `prisma migrate` — mechanics in the per-stack persistence skill.
- Repository tests split: contract tests run callers against an in-memory
  fake; implementation tests hit a real PostgreSQL for the mapping, the
  compare-and-set, and the atomic increment.
- The ts-arch rule "no `@prisma/client` outside `infrastructure/`" is what
  makes the mirror boundary real.
