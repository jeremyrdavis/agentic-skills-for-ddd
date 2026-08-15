# Canonical Example — Value Objects (Java)

Load this file when creating a new value object (or converting primitives into
one) in Java. For placement and review decisions, the Core Rules and Quick
Reference tables in SKILL.md are sufficient on their own.

The native construct is the **record**: immutability, value-based
`equals`/`hashCode`/`toString`, compact declaration. Validation and
normalization live in the canonical (compact) constructor. Comments cite rule
numbers.

## Money

```java
package com.example.shared.domain;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

public record Money(BigDecimal amount, Currency currency) {

    public Money {                                            // canonical constructor (Rule 2)
        Objects.requireNonNull(amount, "amount required");
        Objects.requireNonNull(currency, "currency required");
        if (amount.signum() < 0) {
            throw new IllegalArgumentException("amount must be non-negative");
        }
        // Normalize via canonical-constructor reassignment (Rule 3):
        // scale to the currency's default fraction digits, banker's rounding.
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_EVEN);
    }

    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    // Behavior on the value (Rule 8); replacement, not mutation (Rule 5).
    public Money plus(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch: " + currency + " vs " + other.currency);
        }
        return new Money(amount.add(other.amount), currency);
    }

    public Money times(int quantity) {
        if (quantity < 0) {
            throw new IllegalArgumentException("quantity must be non-negative");
        }
        return new Money(amount.multiply(BigDecimal.valueOf(quantity)), currency);
    }
}
```

## EmailAddress

```java
package com.example.shared.domain;

import java.util.Objects;
import java.util.regex.Pattern;

public record EmailAddress(String value) {

    private static final Pattern PATTERN = Pattern.compile("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$");

    public EmailAddress {
        Objects.requireNonNull(value, "email required");
        value = value.trim().toLowerCase();                   // normalize (Rule 3)
        if (!PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("invalid email: " + value);
        }
    }

    public String domain() {                                  // behavior on the value (Rule 8)
        return value.substring(value.indexOf('@') + 1);
    }
}
```

## Typed ID wrappers

Same rules: record, constructor validation, equality by value (Rule 9). The
wrapper makes `placeOrder(OrderId, CustomerId)` impossible to call with
arguments swapped.

```java
package com.example.orders.domain;

import java.util.Objects;
import java.util.UUID;

public record OrderId(UUID value) {
    public OrderId { Objects.requireNonNull(value); }
    public static OrderId fresh() { return new OrderId(UUID.randomUUID()); }
}

public record CustomerId(UUID value) {
    public CustomerId { Objects.requireNonNull(value); }
}
```

Usage in an aggregate is a plain field — `private Money total;`,
`private final CustomerId customerId;` — full aggregate rules in
`ddd-aggregates`.

## The persistence mirror (lives in `<bc>/infrastructure`)

The domain records know nothing about persistence. Sketch only — full mapping
rules in `ddd-repositories`. Two equally valid shapes; the repository's call:

```java
// Option A: an embeddable mirror record (Hibernate ORM 6+ maps records natively)
@Embeddable
public record MoneyColumns(BigDecimal amount, String currencyCode) {
    Money toDomain() { return new Money(amount, Currency.getInstance(currencyCode)); }
    static MoneyColumns from(Money m) {
        return new MoneyColumns(m.amount(), m.currency().getCurrencyCode());
    }
}

// used inside the mirror entity:
@Entity @Table(name = "orders")
public class OrderRecord {
    @Id public UUID id;
    @Embedded public MoneyColumns total;
    // ...
}

// Option B: flat columns on the mirror entity; the repository assembles Money directly
//   public BigDecimal totalAmount;
//   public String totalCurrency;
```

Either way, `@Embeddable`/`@Embedded` and every other `jakarta.persistence`
concern live on the mirror in `<bc>/infrastructure` — never on the domain
record (Rule 6).

## What this demonstrates

- **Native construct (Rule 1):** all four types are records; equality, hash, and
  string are generated — never overridden (Rule 4).
- **Single construction path (Rule 2):** validation runs whether the record is
  built by application code or by the repository's `toDomain()` translation.
- **Normalization via compact-constructor reassignment (Rule 3):** the Java
  idiom — reassign the parameter before it's committed to the field.
- **Replacement, not mutation (Rule 5):** `plus`/`times` return new instances.
- **Plain domain types (Rules 6, 7):** zero `jakarta.persistence` imports, no
  `@Id` anywhere on these records.
- **Behavior on the value (Rule 8):** `Money.plus`, `EmailAddress.domain()` —
  no `MoneyService` taking primitives.
- **Typed IDs (Rule 9):** `OrderId`/`CustomerId` are nominally distinct record
  types; a swap is a compile error.

## Java idioms

- The compact constructor is the one place for validation and normalization;
  parameter reassignment inside it is the sanctioned idiom.
- **Don't use Lombok `@Value` or `@Data`** (Rule 10): records already do what
  `@Value` was designed for, without an annotation processor; `@Data` generates
  setters — the opposite of a value. If the codebase predates records, prefer
  converting over annotating.
- Value-object subclassing is out (see Excuse/Reality): records are implicitly
  final, which enforces this for free.
- Unit tests are pure JVM: constructing `Money` and asserting on `plus` needs
  no framework bootstrap.
- Domain purity is enforced by the ArchUnit rules from `ddd-foundations`.
