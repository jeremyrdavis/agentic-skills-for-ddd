# Canonical Example — The Refund Triangle (Python)

Load this file when creating a new service or splitting a mixed one in Python.
For placement and review decisions, the Core Rules and Quick Reference tables
in SKILL.md are sufficient on their own.

The "process refund" use case: three classes, three roles. The `Order`
aggregate is the one from `ddd-aggregates`' canonical example, extended with a
`refund` transition. Comments cite rule numbers.

## 1. The aggregate's domain method (in `<bc>/domain`; full design in `ddd-aggregates`)

```python
# orders/domain/order.py — sketch of the relevant method only
def refund(self, amount: Money) -> None:
    if self._status not in (OrderStatus.PLACED, OrderStatus.FULFILLED):
        raise OrderRefundNotAllowedError(OrderId(self._id))
    if amount.amount > self._total.amount:
        raise ValueError("refund amount exceeds order total")
    self._status = OrderStatus.REFUNDED
    self._refunded_amount = amount
    self._pending_events.append(OrderRefundedEvent(OrderId(self._id), amount))
```

Enforces the *invariant* (Rule 7): the order is in a refundable state, the
amount doesn't exceed the total. The aggregate **does not compute** the amount;
it accepts the value and verifies the resulting state is legal.

## 2. The domain service (in `<bc>/domain`)

```python
# orders/domain/refund_calculator_domain_service.py
from datetime import datetime

from orders.domain.money import Money
from orders.domain.order import Order
from orders.domain.refund_policy import RefundPolicy   # a frozen dataclass — a DOMAIN value


class RefundCalculatorDomainService:
    """Pure computation: domain in, domain out (Rule 4). No repository, no
    transaction, no event publisher (Rule 5). An instance, not module-level
    functions, so it can carry configured policy (Rule 9)."""

    def __init__(self, policy: RefundPolicy) -> None:   # configured domain value — allowed (Rule 5)
        self._policy = policy

    def compute(self, order: Order, now: datetime) -> Money:
        base = order.total
        prorated = self._apply_time_proration(base, order.placed_at, now)
        after_restocking = self._apply_restocking_fees(prorated, order.lines)
        return self._round_to_currency_fraction_digits(after_restocking)
        # ... 80 lines of pure rules: prorated within policy.window_days,
        # restocking fees on opened SKUs, HALF_EVEN currency rounding, etc.

    def _apply_time_proration(self, base: Money, placed_at: datetime, now: datetime) -> Money: ...
    def _apply_restocking_fees(self, prorated: Money, lines: tuple) -> Money: ...
    def _round_to_currency_fraction_digits(self, m: Money) -> Money: ...
```

Unit-testable with a real `Order` and a fixed `datetime` — no database, no app
fixture, no framework.

## 3. The application service (in `<bc>/application`)

```python
# orders/application/process_refund_application_service.py
from orders.application.dtos import RefundDTO
from orders.domain.errors import OrderNotFoundError
from orders.domain.order_id import OrderId
from orders.domain.order_repository import OrderRepository            # the Protocol
from orders.domain.refund_calculator_domain_service import RefundCalculatorDomainService


class ProcessRefundApplicationService:
    def __init__(self, orders: OrderRepository,                       # domain interface (Rule 3)
                 calculator: RefundCalculatorDomainService,
                 events,        # the event publisher from shared/events
                 clock,         # callable returning datetime — injected so tests can fix time
                 transaction) -> None:                                # owns the boundary (Rule 8)
        self._orders = orders
        self._calculator = calculator
        self._events = events
        self._clock = clock
        self._transaction = transaction

    def refund(self, order_id: OrderId) -> RefundDTO:
        with self._transaction():                                     # here, only here (Rule 8)
            order = self._orders.find_by_id(order_id)
            if order is None:
                raise OrderNotFoundError(order_id)

            amount = self._calculator.compute(order, self._clock())   # domain-service call (Rule 7)
            order.refund(amount)                                      # aggregate transition
            self._orders.persist(order)

            for event in order.pull_pending_events():                 # drain and hand off (Rule 11)
                self._events.publish(event)

        return RefundDTO.of(order, amount)                            # DTO out — never the aggregate (Rule 10)
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

The three classes have different dependency signatures, and the package each
lives in enforces them (per `ddd-foundations` and the pytest-archon rules). The
aggregate depends on nothing beyond domain types. The domain service has no
I/O dependencies. The application service is where framework concerns
aggregate. If you find yourself passing a session or a repository into a class
in `orders/domain/`, you've discovered the class is actually an application
service — move and rename it.

## Python idioms

- The composition root is `boot/`: construct `RefundPolicy` from configuration
  there, build `RefundCalculatorDomainService(policy)` once, and inject it —
  the domain service never reads config APIs or environment variables itself.
- Don't flatten the domain service into module-level functions (Rule 9): the
  instance carries the configured policy and can be substituted in tests;
  bare functions can do neither without global state.
- `clock` as a zero-arg callable (`datetime.now` in boot, a lambda returning a
  fixed value in tests) is the lightweight Python spelling of an injected time
  source.
- FastAPI's `Depends` stays in `api/` (per `ddd-foundations`); the application
  service receives everything through its constructor.
- Testing split: domain-service tests are pure Python; application-service
  tests use an in-memory `OrderRepository` fake and a capturing publisher.
- Events: past tense, `*Event` suffix (`OrderRefundedEvent`), per
  `ddd-foundations` and `ddd-domain-events`.
