---
name: ddd-services
description: Use when designing or naming a "service" class — deciding whether the responsibility is application-service orchestration (loads aggregates, owns the transaction, dispatches events, returns DTOs) or domain-service computation (pure logic over domain inputs, no I/O, no framework). Trigger on a class named *Service with no Application/Domain qualifier, transaction demarcation on a *DomainService, an injected repository, event publisher, or I/O client inside a *DomainService, a *DomainService located in <bc>/application (or a *ApplicationService in <bc>/domain), a 50+-line block of business rules as a private method inside an application service, or business rules computed inside an aggregate method that should just enforce a state transition. Excludes the layering map (ddd-foundations), aggregate-internal design (ddd-aggregates), repository interface design (ddd-repositories), event payload and dispatch mechanics (ddd-domain-events), and per-stack transaction/DI mechanics.
---

# DDD Services Skill

Two kinds of services, two layers, two jobs — in any language. **Application services orchestrate** (load, mutate, persist, dispatch — own the transaction). **Domain services compute** (pure logic over domain inputs, no I/O, no framework). The class-name suffix and the module must agree.

**Foundational principle.** Aggregates guard invariants. Domain services compute across inputs. Application services orchestrate use cases. Each belongs in a different layer with different dependencies allowed. A class named `*DomainService` that owns a transaction and injects a repository is an application service wearing the wrong hat — **the suffix is a label, not a license; the contract is the dependency signature.** A class named just `*Service` (no qualifier) is the layer-confusion smell that lets transaction logic creep into pure rules and pure rules creep into orchestration.

**Red Flags — STOP if you find yourself thinking:**

- About to name a class `OrderService` (no `Application` or `Domain` qualifier) — pick one and put it in the matching module.
- About to put transaction demarcation on a class named `*DomainService`.
- About to inject a repository, an event publisher, an HTTP client, a message producer, or any other I/O dependency into a `*DomainService`.
- About to put a `*DomainService` in `<bc>/application` (or a `*ApplicationService` in `<bc>/domain`).
- About to inline 50+ lines of business rules as a private method inside an application service.
- About to put a multi-line policy calculation inside an aggregate method that should just enforce a state transition (e.g. an 80-line refund-amount calculation inside `Order.refund(...)`).
- About to write a domain service as a bag of static/module-level functions "to avoid the wiring ceremony" — domain services are instances constructed in the composition root; the discipline is *what's injected into them* (nothing framework-bound), not whether they're wired.

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing a new service class for any use case.
- Naming a class that ends in "Service" — picking `*ApplicationService` vs `*DomainService` vs neither (and putting the logic on the aggregate instead).
- Reviewing a diff with a `*DomainService` that has transaction demarcation or an injected repository.
- Reviewing a diff where business rules are inlined as a private method in an application service (often signals a missing domain service).
- Reviewing a diff where an aggregate method computes policy in addition to enforcing the state transition.

**Out of scope**: layering and module placement (use `ddd-foundations` — this skill assumes the `<bc>/application` and `<bc>/domain` modules and dependency directions); aggregate invariants and state machines (use `ddd-aggregates`); repository interface design (use `ddd-repositories`); domain-event payload and dispatch mechanics (use `ddd-domain-events` — this skill states that the application service drains and dispatches, but the *event shape and after-commit delivery rules* live there); per-stack transaction placement and DI mechanics (use the per-stack persistence skill).

---

## Core Rules

