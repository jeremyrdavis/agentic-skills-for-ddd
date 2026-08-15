# Canonical Example — `Order` Aggregate Root (Python)

Load this file when creating or substantially reshaping an aggregate in Python.
For placement and review decisions, the Core Rules and Quick Reference tables in
SKILL.md are sufficient on their own.

The `Order` aggregate lives in `orders/domain/order.py`. **Plain object — zero
SQLAlchemy/Pydantic/driver imports.** Exercises every Core Rule; comments cite
rule numbers.

## The aggregate

```python
# orders/domain/order.py
from __future__ import annotations

from enum import Enum
from uuid import UUID

from orders.domain.customer_id import CustomerId
from orders.domain.events import (
    DomainEvent,
    OrderCancelledEvent,
    OrderFulfilledEvent,
    OrderPlacedEvent,
)
from orders.domain.errors import OrderCancellationNotAllowedError
from orders.domain.money import Money
from orders.domain.order_id import OrderId
from orders.domain.order_line import OrderLine


class OrderStatus(Enum):
    PLACED = "PLACED"
    FULFILLED = "FULFILLED"
    CANCELLED = "CANCELLED"


class Order:
    """Aggregate root. Construct ONLY via Order.place() or Order.rehydrate().

    Python has no private constructors; __init__ is non-public by convention
    (Rule 4) — direct construction from services or api code is a review
    violation this skill flags. place() and rehydrate() are the only entry
    points.
    """

    def __init__(self, id: UUID, customer_id: CustomerId, lines: list[OrderLine],
                 status: OrderStatus, total: Money, version: int) -> None:
        self._id = id                          # identity (Rule 9)
        self._version = version                # optimistic-lock cursor (Rule 10) — plain int
        self._customer_id = customer_id        # reference by ID, not object (Rules 8, 9)
        self._lines = list(lines)
        self._status = status
        self._total = total                    # value object as a plain field (Rule 11)
        self._pending_events: list[DomainEvent] = []

    @classmethod
    def place(cls, id: OrderId, customer_id: CustomerId,
              lines: list[OrderLine]) -> "Order":
        """Named factory — enforces creation invariants (Rule 5) and raises
        OrderPlacedEvent (Rule 12)."""
        if not lines:
            raise ValueError("an order must have at least one line")
        currency = lines[0].unit_price.currency
        if any(line.unit_price.currency != currency for line in lines):
            raise ValueError("all lines must share a currency")

        total = sum((line.line_total() for line in lines),
                    start=Money.zero(currency))

        order = cls(id.value, customer_id, lines, OrderStatus.PLACED, total, 0)
        order._pending_events.append(OrderPlacedEvent(id, customer_id))  # Rule 12
        return order

    @classmethod
    def rehydrate(cls, id: UUID, customer_id: CustomerId, lines: list[OrderLine],
                  status: OrderStatus, total: Money, version: int) -> "Order":
        """Reconstruction path for the persistence layer (Rule 6). Trusts inputs
        that already passed validation when first persisted; does NOT re-emit
        OrderPlacedEvent. Only the repository implementation in
        <bc>/infrastructure calls this (covered by ddd-repositories)."""
        return cls(id, customer_id, lines, status, total, version)

    def cancel(self) -> None:
        """Transition method — enforces the state-machine invariant (Rule 7)."""
        if self._status is OrderStatus.FULFILLED:
            raise OrderCancellationNotAllowedError(OrderId(self._id))
        self._status = OrderStatus.CANCELLED
        self._pending_events.append(OrderCancelledEvent(OrderId(self._id)))

    def mark_fulfilled(self) -> None:
        if self._status is not OrderStatus.PLACED:
            raise ValueError("only PLACED orders can be fulfilled")
        self._status = OrderStatus.FULFILLED
        self._pending_events.append(OrderFulfilledEvent(OrderId(self._id)))

    # Read accessors — only what callers need (Rule 3). No setters (Rule 2).
    @property
    def id(self) -> OrderId: return OrderId(self._id)
    @property
    def status(self) -> OrderStatus: return self._status
    @property
    def total(self) -> Money: return self._total
    @property
    def customer_id(self) -> CustomerId: return self._customer_id
    @property
    def lines(self) -> tuple[OrderLine, ...]: return tuple(self._lines)  # defensive copy
    @property
    def version(self) -> int: return self._version

    def pull_pending_events(self) -> list[DomainEvent]:
        """Drained by the application service after persist; published after
        commit (Rule 12)."""
        drained = list(self._pending_events)
        self._pending_events.clear()
        return drained
```

