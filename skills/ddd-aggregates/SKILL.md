---
name: ddd-aggregates
description: Use when designing or modifying an aggregate root — picking the identity strategy, writing the creation factory or the rehydrate() method the persistence layer reconstructs from, declaring field visibility, enforcing an invariant or state-machine transition inside the root, referencing another aggregate, carrying a version field for optimistic locking, or raising a domain event from inside the aggregate. Trigger on ORM imports, persistence base classes, or session/connection references in a <bc>/domain type (the aggregate is a plain object; persistence types live in <bc>/infrastructure), on public mutable fields or setters on an aggregate, *Validator companions, direct object references between aggregate roots (a Customer field on Order), or direct constructor calls on an aggregate outside its factory. Excludes layering (ddd-foundations), value-object internals (ddd-value-objects), the persistence mirror and repository implementation (ddd-persistence), and event payload/dispatch (ddd-domain-events).
---

# DDD Aggregates Skill

How to design an aggregate root, in any language. The aggregate is **a plain object** in `<bc>/domain` that enforces its own invariants and is the boundary of one transaction. The persistence-layer mirror type is a *separate* type in `<bc>/infrastructure` (covered by `ddd-persistence`); the aggregate has no knowledge of it.

**Foundational principle.** Domain depends on nothing in your code (per `ddd-foundations`) — and that includes the ORM. The aggregate is a plain object with no persistence annotations, no framework base class, no session or connection reference. The aggregate enforces its own invariants; anything outside it that "validates" before mutating is a sign the invariant doesn't actually live there. The aggregate's public surface is the set of legal operations; private state plus named domain methods is how you guarantee that surface is exhaustive. Public fields, public setters, external `*Validator` classes, and ORM concerns on the aggregate are different shapes of the same anti-pattern: they let infrastructure or untrusted code shape the domain layer.

**Red Flags — STOP if you find yourself thinking:**

- About to import an ORM, persistence framework, or database driver in a type under `<bc>/domain`.
- About to extend a persistence framework base class on an aggregate.
- About to declare a public mutable field on an aggregate ("that's the framework's idiom").
- About to add `setStatus(...)` (or any other setter) on an aggregate root.
- About to add a separate `*Validator` class that "checks the entity is valid" before persistence.
- About to reference another aggregate root by object (a `Customer` field) instead of by ID (`CustomerId`).
- About to construct an aggregate directly from a service instead of through its named factory (`Order.place(...)`).
- About to put transaction demarcation on a method *inside* the aggregate — the aggregate is a plain object and doesn't know about transactions.
- "Mapping the aggregate to a separate persistence type duplicates the field list — DRY says one type."
- "The framework's idiom is public fields / active record — why fight the framework?"
- "It's a small / CRUD-shaped service — the rich-aggregate ceremony is over-engineering."

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing a new aggregate root or modifying an existing one.
- Picking the identity strategy: typed ID wrapper around a UUID, externally-assigned identifier, or — only when the ID has no domain meaning — a plain integer.
- Writing the factory method that constructs a valid initial state.
- Writing the `rehydrate(...)` method the persistence layer reconstructs from.
- Deciding whether an invariant lives in the aggregate or somewhere else.
- Choosing how to reference another aggregate root (always by ID).
- Adding an integer `version` field for optimistic locking.
- Raising a domain event from inside an aggregate when state changes.
- Reviewing a diff with any ORM import, persistence base class, or session reference on a type in `<bc>/domain` — that's a category error this skill prevents (placement itself is `ddd-foundations`; this skill supplies the correct aggregate shape).

**Out of scope**: layering and module placement (use `ddd-foundations`); the internal shape and equality of value objects, including the typed ID wrappers `OrderId`, `CustomerId` (use `ddd-value-objects`); the persistence-layer mirror type, the repository interface, and the repository implementation that translates between domain and persistence types (use `ddd-persistence`); domain-event payload conventions and dispatch mechanics (use `ddd-domain-events`); application-service orchestration (use `ddd-services`) and the framework's transaction mechanics (use `<stack>-ddd`).

