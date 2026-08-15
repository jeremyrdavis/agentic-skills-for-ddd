# Canonical Example — Value Objects (TypeScript)

Load this file when creating a new value object (or converting primitives into
one) in TypeScript. For placement and review decisions, the Core Rules and
Quick Reference tables in SKILL.md are sufficient on their own.

The native construct is a **class with `readonly` fields**, validation and
normalization in the constructor. Two TypeScript-specific realities shape this
file: **the language generates no value equality** (Rule 4 means *writing* one
canonical `equals()`, not avoiding one), and **structural typing silently
defeats typed ID wrappers** unless the types are made nominally distinct
(Rule 9). Comments cite rule numbers.

## Money

```typescript
// shared/domain/money.ts
const FRACTION_DIGITS: Record<string, number> = { USD: 2, EUR: 2, JPY: 0 };

export class Money {
  readonly amount: number;   // minor units? see the money-precision idiom note below
  readonly currency: string;

  constructor(amount: number, currency: string) {         // single construction path (Rule 2)
    if (!(currency in FRACTION_DIGITS)) {
      throw new Error(`unknown currency: ${currency}`);
    }
    if (amount < 0) {
      throw new Error("amount must be non-negative");
    }
    // Normalize before assignment (Rule 3): round to the currency's fraction digits.
    const factor = 10 ** FRACTION_DIGITS[currency];
    this.amount = Math.round(amount * factor) / factor;
    this.currency = currency;
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }

  // TypeScript generates no value equality — define ONE canonical equals (Rule 4).
  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  // Behavior on the value (Rule 8); replacement, not mutation (Rule 5).
  plus(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error(`currency mismatch: ${this.currency} vs ${other.currency}`);
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  times(quantity: number): Money {
    if (quantity < 0 || !Number.isInteger(quantity)) {
      throw new Error("quantity must be a non-negative integer");
    }
    return new Money(this.amount * quantity, this.currency);
  }
}
```

## EmailAddress

```typescript
// shared/domain/email-address.ts
const PATTERN = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;

export class EmailAddress {
  readonly value: string;

  constructor(value: string) {
    const normalized = value.trim().toLowerCase();          // normalize (Rule 3)
    if (!PATTERN.test(normalized)) {
      throw new Error(`invalid email: ${normalized}`);
    }
    this.value = normalized;
  }

  equals(other: EmailAddress): boolean {                    // canonical equals (Rule 4)
    return this.value === other.value;
  }

  domain(): string {                                        // behavior on the value (Rule 8)
    return this.value.slice(this.value.indexOf("@") + 1);
  }
}
```

## Typed ID wrappers — nominal distinctness is the whole point

**The trap:** `type OrderId = { value: string }` and
`type CustomerId = { value: string }` are *structurally identical*, so
TypeScript treats them as interchangeable — the swap protection is illusory.
Classes with a private brand member are nominally distinct (Rule 9):

```typescript
// orders/domain/order-id.ts, customer-id.ts
export class OrderId {
  private readonly __brand = "OrderId";   // makes the class nominal, not structural
  constructor(readonly value: string) {
    if (!value) throw new Error("OrderId requires a value");
  }
  static fresh(): OrderId {
    return new OrderId(crypto.randomUUID());
  }
  equals(other: OrderId): boolean {
    return this.value === other.value;
  }
}

export class CustomerId {
  private readonly __brand = "CustomerId";
  constructor(readonly value: string) {
    if (!value) throw new Error("CustomerId requires a value");
  }
  equals(other: CustomerId): boolean {
    return this.value === other.value;
  }
}
```

With the brand, `placeOrder(id: OrderId, customerId: CustomerId)` rejects a
swap at compile time. (Branded primitive types —
`type OrderId = string & { __brand: "OrderId" }` — are a lighter alternative
when you don't need methods; pick one style per codebase and stick to it.)

Usage in an aggregate is a plain field — `private _total: Money`,
`private readonly _customerId: CustomerId` — full aggregate rules in
`ddd-aggregates`.

## The persistence mirror (lives in `<bc>/infrastructure`)

The domain classes know nothing about persistence. Sketch only — full mapping
rules in `ddd-repositories`. Flat columns in the Drizzle schema; the repository
assembles the value objects:

```typescript
// orders/infrastructure/schema.ts
export const ordersTable = pgTable("orders", {
  id: uuid("id").primaryKey(),
  totalAmount: numeric("total_amount").notNull(),   // Money split into columns — repository's call
  totalCurrency: text("total_currency").notNull(),
  customerId: uuid("customer_id").notNull(),
});

export function toDomain(row: typeof ordersTable.$inferSelect): Order {
  return Order.rehydrate(
    row.id,
    new CustomerId(row.customerId),
    /* ... */
    new Money(Number(row.totalAmount), row.totalCurrency),  // VO validation re-runs here (Rule 2)
    /* ... */
  );
}
```

Every schema/ORM concern lives on the mirror in `<bc>/infrastructure` — never
on the domain class (Rule 6).

## What this demonstrates

- **Native construct (Rule 1):** classes with `readonly` fields; no setters,
  no mutation.
- **Single construction path (Rule 2):** the constructor validates whether the
  value is built by application code or by the repository's `toDomain()`.
- **Normalization before assignment (Rule 3):** compute the normalized value,
  then assign the `readonly` field once.
- **Canonical `equals()` (Rule 4):** TypeScript's `===` is identity, not value
  equality — the rule here is *write* one `equals()` per value type and never
  compare values with `===`/`!==` (except on the wrapped primitives inside
  `equals` itself).
- **Replacement, not mutation (Rule 5):** `plus`/`times` return new instances.
- **Plain domain types (Rules 6, 7):** no `drizzle-orm`, `pg`, or `express`
  imports; no identity field.
- **Behavior on the value (Rule 8):** `Money.plus`, `EmailAddress.domain()`.
- **Nominal typed IDs (Rule 9):** the brand member defeats structural typing;
  without it the wrapper provides zero swap protection.

## TypeScript idioms

- **Money precision:** JavaScript numbers are binary floats — `0.1 + 0.2 !==
  0.3`. For real money, store integer minor units (cents) or use a decimal
  library (`decimal.js`, `big.js`) inside `Money`; the value-object boundary is
  exactly the right place to contain that decision. The `number`-based example
  above keeps the shape readable; harden the internals without changing the
  type's surface.
- Prefer classes over interfaces/type aliases for value objects: aliases are
  structural (no nominal identity, no methods) and can't carry `equals()`.
- `Object.freeze(this)` at the end of the constructor adds runtime immutability
  on top of compile-time `readonly` if you want belt-and-suspenders.
- Don't reach for decorator/codegen libraries to synthesize value classes
  (Rule 10) — the constructor + `readonly` + `equals()` pattern is the whole
  cost.
- Unit tests are pure Node: constructing `Money` and asserting on `plus` needs
  no database or Express app.
- Domain purity is enforced by the ts-arch rules from `ddd-foundations`.
