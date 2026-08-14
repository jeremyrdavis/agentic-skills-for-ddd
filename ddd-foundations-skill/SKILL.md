---
name: ddd-foundations
description: Use when designing the module layout for a new bounded context, placing a class or type (entity, value object, repository, domain or application service, HTTP resource/controller), deciding which layer imports which, or naming a "service" class (must qualify as Application or Domain). Trigger on module/package/directory declarations like <bc>/{domain, application, infrastructure, api}, on an import of a persistence framework (ORM) or HTTP framework inside a domain type, on a Repository interface placed in infrastructure/, or on a class named simply "FooService" without a layer qualifier. Language-agnostic — applies to any stack; the mechanics of a given layer live in the per-stack skills (e.g. quarkus-*, fastapi-*, express-*), and aggregate-internal design lives in ddd-aggregates. This skill is the layering contract every other DDD skill in this set references.
---

# DDD Foundations Skill

The layering map and term definitions for any DDD bounded context, in any language. Every other DDD skill in this set references the four layers and the canonical names below. Packages, modules, namespaces, and directories are all "modules" here — use whatever your language provides to make the boundaries real.

**Foundational principle.** Domain depends on nothing in your code — only the language's standard library and your own domain types. Infrastructure and api depend inward. The word "service" without a qualifier is meaningless: say *application service* (use-case orchestration, owns the transaction, returns DTOs) or *domain service* (stateless domain logic that doesn't naturally belong to one aggregate), and put each in the layer named after it. The layers are enforced by module boundaries and architecture tests — violations show up as imports, not as code review findings.

**Red Flags — STOP if you find yourself thinking:**

- "Repository = persistence = infrastructure, so the interface goes there too."
- "It feels tidy to put the interface next to its implementation."
- "It's just a service — `service/` is fine for both kinds."
- "We're a small team / it's a small BC, let's keep it flat in one module and split later."
- About to import a persistence framework (ORM, query builder) or HTTP framework (router, request/response types) inside a type under `<bc>/domain`.
- About to import anything from `<bc>/infrastructure` in a type under `<bc>/domain` or `<bc>/application`.
- About to name a new class `FooService` without `Application` or `Domain` in the name.

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing the module layout for a new bounded context.
- Placing a type (entity, value object, repository, domain service, application service, HTTP resource/controller/route handler, error mapper).
- Naming a "service" class — choosing the layer-qualified suffix.
- Reviewing a diff that imports an ORM or HTTP framework in a `<bc>/domain` type.
- Reviewing a diff that puts a Repository interface in `<bc>/infrastructure`.
- Reviewing a diff where an HTTP resource calls a repository directly (skipping the application service).

**Out of scope**: the framework mechanics inside any one layer (use the corresponding per-stack skill, e.g. `quarkus-persistence`, `fastapi-rest`, `express-rest`); aggregate-internal design — invariants, factories, identity strategy (use `ddd-aggregates`); domain-event payload and dispatch rules (use `ddd-domain-events`); choosing between active-record and repository pattern at the persistence layer (architectural decision; the per-stack persistence skill covers the *mechanics* of either path).

---

## Core Rules

