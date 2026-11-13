# Canonical Example — `payments` Bounded Context (Python)

Load this file when scaffolding a new bounded context in Python. For placement
and review decisions, the Core Rules and Quick Reference tables in SKILL.md are
sufficient on their own.

Use case: *authorize a payment* — load the `Payment` aggregate, run a
domain-service fraud check, advance the aggregate's state machine, persist via an
infrastructure repository implementation, return a `PaymentDTO`.

## Module layout

Layers are Python packages (directories with `__init__.py`) under one
bounded-context package:

```
payments/
├── domain/
│   ├── payment_method.py         (value object — frozen dataclass)
│   ├── payment.py                (aggregate root — plain class, stdlib imports only)
│   ├── payment_repository.py     (Protocol — uses only domain types)
│   ├── fraud_detection_domain_service.py
│   └── payment_authorized_event.py   (domain event — frozen dataclass, PaymentAuthorizedEvent)
├── application/
│   ├── commands.py               (AuthorizePaymentCommand — input DTO)
│   ├── dtos.py                   (PaymentDTO — output DTO)
│   └── authorize_payment_application_service.py
├── infrastructure/
│   ├── models.py                 (SQLAlchemy-mapped mirror model — ORM lives here)
│   └── payment_repository.py  (implements the domain Protocol, translates)
└── api/
    ├── router.py                 (FastAPI APIRouter)
    └── error_handlers.py         (exception handlers)
```

Allowed dependency directions (one way only):

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

## The key files

### 1. Repository interface — `domain`, speaks only domain language

```python
# payments/domain/payment_repository.py
from typing import Optional, Protocol

from payments.domain.payment import Payment, PaymentId
from payments.domain.order_id import OrderId


class PaymentRepository(Protocol):
    def find_by_id(self, payment_id: PaymentId) -> Optional[Payment]: ...
    def find_by_order_id(self, order_id: OrderId) -> Optional[Payment]: ...
    def persist(self, payment: Payment) -> None: ...
```

No SQLAlchemy, no Pydantic, no driver imports. A `Protocol` means the
infrastructure implementation satisfies it structurally — no inheritance needed.

### 2. Repository implementation — `infrastructure`, brings the framework

```python
# payments/infrastructure/payment_repository.py
from typing import Optional

from sqlalchemy.orm import Session

from payments.domain.payment import Payment, PaymentId
from payments.infrastructure.models import PaymentModel


class PaymentRepository:
    """Satisfies payments.domain.payment_repository.PaymentRepository."""

    def __init__(self, session: Session) -> None:
        self._session = session

    def find_by_id(self, payment_id: PaymentId) -> Optional[Payment]:
        row = self._session.get(PaymentModel, payment_id.value)
        return row.to_domain() if row else None       # persistence -> domain

    def persist(self, payment: Payment) -> None:
        self._session.merge(PaymentModel.from_domain(payment))  # domain -> persistence

    # find_by_order_id elided — same pattern via a select on PaymentModel
```

`PaymentModel` is the SQLAlchemy-mapped mirror class (table, columns, mapper) in
`infrastructure/models.py`. The `to_domain`/`from_domain` translation keeps ORM
concerns out of `domain`.

### 3. Application service — `application`, orchestrates the use case

```python
# payments/application/authorize_payment_application_service.py
from payments.application.commands import AuthorizePaymentCommand
from payments.application.dtos import PaymentDTO
from payments.domain.fraud_detection_domain_service import FraudDetectionDomainService
from payments.domain.payment_repository import PaymentRepository  # the Protocol


class AuthorizePaymentApplicationService:
    def __init__(self, payments: PaymentRepository,
                 fraud: FraudDetectionDomainService,
                 transaction) -> None:
        self._payments = payments          # domain interface — never the impl
        self._fraud = fraud
        self._transaction = transaction    # owns the transaction boundary

    def authorize(self, cmd: AuthorizePaymentCommand) -> PaymentDTO:
        with self._transaction():
            payment = self._payments.find_by_id(cmd.payment_id)
            if payment is None:
                raise PaymentNotFoundError(cmd.payment_id)
            verdict = self._fraud.check(payment, cmd.method)
            payment.authorize(verdict)             # aggregate enforces invariants
            self._payments.persist(payment)
            return PaymentDTO.of(payment)          # DTO out — never the entity
```

Load → call domain service → mutate aggregate → persist → map to DTO.
Transactions and DTO mapping live here and only here.

### 4. HTTP resource — `api`, depends only on `application`

```python
# payments/api/router.py
from fastapi import APIRouter, Depends

from payments.application.authorize_payment_application_service import (
    AuthorizePaymentApplicationService,
)
from payments.application.dtos import PaymentDTO
# never an import from payments.domain or payments.infrastructure

router = APIRouter(prefix="/payments")


@router.post("/{payment_id}/authorize", response_model=PaymentDTO)
def authorize(payment_id: str,
              body: AuthorizeRequest,
              service: AuthorizePaymentApplicationService = Depends(get_service)):
    return service.authorize(body.to_command(payment_id))
```

`get_service` lives in `dependencies.py`, the FastAPI-conventional provider
module: it builds `PaymentRepository` around the request's `Session`
and constructs the application service. App-lifetime objects (engine, settings,
event bus) are built once in `create_app()` in `main.py` — the composition
root. FastAPI's `Depends` appears only in `api` — never in `application`.

## What this demonstrates

- The repository **interface** (a `Protocol`) lives in `domain`; its signatures
  use only domain types.
- The **implementation** lives in `infrastructure` and carries all framework
  concerns — including the SQLAlchemy mirror model and the translation between
  it and the domain aggregate.
- The application service orchestrates, owns the transaction boundary, depends
  on the domain *Protocol*, and returns DTOs.
- The fraud check is a `DomainService`: pure domain logic, no I/O.
- The domain event is `PaymentAuthorizedEvent` — past tense, `Event` suffix, in `domain`.
- The router lives in `api` and imports only from `application`.

## Python idioms

- Value objects and events are `@dataclass(frozen=True)`. Aggregates are plain
  classes; the interface is a `typing.Protocol` (structural — the impl needs no
  inheritance).
- The composition root is the app factory Pythonistas already know:
  `create_app()` in `main.py` builds app-lifetime objects (engine, settings,
  event bus); per-request wiring lives in provider functions in
  `dependencies.py`. No DI framework anywhere — construction is plain function
  calls and constructor arguments. FastAPI's `Depends` belongs in `api` only.
- Domain purity: no `sqlalchemy`, `pydantic`, `fastapi`, or driver imports under
  `payments/domain/`. Pydantic request/response models live in `api`.
- Enforce with **pytest-archon** (or import-linter): rules forbidding
  `payments.domain` from importing anything but stdlib and itself, and
  `payments.application` from importing `payments.infrastructure` or
  `payments.api`.
