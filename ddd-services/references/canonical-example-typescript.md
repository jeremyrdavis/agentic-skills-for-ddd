# Canonical Example — The Refund Triangle (TypeScript)

Load this file when creating a new service or splitting a mixed one in
TypeScript. For placement and review decisions, the Core Rules and Quick
Reference tables in SKILL.md are sufficient on their own.

The "process refund" use case: three classes, three roles. The `Order`
aggregate is the one from `ddd-aggregates`' canonical example, extended with a
`refund` transition. Comments cite rule numbers.

## 1. The aggregate's domain method (in `<bc>/domain`; full design in `ddd-aggregates`)

```typescript
// orders/domain/order.ts — sketch of the relevant method only
refund(amount: Money): void {
  if (this._status !== "PLACED" && this._status !== "FULFILLED") {
    throw new OrderRefundNotAllowedError(new OrderId(this._id));
  }
  if (amount.amount > this._total.amount) {
    throw new Error("refund amount exceeds order total");
  }
  this._status = "REFUNDED";
  this._refundedAmount = amount;
  this.pendingEvents.push(new OrderRefundedEvent(new OrderId(this._id), amount));
}
```

Enforces the *invariant* (Rule 7): the order is in a refundable state, the
amount doesn't exceed the total. The aggregate **does not compute** the amount;
it accepts the value and verifies the resulting state is legal.

## 2. The domain service (in `<bc>/domain`)

```typescript
// orders/domain/refund-calculator-domain-service.ts
import { Money } from "./money";
import { Order } from "./order";
import { OrderLine } from "./order-line";
import { RefundPolicy } from "./refund-policy";   // a plain value type — a DOMAIN value

/**
 * Pure computation: domain in, domain out (Rule 4). No repository, no
 * transaction, no event publisher (Rule 5). An instance, not module-level
 * functions, so it can carry configured policy (Rule 9).
 */
export class RefundCalculatorDomainService {
  constructor(private readonly policy: RefundPolicy) {}   // configured domain value — allowed (Rule 5)

  compute(order: Order, now: Date): Money {
    const base = order.total;
    const prorated = this.applyTimeProration(base, order.placedAt, now);
    const afterRestocking = this.applyRestockingFees(prorated, order.lines);
    return this.roundToCurrencyFractionDigits(afterRestocking);
    // ... 80 lines of pure rules: prorated within policy.windowDays,
    // restocking fees on opened SKUs, currency rounding, etc.
  }

  private applyTimeProration(base: Money, placedAt: Date, now: Date): Money { /* ... */ }
  private applyRestockingFees(prorated: Money, lines: readonly OrderLine[]): Money { /* ... */ }
  private roundToCurrencyFractionDigits(m: Money): Money { /* ... */ }
}
```

Unit-testable with a real `Order` and a fixed `Date` — no database, no Express
app, no framework.

## 3. The application service (in `<bc>/application`)

```typescript
// orders/application/process-refund-application-service.ts
import { OrderNotFoundError } from "../domain/errors";
import { OrderId } from "../domain/order-id";
import { OrderRepository } from "../domain/order-repository";           // the interface
import { RefundCalculatorDomainService } from "../domain/refund-calculator-domain-service";
import { RefundDTO, toRefundDTO } from "./refund-dto";

export class ProcessRefundApplicationService {
  constructor(
    private readonly orders: OrderRepository,                           // domain interface (Rule 3)
    private readonly calculator: RefundCalculatorDomainService,
    private readonly events: EventPublisher,                            // from shared/events
    private readonly clock: () => Date,                                 // injected so tests can fix time
    private readonly tx: Transaction,                                   // owns the boundary (Rule 8)
  ) {}

  async refund(orderId: OrderId): Promise<RefundDTO> {
    return this.tx.run(async () => {                                    // here, only here (Rule 8)
      const order = await this.orders.findById(orderId);
      if (!order) throw new OrderNotFoundError(orderId);

      const amount = this.calculator.compute(order, this.clock());      // domain-service call (Rule 7)
      order.refund(amount);                                             // aggregate transition
      await this.orders.persist(order);

      for (const event of order.pullPendingEvents()) {                  // drain and hand off (Rule 11)
        this.events.publish(event);
      }

      return toRefundDTO(order, amount);                                // DTO out — never the aggregate (Rule 10)
    });
  }
}
```

Load → compute → mutate → persist → drain → map. The application service is the
*only* class in the triangle that touches the transaction, the publisher, or
the repository. No rule computation hides as a private method — that lives in
the domain service.

**After-commit delivery (Rule 11):** `publish(...)` is called inside the use
case, but the publisher implementation must deliver to handlers only after the
transaction commits (buffer-and-flush-on-commit, or an outbox) — that
publisher-side rule lives in `ddd-domain-events`.

## Why this triangle works

The three classes have different dependency signatures, and the directory each
lives in enforces them (per `ddd-foundations` and the ts-arch rules). The
aggregate depends on nothing beyond domain types. The domain service has no
I/O dependencies. The application service is where framework concerns
aggregate. If you find yourself importing the Drizzle db or a repository
implementation into a file under `orders/domain/`, you've discovered the class
is actually an application service — move and rename it.

## TypeScript idioms

- The composition root is `boot/`: build `RefundPolicy` from configuration
  there, construct `new RefundCalculatorDomainService(policy)` once, and pass
  it into the application service's constructor — the domain service never
  reads `process.env` itself.
- Don't flatten the domain service into exported module functions (Rule 9):
  the instance carries configured policy and can be substituted in tests;
  bare exports can't without module-mocking machinery.
- `clock` as `() => Date` (`() => new Date()` in boot, `() => fixed` in tests)
  is the lightweight TS spelling of an injected time source.
- Express routers stay in `api/` (per `ddd-foundations`); the application
  service receives everything through its constructor — no `req`/`res` types
  below the api layer.
- Testing split: domain-service tests are pure Node; application-service tests
  use an in-memory `OrderRepository` fake and a capturing publisher.
- Events: past tense, `*Event` suffix (`OrderRefundedEvent`), per
  `ddd-foundations` and `ddd-domain-events`.
