# Canonical Example — Repository, Mirror, and Use-Case-Optimal Query (Python)

Load this file when implementing a repository or adding a non-trivial query in
Python. For placement and review decisions, the Core Rules and Quick Reference
tables in SKILL.md are sufficient on their own. Comments cite rule numbers.

## 1. The interface — `<bc>/domain`, ubiquitous language only

```python
# orders/domain/order_repository.py
from typing import Optional, Protocol

from orders.domain.order import Order
from orders.domain.order_id import OrderId
from orders.domain.order_page import OrderPage


class OrderRepository(Protocol):
    def find_by_id(self, order_id: OrderId) -> Optional[Order]: ...   # aggregate out (Rule 2)
    def persist(self, order: Order) -> None: ...                       # whole aggregate (Rule 6)
    def remove(self, order_id: OrderId) -> None: ...
    def find_recent(self, page: int, size: int) -> OrderPage: ...      # use-case-optimal VO (Rules 9, 10)
```

## 2. The purpose-built query VOs — `<bc>/domain`, per `ddd-value-objects`

Domain-specific, immutable, equality by value — explicitly **not** DTOs (Rule 9).

```python
# orders/domain/order_page.py
from dataclasses import dataclass

from orders.domain.customer_id import CustomerId
from orders.domain.money import Money
from orders.domain.order import OrderStatus
from orders.domain.order_id import OrderId


@dataclass(frozen=True)
class OrderSummary:
    id: OrderId
    customer_id: CustomerId
    total: Money
    status: OrderStatus


@dataclass(frozen=True)
class OrderPage:
    items: tuple[OrderSummary, ...]     # tuple, not list — immutable (Rule 9)
    total_count: int
    page: int
    size: int
```

## 3. The implementation — `<bc>/infrastructure`, same name (Rule 3)

```python
# orders/infrastructure/order_repository.py
from typing import Optional

from sqlalchemy import func, select, update
from sqlalchemy.orm import Session

from orders.domain.order import Order
from orders.domain.order_id import OrderId
from orders.domain.order_page import OrderPage
from orders.infrastructure.models import OrderModel


class OrderRepository:
    """Satisfies orders.domain.order_repository.OrderRepository.

    Same name as the Protocol — the package path is the distinguisher, and a
    Protocol needs no inheritance, so the names never collide in one file.
    No technology name in the class name (Rule 3).
    """

    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id(self, order_id: OrderId) -> Optional[Order]:
        row = self._session.get(OrderModel, order_id.value)
        return row.to_domain() if row else None              # rehydrate inside (Rule 5)

    def persist(self, order: Order) -> None:
        self._session.merge(OrderModel.from_domain(order))   # whole aggregate (Rule 6)
        # version_id_col on OrderModel does the compare-and-set (Rule 7):
        # a stale write raises StaleDataError.

    def find_recent(self, page: int, size: int) -> OrderPage:   # Vernon query (Rules 9, 10)
        rows = self._session.execute(
            select(OrderModel).order_by(OrderModel.placed_at.desc())
            .offset(page * size).limit(size)
        ).scalars().all()
        total = self._session.execute(
            select(func.count()).select_from(OrderModel)
        ).scalar_one()
        return OrderPage(items=tuple(r.to_summary() for r in rows),
                         total_count=total, page=page, size=size)
```

The mirror `OrderModel` (table mapping, `version_id_col`, `to_domain` /
`from_domain` / `to_summary`) lives in `orders/infrastructure/models.py`, per
`ddd-aggregates`' mirror table — nothing outside the implementation touches it
(Rule 4).

**Conflict surfacing (Rule 7):** the implementation catches SQLAlchemy's
`StaleDataError` and raises the domain-neutral `ConcurrencyConflictError`; the
api layer's exception handler maps it to **409 Conflict**. The repository never
retries — the use case decides what a conflict means.

## 4. An atomic operation — from the Positive Thoughts spec (Rule 8)

`thumbs_up` has no invariant, so load-mutate-persist would be a lost-update
hazard. The Protocol gets an intention-revealing method; the implementation is
one atomic UPDATE:

```python
# thoughts/domain — on ThoughtRepository (Protocol):
def increment_thumbs_up(self, thought_id: ThoughtId) -> None: ...

# thoughts/infrastructure — in the implementation:
def increment_thumbs_up(self, thought_id: ThoughtId) -> None:
    self._session.execute(
        update(ThoughtModel)
        .where(ThoughtModel.id == thought_id.value)
        .values(thumbs_up=ThoughtModel.thumbs_up + 1)        # atomic under concurrency
    )
```

## What this demonstrates

- **Interface in domain vocabulary (Rule 2):** a `Protocol` with typed IDs in,
  aggregates and domain VOs out; zero framework imports in `<bc>/domain`.
- **Same-name implementation (Rule 3):** package path is the distinguisher;
  structural typing means the names never meet.
- **Translation is the whole job (Rule 5):** `to_domain` calls
  `Order.rehydrate(...)`; frozen-dataclass validation re-runs on the way in.
- **Locking (Rule 7):** `version_id_col` does the compare-and-set; the boundary
  surfaces a domain-neutral conflict → 409.
- **Atomic op (Rule 8):** one `UPDATE ... SET x = x + 1` behind a named method.
- **Vernon query (Rules 9, 10):** `OrderPage`/`OrderSummary` are frozen
  dataclasses in domain — not DTOs; the application service maps them onward if
  the api needs a different shape.

## Python idioms

- The tuple-typed `items` field is the frozen-dataclass spelling of an
  immutable collection; a `list` field on a frozen dataclass is still mutable
  inside.
- `Session` arrives through the constructor from the provider in
  `dependencies.py`; the repository never opens transactions —
  `session.begin()` belongs to the application service (per `ddd-services`).
- Migrations: Alembic, versioned revisions — mechanics in the per-stack
  persistence skill.
- Repository tests split: contract tests run callers against an in-memory
  fake; implementation tests hit a real PostgreSQL (or SQLite where faithful)
  for the mapping, the lock, and the atomic UPDATE.
- mypy/pyright in CI is what makes the Protocol contract real.