1. **Four layers per bounded context**: `domain`, `application`, `infrastructure`, `api`. One bounded context = one top-level module; one set of layers under it.
2. **Domain depends on nothing in your code.** Only standard-library types and types under `<bc>/domain` itself.
3. **Application depends on domain.** Application services orchestrate use cases by calling domain types and domain interfaces.
4. **Infrastructure depends on domain.** Infrastructure provides *implementations* of domain interfaces (repositories, external clients).
5. **Api depends on application.** HTTP resources / gRPC handlers / CLI commands call application services and pass DTOs back and forth.
6. **Never the reverse.** Domain must not import application, infrastructure, or api. Application must not import infrastructure or api.
7. **Repository interfaces live in `<bc>/domain`**, alongside the aggregate they serve. The interface mentions only domain types — no ORM, no persistence framework, no driver types.
8. **Repository implementations live in `<bc>/infrastructure`.** They implement the domain interface and bring in framework concerns. The composition root (or DI container) wires them at runtime.
9. **Application service** = use-case orchestrator. Owns the transaction boundary. Returns DTOs, never entities. Lives in `<bc>/application`. Class name ends in `ApplicationService` (e.g. `AuthorizePaymentApplicationService`).
10. **Domain service** = stateless domain logic that doesn't naturally belong to one aggregate. No I/O, no framework calls; takes domain types and returns domain types. Lives in `<bc>/domain`. Class name ends in `DomainService` (e.g. `FraudDetectionDomainService`).
11. **Never name a class just `FooService`.** The unqualified name is the layer-confusion smell that lets transaction logic and domain rules drift between files in a shared `service/` folder.
12. **HTTP resources live in `<bc>/api`** and depend only on `<bc>/application`. They never construct or call a repository or an entity directly.
13. **DTOs (commands and read models) live in `<bc>/application`.** They cross the application↔api boundary. The domain stays in domain types.

For the worked example — full module tree and idiomatic code for a `payments` bounded context — read the reference file matching the working language: `references/canonical-example-java.md`, `references/canonical-example-python.md`, or `references/canonical-example-typescript.md`. Load exactly one, and only when scaffolding a new bounded context; the rules and tables in this file are sufficient for placement and review decisions. If the working language has no file, read the closest listed one and adapt the idioms — the layering contract is identical in every language.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| Put `PaymentRepository` (interface) in `<bc>/infrastructure` next to its implementation | Inverts the dependency: `domain.Payment` would have to import from infrastructure to refer to its own persistence contract. The layering map breaks. | Move the interface to `<bc>/domain` next to the aggregate it serves. Only the implementation belongs in `infrastructure`. |
| Name a class `OrderService` (no qualifier) | Hides whether it's orchestration or domain logic. The two end up in the same folder and rules drift between them — transaction handling creeps into rule logic, or rules creep into orchestration. | Pick the qualifier and the layer: `OrderApplicationService` in `application/` *or* `OrderDomainService` in `domain/`. |
| Domain type importing an ORM or HTTP framework | Domain now depends on the persistence framework / HTTP framework. ORM upgrades, mapping-strategy changes, and column-naming policies cascade through the model. The "domain depends on nothing" rule is broken at the source. | Domain stays a plain object. The framework-mapped mirror type lives in `<bc>/infrastructure`, and the repository implementation translates between domain aggregates / value objects and persistence types (covered in `ddd-aggregates`, `ddd-value-objects`, `ddd-repositories`). |
| Application service importing `<bc>/infrastructure` types directly | Inverts dependency direction; `application → infrastructure` is forbidden. The application now knows about the ORM. | Application imports the *domain interface* (`PaymentRepository`); the composition root wires the infrastructure implementation at runtime. The application never names the implementation class. |
| Single `service/` module containing both `OrderApplicationService` and `OrderDomainService` | Hides the layer distinction. Reviewers can't tell at a glance which class owns transactions and which owns rules. | Split into `application/` and `domain/`. The two have different jobs and different layer constraints. |
| HTTP resource calling a repository directly | Skips the application service. Transaction boundary, DTO mapping, and use-case logic happen in the wrong layer. | Resource depends only on `application`. Application service owns the orchestration. |
| One module per technical kind (`controllers/`, `services/`, `repositories/`, `entities/`) at the bounded-context root | This is "horizontal slicing" — it scatters one feature across four modules and makes layer violations invisible. The DDD layout is "vertical": one BC, four layers. | Adopt the four-layer structure. Each BC is self-contained vertically. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "Repository = persistence = infrastructure. The interface goes next to its implementation; that's how every framework tutorial does it." | Tutorial-as-authority is the cargo-cult version of architecture. The interface describes a domain contract — what the aggregate needs to be loaded and saved — in domain language. The implementation supplies persistence. Putting the interface in infrastructure forces the domain to import infrastructure to refer to its own contract. The contract belongs with the thing it serves, not the thing that fulfills it. |
| "Putting the interface next to its implementation feels tidy and reduces folder navigation." | "Tidy" is comparing on the wrong axis. The dependency direction is the architectural axis; folder ergonomics is the personal-preference axis. When the two conflict, the dependency direction wins, because broken dependencies show up as import cycles or framework coupling — not as inconvenience. |
| "It's just a service — `service/` is fine for both `ApplicationService` and `DomainService`." | The two have different jobs: orchestration owns transactions and DTOs; domain logic owns invariants and is pure. Lumping them in one folder is the layer-confusion smell that lets transaction logic creep into domain code (and vice versa). The qualifier in the class name *and* the layer in the module together prevent the drift. |
| "We're a small team / small BC — let's keep it flat now and split when it grows." | Dependency violations don't show up in a flat layout because there are no boundaries to enforce them. Flat-now-split-later means the splitting work has to disentangle violations that accumulated. The moment you grow past three classes, the layering map costs less than the cleanup. |
| "The framework wants the entity to extend its base class / carry its mapping annotations — the active-record shape is the framework-suggested one, so the aggregate has to carry the ORM concerns." | The rule is "domain depends on nothing in your code" — and that includes the ORM. The aggregate is a plain object in `<bc>/domain`; the framework-mapped entity is a *separate* type in `<bc>/infrastructure`; the repository implementation translates. Active-record shapes are for *persistence types* — perfectly fine on the infrastructure-layer mirror class. Conflating the domain aggregate and the ORM entity into one type is a framework-mechanics shortcut for CRUD apps, not a license to ship ORM concerns into the domain layer. (Detailed in `ddd-aggregates`; the persistence-side mechanics live in the per-stack persistence skill.) |