1. **Two kinds of services, two suffixes, two layers.** `*ApplicationService` lives in `<bc>/application`. `*DomainService` lives in `<bc>/domain`. Never name a class just `*Service` — the unqualified name is a layer-confusion smell (per `ddd-foundations`).
2. **Application service = use-case orchestration.** Loads aggregates via repository interfaces, calls domain methods on them, optionally calls a domain service for cross-aggregate computation, persists via the repository, drains `pullPendingEvents()` and hands the events to the publisher, maps the result to a DTO, and returns. Owns the transaction boundary.
3. **Application service may depend on:** domain repository interfaces, domain services, other application services (sparingly), the event publisher, a clock/time source (injected so tests can fix time), configuration values, and infrastructure adapters *via their domain interfaces*. It is the correct place for the transaction boundary and every I/O dependency.
4. **Domain service = pure logic that doesn't naturally belong to one aggregate.** Computes across multiple aggregates / value objects / a clock value / configured domain policy. Returns a domain value (a value object, a verdict, a calculated `Money`). No state changes, no persistence, no I/O.
5. **Domain service may depend on:** other domain services and configuration-supplied *domain* values (e.g. a `RefundPolicy` configured for the BC). It must not depend on repositories, the event publisher, HTTP/message clients, ORM sessions, or transaction demarcation. This is `ddd-foundations`' "domain depends on nothing" expressed at the service level.
6. **Default first: put logic on the aggregate.** Most domain logic belongs inside the aggregate root, on its domain methods, adjacent to the state it operates on. Extract a domain service only when the logic *doesn't naturally belong to one aggregate* — typically because it spans two aggregates or value objects, or needs an external input (a clock value, a configured policy) the aggregate doesn't carry.
7. **Aggregates guard invariants; domain services compute across inputs.** This is the heuristic for choosing between Rule 6 and a domain service. If the operation is "given the current state plus this input, am I allowed to do X?" — that's an aggregate method (the aggregate guards its own transition). If the operation is "given these inputs, what is the answer?" — that's a domain service (computes a result; mutates nothing). `Order.refund(Money amount)` enforces the state transition; `RefundCalculatorDomainService.compute(order, now)` computes the `Money` value the aggregate then accepts.
8. **The application service is the only owner of the transaction boundary.** Not the aggregate (it doesn't know about transactions, per `ddd-aggregates`), not the domain service (it does no I/O), not the api layer (it doesn't manage state).
9. **Domain services are instances, constructed in the composition root — not bags of static/module-level functions.** Registration with a DI container or explicit construction in `boot/` is fine and expected; an instance can carry configured domain policy and be substituted in tests. The "framework-free" discipline is about the *dependency signature* — what's injected into the service — not about whether the composition root manages it.
10. **Application service returns DTOs, never aggregates.** The DTO crosses the application↔api boundary (per `ddd-foundations`); the mapping is the application service's job. The aggregate stays in `domain`.
11. **Events drain here, deliver after commit.** The application service drains `pullPendingEvents()` and hands the events to the publisher inside the use case — but *delivery to handlers happens after the transaction commits*. The mechanism that guarantees after-commit delivery (observer phases, post-commit hooks, an outbox) is per-stack and specified in `ddd-domain-events`; the application service's responsibility is only drain-and-hand-off.

