# Canonical Example — `payments` Bounded Context (TypeScript)

Load this file when scaffolding a new bounded context in TypeScript. For
placement and review decisions, the Core Rules and Quick Reference tables in
SKILL.md are sufficient on their own.

Use case: *authorize a payment* — load the `Payment` aggregate, run a
domain-service fraud check, advance the aggregate's state machine, persist via an
infrastructure repository implementation, return a `PaymentDTO`.

## Module layout

Layers are directories under one bounded-context directory:

```
payments/
├── domain/
│   ├── payment-method.ts          (value object)
│   ├── payment.ts                 (aggregate root — no framework imports)
│   ├── payment-repository.ts      (interface — uses only domain types)
│   ├── fraud-detection-domain-service.ts
│   └── payment-authorized-event.ts    (domain event — PaymentAuthorizedEvent)
├── application/
│   ├── authorize-payment-command.ts   (input DTO)
│   ├── payment-dto.ts                 (output DTO)
│   └── authorize-payment-application-service.ts
├── infrastructure/
│   ├── schema.ts                  (Drizzle table definitions — ORM lives here)
│   └── postgres-payment-repository.ts  (implements the domain interface, translates)
└── api/
    ├── payments-router.ts         (Express router)
    └── error-handlers.ts          (error-mapping middleware)
```

Allowed dependency directions (one way only):

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

## The key files

### 1. Repository interface — `domain`, speaks only domain language

```typescript
// payments/domain/payment-repository.ts
import { Payment, PaymentId } from "./payment";
import { OrderId } from "./order-id";

export interface PaymentRepository {
  findById(id: PaymentId): Promise<Payment | null>;
  findByOrderId(orderId: OrderId): Promise<Payment | null>;
  persist(payment: Payment): Promise<void>;
}
```

No Drizzle, no `pg`, no Express types. Its vocabulary is the aggregate's
vocabulary.

### 2. Repository implementation — `infrastructure`, brings the framework

```typescript
// payments/infrastructure/postgres-payment-repository.ts
import { eq } from "drizzle-orm";
import type { Db } from "../../shared/db";
import { Payment, PaymentId } from "../domain/payment";
import { PaymentRepository } from "../domain/payment-repository";
import { paymentsTable, toDomain, fromDomain } from "./schema";

export class PostgresPaymentRepository implements PaymentRepository {
  constructor(private readonly db: Db) {}

  async findById(id: PaymentId): Promise<Payment | null> {
    const rows = await this.db.select().from(paymentsTable)
      .where(eq(paymentsTable.id, id.value));
    return rows[0] ? toDomain(rows[0]) : null;      // persistence -> domain
  }

  async persist(payment: Payment): Promise<void> {
    await this.db.insert(paymentsTable).values(fromDomain(payment))
      .onConflictDoUpdate({ target: paymentsTable.id, set: fromDomain(payment) });
  }

  // findByOrderId elided — same pattern
}
```

The Drizzle table definition and the row↔domain translation (`toDomain` /
`fromDomain`) live in `infrastructure/schema.ts`, keeping ORM concerns out of
`domain`.

### 3. Application service — `application`, orchestrates the use case

```typescript
// payments/application/authorize-payment-application-service.ts
import { FraudDetectionDomainService } from "../domain/fraud-detection-domain-service";
import { PaymentRepository } from "../domain/payment-repository";  // the interface
import { AuthorizePaymentCommand } from "./authorize-payment-command";
import { PaymentDTO, toPaymentDTO } from "./payment-dto";

export class AuthorizePaymentApplicationService {
  constructor(
    private readonly payments: PaymentRepository,          // domain interface
    private readonly fraud: FraudDetectionDomainService,   // domain service
    private readonly tx: Transaction,                      // owns the transaction boundary
  ) {}

  async authorize(cmd: AuthorizePaymentCommand): Promise<PaymentDTO> {
    return this.tx.run(async () => {
      const payment = await this.payments.findById(cmd.paymentId);
      if (!payment) throw new PaymentNotFoundError(cmd.paymentId);
      const verdict = this.fraud.check(payment, cmd.method);
      payment.authorize(verdict);                    // aggregate enforces invariants
      await this.payments.persist(payment);
      return toPaymentDTO(payment);                  // DTO out — never the entity
    });
  }
}
```

Load → call domain service → mutate aggregate → persist → map to DTO.
Transactions and DTO mapping live here and only here.

### 4. HTTP resource — `api`, depends only on `application`

```typescript
// payments/api/payments-router.ts
import { Router } from "express";
import { AuthorizePaymentApplicationService } from "../application/authorize-payment-application-service";
// never an import from ../domain or ../infrastructure

export function paymentsRouter(authorizePayment: AuthorizePaymentApplicationService): Router {
  const router = Router();

  router.post("/payments/:id/authorize", async (req, res, next) => {
    try {
      const dto = await authorizePayment.authorize(toCommand(req.params.id, req.body));
      res.status(200).json(dto);
    } catch (err) {
      next(err);   // mapped to a status code by error-handlers.ts
    }
  });

  return router;
}
```

The router receives the application service from the composition root (`boot/`),
which constructs `PostgresPaymentRepository` and wires everything. The router
never sees `Payment`, `PaymentRepository`, or `PostgresPaymentRepository`.

## What this demonstrates

- The repository **interface** lives in `domain`; its signatures use only domain types.
- The **implementation** lives in `infrastructure` and carries all framework
  concerns — including the Drizzle schema and the row↔domain translation.
- The application service orchestrates, owns the transaction boundary, depends
  on the domain *interface*, and returns DTOs.
- The fraud check is a `DomainService`: pure domain logic, no I/O.
- The domain event is `PaymentAuthorizedEvent` — past tense, `Event` suffix, in `domain`.
- The router lives in `api` and imports only from `application`.

## TypeScript idioms

- The repository contract is a TS `interface`; value objects and events are
  classes or `readonly` object types. Aggregates are classes with state
  transitions as methods.
- The composition root is an explicit wiring module (`boot/`) — or the DI
  container in NestJS. Application services receive interfaces via constructor
  parameters; routers are factory functions taking the services they need.
- Domain purity: no `drizzle-orm`, `pg`, `prisma`, or `express` imports under
  `payments/domain/`. Table schemas are infrastructure; request/response typing
  is `api`.
- Enforce with **ts-arch** (or dependency-cruiser): rules forbidding
  `payments/domain` from importing other layers, and `payments/application`
  from importing `payments/infrastructure` or `payments/api`.