---

## Quick Reference

### Type kind → module

| Type kind | Module |
| --- | --- |
| Entity | `<bc>/domain` |
| Value object | `<bc>/domain` |
| Aggregate root | `<bc>/domain` |
| Repository **interface** | `<bc>/domain` |
| Domain service (stateless, framework-free) | `<bc>/domain` |
| Domain event | `<bc>/domain` |
| Application service (use-case orchestration) | `<bc>/application` |
| Command DTO (input) | `<bc>/application` |
| Read DTO / query result | `<bc>/application` |
| Repository **implementation** (ORM, SQL, HTTP-backed) | `<bc>/infrastructure` |
| External-system adapter (HTTP client, message producer/consumer, LLM client) | `<bc>/infrastructure` |
| HTTP resource / controller / route handler | `<bc>/api` |
| Error mapper / exception handler | `<bc>/api` |

### Type name → suffix

| Type kind | Suffix |
| --- | --- |
| Application service | `*ApplicationService` |
| Domain service | `*DomainService` |
| Aggregate root, entity, value object | (no suffix — name is the noun: `Order`, `Payment`, `Money`) |
| Repository interface | `*Repository` |
| Repository implementation | `<Tech>*Repository` (e.g. `PostgresPaymentRepository`) or `*RepositoryImpl` if the implementation detail isn't worth highlighting |
| Domain event | `*Event`, named in the past tense — `OrderPlacedEvent`, `PaymentAuthorizedEvent`; never `PlaceOrderEvent` |

### Allowed dependency directions

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

Read it as: arrows go from *who knows about whom*. Domain knows about nothing. Application knows about domain. Infrastructure knows about domain (to implement its interfaces). Api knows about application (to call its services). Any other arrow is a layering violation.

### Companion skills

- **`ddd-aggregates`** — aggregate root invariants, identity strategy, factories, cross-aggregate references by ID.
- **`ddd-value-objects`** — value objects, equality by value, validation in canonical constructors.
- **`ddd-services`** — application service vs domain service, in depth.
- **`ddd-repositories`** — repository interface design and the rules about returning aggregates.
- **`ddd-domain-events`** — past-tense `*Event` naming, raise vs dispatch, after-commit publication.
- **Per-stack mechanics skills** (`quarkus-*`, `fastapi-*`, `express-*`, or your stack's equivalents) — the framework mechanics inside each layer: persistence, REST, testing, logging.