For the worked example — the refund triangle (aggregate method, domain service, application service) with idiomatic wiring, transaction placement, and event hand-off — read the reference file matching the working language: `references/canonical-example-java.md`, `references/canonical-example-python.md`, or `references/canonical-example-typescript.md`. Load exactly one, and only when creating a new service or splitting a mixed one; the rules and tables in this file are sufficient for placement and review decisions. If the working language has no file, read the closest listed one and adapt the idioms — the service contract is identical in every language.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| Name a class `OrderService` (no qualifier) | Hides whether the class orchestrates or computes. Reviewers can't tell at a glance which layer rules apply; the two kinds of logic drift into the same file. | Pick a suffix and a layer: `OrderApplicationService` in `application/` or `OrderDomainService` in `domain/`. |
| Transaction demarcation on a `*DomainService` | The suffix says "pure domain logic, no I/O"; the transaction says "owns I/O." The two contradict — the class is an application service mislabeled. | Remove the transaction. If the method genuinely needs one, it's not a domain service — rename to `*ApplicationService` and move to `<bc>/application`. |
| A repository, event publisher, or HTTP/message client injected into a `*DomainService` | The dependency is I/O-bound; domain services don't do I/O. The load belongs in the application service that *calls* the domain service. | Remove the injection. The application service loads the aggregate, then passes the aggregate (or the values the calculation needs) into the domain service's pure method. |
| A `*DomainService` in the `application` module (or the reverse) | Class name and module disagree, so the module-level dependency rules from `ddd-foundations` can't enforce anything. | Move the file to the matching module. The class name picks the module. |
| An 80-line calculation inlined as a private method inside an application service | The named business concept ("how do we compute a refund?") hides inside orchestration code. It can't be unit-tested without instantiating the application service and all its dependencies; it can't be reused; it grows invisibly. | Extract to a `*DomainService` in `<bc>/domain`. The application service calls it; the calculation is testable with plain domain objects and a fixed time value. |
| The same 80-line calculation inlined inside the aggregate's `refund(...)` method | The aggregate now *computes the policy* in addition to *enforcing the transition* — two responsibilities, two reasons to change. The method needs a time parameter the aggregate doesn't otherwise care about. | The aggregate accepts the *result* (`refund(Money amount)`); the calculation lives in a `*DomainService`. **Aggregates guard invariants; domain services compute across inputs.** |
| One big `RefundService` containing the calculation *and* the orchestration (load / persist / dispatch / transaction) | Two layers of concern in one class in an unmarked module. Tests of the calculation need the whole app wiring; tests of the orchestration touch the calculation accidentally. | Split: `RefundCalculatorDomainService` (pure, in `domain/`) + `ProcessRefundApplicationService` (orchestration, in `application/`). |
| A domain service written as static/module-level functions "to avoid wiring ceremony" | Loses the ability to carry configured domain policy (a `RefundPolicy` constructed once in boot), forces every caller to hard-bind to the function, and prevents substitution when a test or variant needs it. | An instance, constructed in the composition root (DI-registered or explicitly built in `boot/`). The framework-free discipline is *what's injected*, not whether it's wired. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "One service is simpler — splitting into Application + Domain feels like ceremony, especially with only one caller today." | If the calculation is 80 lines of pure rules, it's a cohesive named concept earning its own class — not boilerplate. Inlined as a private method, the named business abstraction (refund policy is a thing the business talks about) hides inside orchestration code, and the next caller — second use case, scheduled job, batch import — has nowhere to reuse it from. The split is the first time the concept gets a name; the name pays for itself by the second caller. |
| "It's named `*DomainService`, so it's a domain service — the suffix is the contract." | The suffix is a label. The contract is the *dependency signature*: a domain service takes domain inputs and returns domain values, with no I/O and no transaction. A `RefundDomainService` that owns a transaction and injects `OrderRepository` is an application service with a misleading name. Rename it (`ProcessRefundApplicationService`) or remove the dependencies — pick one, so the suffix and the behavior agree. |
| "Why extract — it's just a private method? Private methods are fine." | Private methods inside an application service can't be unit-tested in isolation (they require the parent instantiated with all its injected dependencies), can't be reused by another use case without copy-paste, and hide named business concepts behind orchestration. The extraction cost is one class declaration; the testability and readability gains compound from the first reuse. |
| "Why not put the 80-line calculation inside the aggregate's method? The aggregate already knows its own state." | Aggregates guard invariants; domain services compute across inputs. `Order.refund()` should enforce *"this order is in a state that allows refund and the amount doesn't exceed the total"* — that's the invariant. *Computing* the right amount needs a time value, a configured policy, and pricing rules — none of which are the aggregate's invariants. Forcing the aggregate to take a clock parameter for one method is the smell that the calculation belongs elsewhere. |
| "Domain services with no infrastructure dependencies feel pointless — they're just functions in a class." | Yes — functions over domain types, in the domain module, with no framework dependencies. That's the intentional shape: it's what makes them unit-testable with plain objects, reusable across use cases, and stable across infrastructure swaps (change the persistence layer, change the message bus — the domain service doesn't notice). Construct them in the composition root so they can carry configured policy; "just functions" plus a constructor is exactly right. |
| "Tutorials show one `Service` class per feature with everything in it — the framework convention is one-service-per-class." | Tutorial conventions optimize for showing one feature, not for separation of concerns; a 200-line tutorial service is a teaching aid, not a maintenance pattern. The two-services rule makes the layer of every method visible from the file path. Tutorial-as-authority is the cargo-cult version of architecture (see `ddd-foundations` Excuse / Reality). |

---

## Quick Reference

### Decision tree

```
You're about to write a new method. Where does it go?

Q1: Does the method change the state of ONE aggregate while enforcing
    that aggregate's invariants?
    YES → aggregate method (Order.cancel(), Order.refund(Money)).
          Lives on the aggregate. Often returns nothing; throws on
          invariant violation. (Design in ddd-aggregates.)
    NO  → continue.

Q2: Does the method compute a domain value (a Money, a verdict, a
    score, a derived quantity) from one or more aggregates plus
    auxiliary inputs (a clock value, configured policy, value objects)?
    YES → domain service method.
          Lives in <bc>/domain on a *DomainService instance.
          No transaction, no injected I/O.
    NO  → continue.

Q3: Does the method orchestrate a use case end-to-end — load aggregate,
    call its domain method, optionally call a domain service, persist,
    drain and hand off events, map to DTO?
    YES → application service method.
          Lives in <bc>/application on a *ApplicationService.
          Owns the transaction; may depend on anything the
          application layer allows.
    NO  → it's not a service method. Probably belongs in the api layer
          (HTTP-shape concerns), in infrastructure (adapter to an
          external system), or doesn't exist yet.
```

### Allowed dependencies — at a glance

| Class kind | May depend on | Must not depend on | Transaction |
| --- | --- | --- | --- |
| `*ApplicationService` (in `<bc>/application`) | Domain repository interfaces, `*DomainService`, other `*ApplicationService` (sparingly), the event publisher, a clock/time source, configuration values, infrastructure adapters via their domain interfaces | Concrete repository implementations, HTTP framework request/response types | **Owns it** |
| `*DomainService` (in `<bc>/domain`) | Other `*DomainService`, configuration-supplied *domain* values (a configured policy) | Repositories, the event publisher, HTTP/message clients, ORM sessions, api-layer types | Never |
| Aggregate root (in `<bc>/domain`) | Nothing — inputs arrive as method parameters (per `ddd-aggregates`) | Anything | Never |

### Naming and placement

| Class kind | Suffix | Module |
| --- | --- | --- |
| Application service | `*ApplicationService` | `<bc>/application` |
| Domain service | `*DomainService` | `<bc>/domain` |
| (Don't use) | `*Service` (no qualifier) | (n/a — the layer is unclear) |

### Companion skills

- **`ddd-foundations`** — the layering map; the `<bc>/application` and `<bc>/domain` modules and the no-qualifier-`*Service` anti-pattern in summary form.
- **`ddd-aggregates`** — what the aggregate's `refund(Money)` method enforces; the `pendingEvents` / `pullPendingEvents()` pattern this skill's application service drains.
- **`ddd-value-objects`** — the `Money` value the calculator computes and the aggregate accepts.
- **`ddd-repositories`** — the `OrderRepository` interface the application service depends on.
- **`ddd-domain-events`** — the `OrderRefundedEvent` shape (past tense, `*Event` suffix) and the after-commit delivery rules.
- **Per-stack mechanics skills** (`quarkus-*`, `fastapi-*`, `express-*`, or your stack's equivalents) — transaction placement, DI/wiring mechanics, DTO-over-the-wire, and the testing split (pure unit tests for domain services; wired tests for application services).