## Companion types

Also plain, also persistence-free (full coverage in `ddd-value-objects` and
`ddd-domain-events`):

```python
# orders/domain/order_id.py, customer_id.py, order_line.py, events.py
from dataclasses import dataclass
from uuid import UUID


@dataclass(frozen=True)
class OrderId:
    value: UUID


@dataclass(frozen=True)
class CustomerId:
    value: UUID


@dataclass(frozen=True)
class OrderLine:
    product_sku: str
    quantity: int
    unit_price: Money

    def __post_init__(self) -> None:
        if self.quantity <= 0:
            raise ValueError("quantity must be positive")

    def line_total(self) -> Money:
        return self.unit_price.times(self.quantity)


# Events: past tense, *Event suffix (per ddd-foundations)
@dataclass(frozen=True)
class OrderPlacedEvent(DomainEvent):
    order_id: OrderId
    customer_id: CustomerId


@dataclass(frozen=True)
class OrderCancelledEvent(DomainEvent):
    order_id: OrderId


@dataclass(frozen=True)
class OrderFulfilledEvent(DomainEvent):
    order_id: OrderId
```

## The persistence mirror (lives in `<bc>/infrastructure`)

The aggregate has no idea this class exists. Sketch only — full coverage in
`ddd-repositories`:

```python
# orders/infrastructure/models.py
from sqlalchemy.orm import Mapped, mapped_column
from orders.infrastructure.base import Base


class OrderModel(Base):
    __tablename__ = "orders"
    __mapper_args__ = {"version_id_col": "version"}   # the framework's lock column (Rule 10)

    id: Mapped[UUID] = mapped_column(primary_key=True)
    version: Mapped[int] = mapped_column(nullable=False)
    customer_id: Mapped[UUID]
    status: Mapped[str]
    total_amount: Mapped[Decimal]     # Money split into columns — repository's call
    total_currency: Mapped[str]
    # lines: relationship to OrderLineModel

    @staticmethod
    def from_domain(order: Order) -> "OrderModel": ...
    def to_domain(self) -> Order:
        return Order.rehydrate(self.id, CustomerId(self.customer_id),
                               self._map_lines(), OrderStatus(self.status),
                               Money(self.total_amount, self.total_currency),
                               self.version)
```

**Optimistic locking mechanics:** SQLAlchemy's `version_id_col` raises
`StaleDataError` on a stale write (alternatively, an explicit
`UPDATE ... WHERE version = :loaded` with a rowcount check). The repository
implementation wraps it in a domain-neutral `ConcurrencyConflictError`; the api
layer's exception handler turns it into **409 Conflict**.

## What this demonstrates

- **Plain object (Rule 1):** imports only stdlib and the BC's own domain types.
- **No setters, no public mutable fields (Rules 2, 3):** state is underscore-
  private; read access via `@property`; only domain methods mutate.
- **Factory enforces creation invariants (Rules 4, 5):** empty lines and mixed
  currencies raise before the aggregate exists.
- **Rehydrate path (Rule 6):** the repository reconstructs via `rehydrate(...)`;
  no `OrderPlacedEvent` re-emitted; domain code never calls it.
- **Cross-aggregate reference by ID (Rule 8):** `CustomerId`, never a `Customer` field.
- **Typed identity (Rule 9):** `OrderId`/`CustomerId` frozen dataclasses at the
  API surface; raw `UUID` internally.
- **Plain `int` version (Rule 10):** the mirror carries `version_id_col`; the
  aggregate carries `_version: int`.
- **Value object as plain field (Rule 11):** `_total: Money`, no mapping directives.
- **Events (Rule 12):** `*Event` suffix, past tense, raised inside, drained via
  `pull_pending_events()`, published after commit.

## Python idioms

- No private constructors exist, so Rule 4 is convention-plus-review here:
  `__init__` is documented as non-public, and any `Order(...)` call outside
  `orders/domain/` and the repository's `rehydrate` path is a review violation.
  Skip sentinel-guard machinery — that ceremony feeds the over-engineering excuse
  without adding real enforcement.
- Value objects, IDs, and events are `@dataclass(frozen=True)`; the aggregate is
  a regular class (it's mutable by design, through named methods only).
- Invariant unit tests are pure Python: no database, no FastAPI app, no fixtures
  beyond constructing domain values — `Order.place(...)` and `order.cancel()`
  are directly testable.
- Domain purity: no `sqlalchemy`, `pydantic`, `fastapi`, or driver imports under
  `orders/domain/` — enforced by the pytest-archon rules from `ddd-foundations`.
