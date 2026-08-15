# Canonical Example — `payments` Bounded Context (Java)

Load this file when scaffolding a new bounded context in Java. For placement and
review decisions, the Core Rules and Quick Reference tables in SKILL.md are
sufficient on their own.

Use case: *authorize a payment* — load the `Payment` aggregate, run a
domain-service fraud check, advance the aggregate's state machine, persist via an
infrastructure repository implementation, return a `PaymentDTO`.

## Module layout

Layers are Java packages under one bounded-context package:

```
com.example.payments
├── domain
│   ├── PaymentMethod.java              (value object — record)
│   ├── Payment.java                    (aggregate root — plain class, no framework imports)
│   ├── PaymentRepository.java          (interface — uses only domain types)
│   ├── FraudDetectionDomainService.java
│   └── PaymentAuthorizedEvent.java     (domain event — record, past tense, *Event suffix)
├── application
│   ├── AuthorizePaymentCommand.java    (input DTO — record)
│   ├── PaymentDTO.java                 (output DTO — record)
│   └── AuthorizePaymentApplicationService.java
├── infrastructure
│   ├── PaymentRecord.java              (JPA-mapped mirror entity — jakarta.persistence lives here)
│   └── PostgresPaymentRepository.java  (implements domain PaymentRepository, translates)
└── api
    ├── PaymentsResource.java           (HTTP resource)
    └── PaymentNotFoundMapper.java      (exception mapper)
```

Allowed dependency directions (one way only):

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

## The key files

### 1. Repository interface — `domain`, speaks only domain language

```java
package com.example.payments.domain;

import java.util.Optional;

public interface PaymentRepository {
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId orderId);
    void persist(Payment payment);
}
```

No `jakarta.persistence`, no Panache, no Hibernate. Its vocabulary is the
aggregate's vocabulary.

### 2. Repository implementation — `infrastructure`, brings the framework

```java
package com.example.payments.infrastructure;

import com.example.payments.domain.Payment;
import com.example.payments.domain.PaymentId;
import com.example.payments.domain.PaymentRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.EntityManager;
import java.util.Optional;

@ApplicationScoped
public class PostgresPaymentRepository implements PaymentRepository {

    private final EntityManager em;

    PostgresPaymentRepository(EntityManager em) { this.em = em; }

    @Override
    public Optional<Payment> findById(PaymentId id) {
        return Optional.ofNullable(em.find(PaymentRecord.class, id.value()))
                .map(PaymentRecord::toDomain);          // persistence -> domain
    }

    @Override
    public void persist(Payment payment) {
        em.merge(PaymentRecord.fromDomain(payment));    // domain -> persistence
    }

    // findByOrderId elided — same pattern via a typed query on PaymentRecord
}
```

`PaymentRecord` is the JPA-mapped mirror entity (`@Entity`, `@Table`, column
mappings) and it lives in this package. The `toDomain`/`fromDomain` translation
keeps ORM concerns out of `domain`. CDI wires this class as *the*
`PaymentRepository`; nothing outside `infrastructure` ever names it.

### 3. Application service — `application`, orchestrates the use case

```java
package com.example.payments.application;

import com.example.payments.domain.*;   // domain types and interfaces — never the impl
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class AuthorizePaymentApplicationService {

    private final PaymentRepository payments;            // domain interface
    private final FraudDetectionDomainService fraud;     // domain service

    AuthorizePaymentApplicationService(PaymentRepository payments,
                                       FraudDetectionDomainService fraud) {
        this.payments = payments;
        this.fraud = fraud;
    }

    @Transactional                                       // owns the transaction boundary
    public PaymentDTO authorize(AuthorizePaymentCommand cmd) {
        Payment payment = payments.findById(cmd.paymentId())
                .orElseThrow(PaymentNotFoundException::new);
        FraudVerdict verdict = fraud.check(payment, cmd.method());
        payment.authorize(verdict);                      // aggregate enforces invariants
        payments.persist(payment);
        return PaymentDTO.of(payment);                   // DTO out — never the entity
    }
}
```

Load → call domain service → mutate aggregate → persist → map to DTO.
Transactions and DTO mapping live here and only here.

### 4. HTTP resource — `api`, depends only on `application`

```java
package com.example.payments.api;

import com.example.payments.application.AuthorizePaymentApplicationService;
import com.example.payments.application.PaymentDTO;
// never an import from ..payments.domain or ..payments.infrastructure
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.core.Response;

@Path("/payments")
public class PaymentsResource {

    private final AuthorizePaymentApplicationService authorizePayment;

    PaymentsResource(AuthorizePaymentApplicationService authorizePayment) {
        this.authorizePayment = authorizePayment;
    }

    @POST
    @Path("/{id}/authorize")
    public Response authorize(@PathParam("id") String id, AuthorizeRequest body) {
        PaymentDTO dto = authorizePayment.authorize(body.toCommand(id));
        return Response.ok(dto).build();
    }
}
```

The resource never sees `Payment`, `PaymentRepository`, or
`PostgresPaymentRepository`. Request parsing, status codes, and the exception
mapper in this package are its whole job.

## What this demonstrates

- The repository **interface** lives in `domain`; its signatures use only domain types.
- The **implementation** lives in `infrastructure` and carries all framework
  concerns — including the JPA mirror entity `PaymentRecord` and the translation
  between it and the domain aggregate.
- The application service orchestrates, owns `@Transactional`, depends on the
  domain *interface*, and returns DTOs.
- The fraud check is a `DomainService`: pure domain logic, no I/O.
- The domain event is `PaymentAuthorizedEvent` — past tense, `*Event` suffix, in `domain`.
- The resource lives in `api` and imports only from `application`.

## Java idioms

- Value objects, events, and DTOs are `record`s. Aggregates are plain classes
  with state transitions as methods.
- CDI (or Spring's container) is the composition root: the constructor-injected
  `PaymentRepository` resolves to `PostgresPaymentRepository` at runtime.
- Domain purity: no `jakarta.persistence.*`, `jakarta.ws.rs.*`, `io.quarkus.*`,
  or `org.springframework.*` imports under `<bc>.domain`.
- Enforce with **ArchUnit**: a layered-architecture rule per bounded context plus
  a rule that `..domain..` classes depend only on `..domain..` and `java..`.