---

## Core Rules

1. **The aggregate is a plain object.** No ORM imports, no persistence annotations, no framework base class, no session/connection reference. It lives in `<bc>/domain` and depends on the language's standard library plus its own domain types (per `ddd-foundations`). The persistence-layer mirror type is a **separate type** in `<bc>/infrastructure`, and the repository implementation maps between them (covered by `ddd-persistence`).
2. **The aggregate is the only entry point to its internal state.** No public mutable fields, no setters. Mutation happens through named domain methods (`cancel()`, `markFulfilled()`, `addLine(...)`) that enforce the invariants of that operation.
3. **Private state, intentional accessors.** Fields are private; expose what callers need via read accessors (`status()`, `total()`) — not getters mirroring every field. The aggregate decides what's visible.
4. **Construct via a named factory, never the raw constructor.** A method like `Order.place(id, customerId, lines)` validates inputs and returns a fully-constructed, valid aggregate. The raw constructor is not part of the public surface — hide it with the strongest mechanism the language offers (private constructor where the language has one; documented-non-public convention plus review enforcement where it doesn't — see the per-language reference file).
5. **The factory enforces creation invariants.** Empty line list? Mixed currencies? Missing customer ID? The factory throws — it does not return a partially valid aggregate that a downstream `validate()` will repair.
6. **Provide a `rehydrate(...)` factory for the persistence layer.** The repository implementation reconstructs an existing aggregate from stored state via this method. Rehydration trusts that the data was valid when first persisted; it does not re-run creation side effects (no `OrderPlacedEvent` re-emitted). Domain and application code never call `rehydrate(...)` — only the repository implementation in `<bc>/infrastructure`.
7. **Domain methods enforce transition invariants.** `cancel()` checks the current state and throws if cancellation is illegal *before* mutating. Each method states its precondition and either succeeds in full or throws — no half-applied state changes.
8. **Reference other aggregates by ID, never by object.** `CustomerId customerId`, not a `Customer` field. Object references drag the other aggregate into your transaction (one-aggregate-per-transaction violation), conflate identity with reachability, and turn aggregate boundaries into ORM association graphs — those belong in the persistence layer, never in the domain.
9. **Use typed ID wrappers when the ID has domain meaning.** `OrderId(UUID)`. The wrapper makes `placeOrder(OrderId, CustomerId)` impossible to call with arguments swapped. Prefer it over raw UUID/integer parameters that are easy to confuse. (Defined in `ddd-value-objects`.)
10. **Carry a plain integer `version` field for optimistic locking.** The aggregate stores the version it was loaded with; the persistence layer applies the actual lock mechanics on the mirror type (framework version column, or an explicit `WHERE version = ?` check) and surfaces the conflict; the api layer maps it to 409 Conflict. Skip versioning only when the aggregate is genuinely single-writer.
11. **Hold value objects as plain fields.** `total` is a field of type `Money` (a plain value type from `ddd-value-objects`). No embedding/mapping annotations on the aggregate — mapping directives belong on the persistence-layer mirror type, never on the aggregate.
12. **Raise domain events from inside the aggregate; dispatch them from the application service.** When `cancel()` succeeds it appends an `OrderCancelledEvent` — `eventId` and `occurredAt` assigned at that moment (`ddd-domain-events` Rule 3) — to a `pendingEvents` list on the aggregate. The application service drains that list (`pullPendingEvents()`) and publishes after the transaction commits (see `ddd-domain-events`). Events are named past tense with the `Event` suffix, per `ddd-foundations`.
13. **One aggregate per transaction.** A use case loads, mutates, and persists *one* aggregate plus its value objects. If two aggregates need to change consistently, the model is wrong — raise a domain event and update the second aggregate in a separate transaction from its handler.
14. **No transaction demarcation inside an aggregate.** The aggregate is a plain object; it carries no framework annotations or transaction calls. The transaction boundary lives on the application service (per `ddd-services`; the annotation itself is `<stack>-ddd`'s).

For the worked example — the full `Order` aggregate with factory, `rehydrate`, transition methods, pending events, companion types, and the persistence mirror with idiomatic optimistic locking — read the reference file matching the working language: `references/canonical-example-java.md`, `references/canonical-example-python.md`, or `references/canonical-example-typescript.md`. Load exactly one, and only when creating or substantially reshaping an aggregate; the rules and tables in this file are sufficient for placement and review decisions. If the working language has no file, read the closest listed one and adapt the idioms — the aggregate contract is identical in every language. The example is stack-neutral; its persistence-mirror sketch is illustrative only — the `<stack>-ddd` skill for your stack has the real mapping.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| Declare the aggregate as an ORM entity (mapping annotations, framework base class) | Domain depends on the ORM. The aggregate's design is now constrained by persistence concerns: framework-required constructors, public mutable fields for the active-record idiom, collection-mapping shape pressure, lazy-fetch surprises, cascade footguns. The "domain depends on nothing" rule from `ddd-foundations` is broken at the source. | The aggregate is a plain object in `<bc>/domain`. A separate mirror type lives in `<bc>/infrastructure` (covered by `ddd-persistence`) with the mapping concerns; the repository implementation translates. |
| Embedding/mapping annotations on the aggregate's value-object fields | Mapping directives are persistence concerns; the aggregate now imports the ORM. Same root violation. | The aggregate holds a plain field of a plain value type. The persistence-layer mirror carries the embedding or split-column mapping — that's the repository's call. |
| The framework's version/lock annotation on the aggregate | Lock detection is framework semantics; the domain shouldn't know about it. | The aggregate carries a plain integer `version`. The mirror type carries the lock mechanics. Equivalent data; one layer wears the framework. |
| A do-nothing constructor kept "for the framework only" | A constructor whose only justification is the persistence framework is the clearest signal the class is being shaped by infrastructure. | The aggregate's constructor serves the factory and `rehydrate(...)` only. The mirror type has whatever constructor its framework requires; that's a separate file. |
| Public mutable fields on the aggregate ("the framework's idiom") | Encapsulation lost; invariants become advisory. The "framework idiom" argument is a category error — active-record conveniences are persistence-layer conveniences, not domain-modeling guidelines. | Private fields, public domain methods. The framework's shape lives on the mirror type. |
| `setStatus(s)` (or any setter) on an aggregate | A setter is a domain method with no domain meaning. It accepts any input, including ones that violate the state machine. | Named methods: `cancel()`, `markFulfilled()`. The method's name describes the rule. |
| Separate `OrderValidator` class called by the application service | The invariant is enforced *only if* callers remember the validator. That's the definition of an invariant that isn't one. | Move the validation into the aggregate's factory and domain methods. The aggregate guarantees its own validity. |
| A `Customer` object field on `Order` (direct reference between aggregate roots) | Drags `Customer` into `Order`'s transaction (one-aggregate-per-transaction violation); lazy-load surprises and cascade footguns follow — now living in your domain. | Reference by ID: `CustomerId customerId`. Need Customer data? Look it up in the application service via the Customer repository and pass the values in. |
| Raw UUID/integer parameters in pairs (`placeOrder(id, id2)`) | Loses domain-typed identity; the call is silently swappable at no compile-time cost. | Typed wrappers at API surfaces: `placeOrder(OrderId, CustomerId)`. (Defined in `ddd-value-objects`.) |
| Direct construction from a service, fields assigned one by one | Bypasses the factory. The aggregate exists in an invalid intermediate state during assignment — half the invariants are broken before the last line runs. | Named factory: `Order.place(...)` validates inputs and returns the fully-constructed aggregate. The raw constructor stays out of the public surface. |
| Transaction demarcation on a method inside the aggregate | Couples the plain domain object to a framework. | The transaction boundary belongs on the application service (per `ddd-services`; the annotation itself is `<stack>-ddd`'s). |
| The aggregate loading other objects through the persistence layer (repository/finder calls inside domain methods) | The aggregate is reaching into infrastructure to mutate itself. Repository coupling and a circular dependency follow. | Aggregate methods take what they need as parameters. The application service does the loading and passes domain values in. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "The framework's active-record shape makes public fields idiomatic — extending the persistence base class is the framework-suggested shape for any persistent type." | Domain depends on nothing in your code (per `ddd-foundations`) — and that includes the ORM. The aggregate is a plain object in `<bc>/domain`; the persistence type is a separate class in `<bc>/infrastructure`; the repository implementation translates. The "framework-suggested shape" is for *persistence types*, not domain aggregates. Conflating them lets ORM concerns (lazy-fetch surprises, cascade footguns, framework-required constructors, public-fields-for-the-idiom) shape your invariant-enforcement layer. Active record is a mechanics shortcut for CRUD apps; for an aggregate with state-machine transitions, cross-aggregate references, and domain events, the cost of conflation eclipses the cost of the mapper. |
| "Mapping the aggregate to a separate persistence type duplicates the field list — DRY says one type." | The duplication is the cost of separating concerns. The aggregate evolves on domain rules (state-machine changes, new invariants, ubiquitous-language renames); the persistence type evolves on schema rules (column renames, indexes, denormalization, soft-delete columns). When the two diverge — and they will — one shared type forces every change to compromise both. DRY applies to *knowledge*, not field lists; the two types know different things. The mapping cost is real and bounded; the conflation cost is unbounded and shows up at the worst time. |
| "But the ORM needs a no-arg constructor / a specific shape — plain objects with hidden constructors won't work." | The ORM needs its shape *on the persistence type it manages* — the mirror class — not on the aggregate. The aggregate's constructor serves `Order.place(...)` and `Order.rehydrate(...)`. The mirror type in `<bc>/infrastructure` has whatever shape the framework requires; that's its problem, not the aggregate's. Two types, two responsibilities. |
| "Hidden constructor + a named factory + a typed `CustomerId` + a `Money` value object feels like over-engineering for a CRUD-shaped service." | If the requirements list a state-machine transition with a typed error, a cross-aggregate reference, a multi-field invariant (currency match), and domain events, you don't have a CRUD shape — you have an aggregate. The "ceremony" is load-bearing for every one of those bullets. The over-engineering smell is real for a 3-field lookup table; not real here. |
| "Validation belongs in a separate `Validator` class for testability." | An invariant that's enforced only when the caller remembers to invoke a validator is not enforced. Testability doesn't improve by moving rules outside the object that owns them — a factory and a `cancel()` method are trivially unit-testable with no database, no framework bootstrap, no test containers, because the aggregate is a plain object. The validator pattern is a workaround for languages without exceptions and failing constructors; you have both. |
| "Direct object references between aggregates are simpler than threading IDs through the codebase." | Simpler at one call site, disastrous everywhere else. A `Customer` field on `Order` means every transaction touching an `Order` also pulls a `Customer` row; a stale `Customer` from another transaction can sneak into the write; cascade behavior now has to be designed across two aggregate boundaries. ID + lookup in the application service is the boring answer that scales. |
| "Setters are how every ORM tutorial models entities — fighting the convention costs more than it saves." | Tutorials model *records of data*; aggregates model *invariants*. If the type is genuinely just a row (a `Country` lookup), setters are fine — it's also probably not an aggregate. If the type has rules about legal states and transitions, a setter is a vehicle for breaking those rules silently. Tutorials don't distinguish; you have to. |
| "Adding a `version` field everywhere is paranoid — most updates won't conflict." | The aggregate's integer version is a one-line cost; the corresponding lock check on the persistence side is the one-line safety net that turns silent corruption (last-write-wins) into a detectable conflict the api layer maps to 409. Skip it only when the aggregate is genuinely single-writer (a per-user cart never touched by a background job). For everything else, the field costs less than the incident. |

---

## Quick Reference

### Aggregate-root commit checklist

When you commit a new aggregate root, every line below should be true:

- [ ] The type is a **plain object**: zero ORM imports, no persistence annotations, no framework base class, no session/connection reference.
- [ ] It lives in `<bc>/domain` (per `ddd-foundations`).
- [ ] Construction is via a named factory (`Order.place(...)`) plus `rehydrate(...)` for the persistence layer; the raw constructor is not part of the public surface (strongest mechanism the language offers).
- [ ] No public mutable fields.
- [ ] No setters. Mutation is through named domain methods.
- [ ] References to other aggregate roots are by typed ID (`ddd-value-objects`), not by object.
- [ ] A plain integer `version` field for optimistic locking (the persistence mirror carries the lock mechanics).
- [ ] Value objects held as plain fields — no mapping annotations on the aggregate.
- [ ] Invariants enforced inside the factory (creation) and inside transition methods (mutation).
- [ ] No `*Validator` companion class enforcing rules the aggregate should self-enforce.
- [ ] No transaction demarcation inside the aggregate.
- [ ] Domain events (past tense, `*Event` suffix) appended to `pendingEvents` from inside the methods that change state.
- [ ] `pullPendingEvents()` (or equivalent) returns and clears the list for the application service to drain.

### Where the persistence-layer mirror lives

| Concern | Type | Module | Carries framework concerns? |
| --- | --- | --- | --- |
| Domain aggregate (invariants, transitions, events) | `Order` (plain object) | `<bc>/domain` | **No** |
| Persistence mirror (table mapping, lock column) | `OrderRecord` (or similar) | `<bc>/infrastructure` | **Yes** |
| Repository interface (uses domain types) | `OrderRepository` | `<bc>/domain` | **No** |
| Repository implementation (translates aggregate ↔ mirror) | `OrderRepository` (same name; the module path distinguishes) | `<bc>/infrastructure` | **Yes** |

The mapping `Order ↔ OrderRecord` lives in the repository implementation. Covered in `ddd-persistence`.

### Cross-aggregate reference policy

| Need | Do |
| --- | --- |
| Identify the related aggregate | Hold the typed ID: `CustomerId customerId`. |
| Read data from the related aggregate | Look it up in the application service via its repository; pass the value(s) you need into the aggregate method. |
| Mutate the related aggregate atomically with this one | You can't — and shouldn't. Raise a domain event; a handler mutates the second aggregate in a separate transaction. |
| Display fields from the related aggregate to a client | Map to a DTO in the application service; the api layer sees both as DTO fields. |

### Companion skills

- **`ddd-foundations`** — the layering map (`domain` / `application` / `infrastructure` / `api`) and where this aggregate lives.
- **`ddd-value-objects`** — the `Money`, `OrderId`, `CustomerId` types this aggregate holds as plain fields.
- **`ddd-persistence`** — the `OrderRepository` interface (in `<bc>/domain`) and the mirror type + repository implementation (in `<bc>/infrastructure`) that translates between them.
- **`ddd-domain-events`** — event payload conventions and how the application service dispatches after commit.
- **`ddd-services`** — the application service that loads, orchestrates, and persists this aggregate.
- **`<stack>-ddd`** — the mechanics skill for your stack (`quarkus-ddd`, `spring-ddd`, `fastapi-ddd`, ...), if installed: the transaction annotation on the application service, the mirror's lock column and its mapping to 409, and the test tiers. The aggregate itself is stack-neutral — its worked example stays here in `references/`.
