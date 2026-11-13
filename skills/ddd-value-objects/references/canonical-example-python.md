# Canonical Example — Value Objects (Python)

Load this file when creating a new value object (or converting primitives into
one) in Python. For placement and review decisions, the Core Rules and Quick
Reference tables in SKILL.md are sufficient on their own.

The native construct is **`@dataclass(frozen=True)`**: immutability, generated
value-based `__eq__`/`__hash__`/`__repr__`, compact declaration. Validation
lives in `__post_init__`; normalization inside a frozen dataclass uses
`object.__setattr__` — the sanctioned idiom, not a hack. Comments cite rule
numbers.

## Money

```python
# shared/domain/money.py
from dataclasses import dataclass
from decimal import ROUND_HALF_EVEN, Decimal

_FRACTION_DIGITS = {"USD": 2, "EUR": 2, "JPY": 0}  # or derive from a currency table


@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:                       # single construction path (Rule 2)
        if self.currency not in _FRACTION_DIGITS:
            raise ValueError(f"unknown currency: {self.currency}")
        if self.amount < 0:
            raise ValueError("amount must be non-negative")
        # Normalize (Rule 3): frozen dataclasses forbid plain assignment, so the
        # idiom for normalization in __post_init__ is object.__setattr__.
        exponent = Decimal(10) ** -_FRACTION_DIGITS[self.currency]
        object.__setattr__(self, "amount",
                           self.amount.quantize(exponent, rounding=ROUND_HALF_EVEN))

    @classmethod
    def zero(cls, currency: str) -> "Money":
        return cls(Decimal("0"), currency)

    # Behavior on the value (Rule 8); replacement, not mutation (Rule 5).
    def plus(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"currency mismatch: {self.currency} vs {other.currency}")
        return Money(self.amount + other.amount, self.currency)

    def times(self, quantity: int) -> "Money":
        if quantity < 0:
            raise ValueError("quantity must be non-negative")
        return Money(self.amount * quantity, self.currency)
```

## EmailAddress

```python
# shared/domain/email_address.py
import re
from dataclasses import dataclass

_PATTERN = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


@dataclass(frozen=True)
class EmailAddress:
    value: str

    def __post_init__(self) -> None:
        normalized = self.value.strip().lower()             # normalize (Rule 3)
        if not _PATTERN.match(normalized):
            raise ValueError(f"invalid email: {normalized}")
        object.__setattr__(self, "value", normalized)

    def domain(self) -> str:                                # behavior on the value (Rule 8)
        return self.value.split("@", 1)[1]
```

## Typed ID wrappers

Same rules: frozen dataclass, constructor validation, equality by value
(Rule 9). Python is duck-typed at runtime, but the type checker (mypy/pyright)
treats `OrderId` and `CustomerId` as distinct — run it in CI or the swap
protection is advisory.

```python
# orders/domain/order_id.py, customer_id.py
from dataclasses import dataclass
from uuid import UUID, uuid4


@dataclass(frozen=True)
class OrderId:
    value: UUID

    @classmethod
    def fresh(cls) -> "OrderId":
        return cls(uuid4())


@dataclass(frozen=True)
class CustomerId:
    value: UUID
```

Usage in an aggregate is a plain field — `self._total: Money`,
`self._customer_id: CustomerId` — full aggregate rules in `ddd-aggregates`.

## The persistence mirror (lives in `<bc>/infrastructure`)

The domain dataclasses know nothing about persistence. Sketch only — full
mapping rules in `ddd-persistence`. Flat columns on the SQLAlchemy mirror; the
repository assembles the value objects:

```python
# orders/infrastructure/models.py
class OrderModel(Base):
    __tablename__ = "orders"

    id: Mapped[UUID] = mapped_column(primary_key=True)
    total_amount: Mapped[Decimal]        # Money split into columns — repository's call
    total_currency: Mapped[str]
    customer_id: Mapped[UUID]
    # ...

    def to_domain(self) -> Order:
        return Order.rehydrate(
            self.id,
            CustomerId(self.customer_id),
            ...,
            total=Money(self.total_amount, self.total_currency),  # VO validation re-runs here (Rule 2)
            ...,
        )
```

(SQLAlchemy composites are the embedded-mirror alternative; either way the
mapping lives on the mirror, never on the domain dataclass — Rule 6.)

## What this demonstrates

- **Native construct (Rule 1):** all four types are frozen dataclasses;
  `__eq__`/`__hash__`/`__repr__` are generated — never hand-written (Rule 4).
- **Single construction path (Rule 2):** `__post_init__` runs whether the value
  is built by application code or by the repository's `to_domain()`.
- **Normalization idiom (Rule 3):** `object.__setattr__` inside
  `__post_init__` is *the* sanctioned way to normalize a frozen dataclass —
  don't drop `frozen=True` to make assignment easier.
- **Replacement, not mutation (Rule 5):** `plus`/`times` return new instances.
- **Plain domain types (Rules 6, 7):** stdlib imports only; no SQLAlchemy, no
  Pydantic, no identity field.
- **Behavior on the value (Rule 8):** `Money.plus`, `EmailAddress.domain()` —
  no `MoneyService` taking primitives.
- **Typed IDs (Rule 9):** distinct dataclass types; the type checker catches the
  swap.

## Python idioms

- `@dataclass(frozen=True)` over hand-rolled classes, `attrs`, or Pydantic
  models in the domain (Rule 10): Pydantic is a serialization/validation
  framework for the api layer; importing it in `<bc>/domain` couples the domain
  to a framework and its coercion semantics.
- Add `slots=True` (`@dataclass(frozen=True, slots=True)`) for memory-heavy
  values if you like — orthogonal to these rules.
- Use `Decimal` (never `float`) for money amounts; construct from strings
  (`Decimal("19.99")`) to avoid binary-float dust.
- Typed-ID swap protection is only as strong as the type checker — mypy or
  pyright in CI is part of the pattern here, alongside the pytest-archon purity
  rules from `ddd-foundations`.
- Unit tests are pure Python: constructing `Money` and asserting on `plus`
  needs no database or app fixture.
