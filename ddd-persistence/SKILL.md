---
name: ddd-persistence
description: Use when designing a repository interface or implementation, mapping between domain objects and persistence types, wiring optimistic locking, adding an atomic in-database operation, or writing a query whose result shape isn't an aggregate. Trigger on a repository interface in <bc>/infrastructure or returning ORM/mirror/session types, a repository for a value object or child entity, a generic Repository<T> base class, transaction demarcation inside a repository, DTOs returned from repository methods, raw SQL or ORM calls inside application services, or load-mutate-persist used for a plain counter. Owns: interface design (one per aggregate root, domain vocabulary, typed IDs), same-name implementations in <bc>/infrastructure, the persistence mirror, rehydrate/mapping translation, compare-and-set locking mapped to 409, atomic operations, and use-case-optimal queries returning purpose-built immutable Value Objects — not DTOs. Excludes aggregate design (ddd-aggregates) and transactions (ddd-services).
---

# DDD Persistence Skill

How domain objects cross into and out of storage, in any language. The repository is the boundary: its **interface** lives in `<bc>/domain` and speaks only domain language; its **implementation** lives in `<bc>/infrastructure` and owns every framework concern — the persistence mirror, the translation, the lock mechanics, the SQL.

**Foundational principle.** Everything a repository accepts and returns is a domain type: aggregates, typed IDs, and — for queries whose shape isn't an aggregate — purpose-built immutable Value Objects. Nothing persistence-flavored crosses the interface in either direction: no ORM entities, no generated row types, no sessions, no query builders, no DTOs. The interface describes what the aggregate needs from storage, in the ubiquitous language; the implementation supplies it however the framework works this year. When those two concerns share a type or a layer, every schema decision becomes a domain decision and every domain rename becomes a migration.

**Red Flags — STOP if you find yourself thinking:**

- About to place a repository interface in `<bc>/infrastructure` (it goes in `<bc>/domain`, per `ddd-foundations`).
- About to return an ORM entity, generated row type, or mirror class from a repository method.
- About to return a DTO from a repository method — the query result is domain-specific; DTOs are application-specific.
- About to write a repository for a value object or a child entity — repositories exist per aggregate root only.
- About to write a generic `Repository<T>` base class or interface.
- About to put transaction demarcation inside a repository (the application service owns it, per `ddd-services`).
- About to expose the session, the query builder, or a query-language string through the interface ("for flexibility").
- About to write raw SQL or ORM calls inside an application service instead of adding an intention-revealing repository method.
- About to implement a counter bump as load-mutate-persist — that's a lost-update hazard wearing the aggregate pattern.
- About to name the implementation after its technology — implementation details do not belong in names.

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing a new repository interface, or adding a method to one.
- Implementing a repository: the mirror type, the translation, the lock check.
- Adding a query whose natural result isn't an aggregate — a list row, a page, a stats summary.
- Adding an atomic in-database operation (counter increment, bulk status flip).
- Reviewing a diff where persistence types, sessions, or DTOs cross a repository interface, where a repository owns a transaction, or where an application service contains raw SQL/ORM calls.

**Out of scope**: where the interface and implementation live (`ddd-foundations` Rules 7–8 — assumed here); the aggregate's own shape, its `version` field, and the `rehydrate(...)` contract's *existence* (`ddd-aggregates` — this skill governs the *calling* side); transaction ownership and the load-compute-mutate-persist orchestration (`ddd-services`); event dispatch (`ddd-domain-events`); migration-tool and ORM mechanics (the per-stack persistence skill).

---

## Core Rules

1. **One repository per aggregate root — and only per aggregate root.** `OrderRepository`, `ThoughtRepository`. Value objects and child entities are loaded and saved *with* their aggregate; they never get their own repository.
2. **The interface lives in `<bc>/domain` and speaks the ubiquitous language.** `findById`, `findByOrderId`, `persist`, `remove` — domain verbs, typed-ID parameters (`OrderId`, never raw UUIDs in pairs), aggregate returns (present-or-absent, in the language's idiom). No ORM types, no session types, no query-language strings, no framework imports.
3. **The implementation lives in `<bc>/infrastructure` and shares the interface's name.** The module path is the distinguisher — implementation details (technology names) do not belong in names. Each language has a native spelling of same-name-different-module (see the reference files).
4. **The persistence mirror carries every framework concern.** The ORM entity, the mapped model, or the generated row types — mapping annotations, framework base classes, lock columns, column naming. It lives in `<bc>/infrastructure` (or is generated), and nothing outside the repository implementation touches it.
5. **Translation is the implementation's whole job.** Inbound: reconstruct via the aggregate's `rehydrate(...)` — the repository implementation is that method's *only* caller, and value-object constructor validation re-runs on the way in (per `ddd-value-objects` Rule 2). Outbound: map the aggregate to the mirror (`toRow`/`fromDomain`). The mapping functions live beside the implementation.
6. **`persist` saves the whole aggregate, atomically, upsert-style.** One aggregate per call; the aggregate's internal consistency is all-or-nothing. Cross-aggregate consistency is events, not multi-aggregate saves (per `ddd-aggregates` Rule 13).
7. **Optimistic locking is a compare-and-set against the loaded version.** Whether the framework provides it (a version column annotation) or the implementation does it by hand (`WHERE version = :loaded`, count check), a stale write surfaces as a **domain-neutral conflict error** — defined in domain or shared code, never a framework exception type — which the api layer maps to **409 Conflict**. The repository never swallows or retries the conflict; deciding what a conflict means belongs to the use case.
8. **Atomic in-database operations get intention-revealing interface methods.** A plain counter with no invariant (`thumbsUp`) bumped via load-mutate-persist is a lost-update hazard; expose it as `incrementThumbsUp(id)` on the interface and implement it as one atomic UPDATE. The method name carries the domain intent; the SQL stays behind the boundary. Reserve this for invariant-free state — anything the aggregate must guard goes through load-mutate-persist.
9. **Use-case-optimal queries return purpose-built immutable Value Objects — not DTOs** (Vernon, *Implementing Domain-Driven Design*). When a query's natural result isn't an aggregate — a listing row, a page, a stats summary — keep the query on the repository and return a Value Object designed for that use case, living in `<bc>/domain`, immutable, equality by value (per `ddd-value-objects`). It is explicitly *not* a DTO: DTOs are application-specific and live in `<bc>/application`; this query result is domain-specific. The application service may map the VO to a DTO for the api layer, as with any domain type.
10. **Pagination is a use-case-optimal query.** Page/size arrive as plain parameters; the result is a purpose-built VO carrying the items and the total count (`OrderPage(items, totalCount, page, size)`). The repository does not leak the framework's pagination object.
11. **No generic `Repository<T>`.** A shared base class flattens every aggregate to the same CRUD surface, invites methods no aggregate asked for, and turns the interface from a domain contract into a framework echo. Each repository earns its methods from its aggregate's use cases.
12. **No transactions inside repositories.** The application service owns the boundary (per `ddd-services` Rule 8); a repository method runs inside whatever transaction its caller opened.
13. **Schema evolves by versioned migrations.** The migration tool and workflow are per-stack; the requirement is family-wide.

For the worked example — interface, same-name implementation, mirror, locking, an atomic operation, and a Vernon use-case-optimal paged query — read the reference file matching the working language: `references/canonical-example-java.md`, `references/canonical-example-python.md`, or `references/canonical-example-typescript.md`. Load exactly one, and only when implementing a repository or adding a non-trivial query; the rules and tables in this file are sufficient for placement and review decisions. If the working language has no file, read the closest listed one and adapt the idioms — the persistence contract is identical in every language.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| Repository method returning the ORM entity / generated row type | Persistence types escape the boundary; every caller now depends on the schema, and the "mirror" stops mirroring — it *is* the model. | Return the aggregate (via `rehydrate`) or a purpose-built domain VO. The mirror never crosses the interface. |
| Repository method returning a DTO | DTOs are application-specific; the repository is a domain contract. The query result loses its domain meaning and the domain now depends on application types (dependency inversion, violated). | A purpose-built immutable Value Object in `<bc>/domain` (Rule 9). The application service maps VO → DTO if the api needs one. |
| Exposing the session, query builder, or a `query(sql)` escape hatch on the interface | The interface stops being a contract; every caller can now do anything, and the implementation can guarantee nothing. "Flexibility" here means "no boundary." | Add the intention-revealing method the use case actually needs. If a use case needs a new query, the interface grows one named method. |
| A repository for a value object or child entity (`OrderLineRepository`) | Value objects and child entities live and die with their aggregate; loading them independently breaks the aggregate's consistency boundary. | Load the aggregate. If you only need a slice, that's a use-case-optimal VO query on the *aggregate's* repository. |
| Generic `Repository<T>` base class/interface | Flattens every aggregate to the same CRUD surface; invites unused methods (`deleteAll` on `Payment`?); turns domain contracts into framework echoes. | One hand-written interface per aggregate root, earning its methods from real use cases. |
| Transaction demarcation inside the repository | The repository can't know the use case's boundary — maybe this call shares a transaction with an event-handler cleanup. | The application service owns the transaction (per `ddd-services`); the repository runs inside it. |
| Raw SQL / ORM calls inside an application service | Persistence mechanics leak into orchestration; the query is untestable without the database and invisible to the interface's contract. | Add a named method to the repository interface; implement it behind the boundary. |
| Load-mutate-persist for an invariant-free counter | Two concurrent thumbs-ups: both load 5, both persist 6 — a lost update. The aggregate pattern is protecting an invariant that doesn't exist here. | An atomic interface method (`incrementThumbsUp(id)`) implemented as one UPDATE (Rule 8). |
| Swallowing or silently retrying the optimistic-lock conflict inside the repository | Whether to retry, merge, or surface the conflict is a use-case decision; hiding it turns explicit contention into silent last-write-wins. | Surface a domain-neutral conflict error; the api layer maps it to 409; the use case decides anything smarter. |
| Naming the implementation after its technology | The name outlives the technology and leaks an implementation detail into every mention. | Same name as the interface; the module path is the distinguisher (Rule 3). |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "A generic `Repository<T>` base saves boilerplate — every repo needs findById and save anyway." | The boilerplate saved is three one-line signatures; the cost is a contract that no longer describes any aggregate. `Payment` gets `deleteAll()` because `T` demanded it; `Thought` gets `findAll()` unpaginated because the base had it. Interfaces earn methods from use cases, not from type parameters. When two repositories genuinely share implementation plumbing, share it *inside* the infrastructure layer — never in the domain contract. |
| "Returning the ORM entity avoids the mapping cost — it has all the same fields." | Same fields today. The moment they diverge (schema denormalization, soft-delete column, rename), every caller of the repository breaks with the schema. The mapping is a bounded, mechanical cost paid in one file; the leak is an unbounded coupling paid everywhere. And the ORM entity is mutable framework-managed state — handing it out gives every caller a live wire into the persistence context. |
| "Returning a DTO from the query is pragmatic — the data's going to the api anyway." | Where the data ends up doesn't change what layer the contract lives in. The repository is a domain interface; a DTO return makes domain depend on application. Vernon's rule resolves this exactly: a purpose-built immutable Value Object — domain-specific, validated, equality by value. The application service maps it to a DTO in one line if the api needs one. Same pragmatism, right direction. |
| "One repository for the whole schema is simpler — it's all one database." | One database is a deployment fact; one repository is a domain claim — that the whole schema is one aggregate. It isn't. Per-aggregate repositories keep each contract small and let the module boundaries (and architecture tests) mean something. |
| "The counter increment is two lines of SQL — I'll just put it in the application service." | Two lines of SQL in the service is persistence leaking through the layer boundary: untestable without a database, invisible to the interface, and unfindable by the next reader of the contract. `incrementThumbsUp(id)` on the interface is the same two lines, behind the boundary, with a domain name. |
| "The repository should retry on lock conflict — callers shouldn't have to care." | Whether a conflict is retryable is a *use-case* property, not a storage property. Retrying a thumbs-up is harmless; retrying a refund double-charges. The repository surfaces the conflict; the layer that knows the stakes decides. |
| "Exposing the query builder gives callers flexibility without endless interface methods." | "Endless" is hypothetical; the leak is immediate. Every capability the builder exposes becomes part of the de facto contract, and the implementation can never again change storage strategy without auditing every call site. Interfaces grow one named method per real use case — that growth rate is a feature. |

---

## Quick Reference

### Query decision table

| The use case needs | Repository answer |
| --- | --- |
| One aggregate by identity | `findById(OrderId) → Order?` — rehydrated aggregate |
| To change an aggregate's state | Load-mutate-persist through the aggregate's methods (orchestrated per `ddd-services`) |
| To bump invariant-free state (a counter) | Atomic intention-revealing method: `incrementThumbsUp(ThoughtId)` — one UPDATE behind the boundary |
| A listing/page whose rows aren't full aggregates | Use-case-optimal query returning a purpose-built VO page: `findRecent(page, size) → OrderPage` |
| A summary/stats shape | Use-case-optimal query returning a purpose-built VO: `ratingStats() → RatingStats` |
| Data from *another* BC's aggregate | Not this repository's job — the application service composes across contexts (per `ddd-foundations`) |

### Repository interface checklist

When you commit a repository interface, every line below should be true:

- [ ] One interface, one aggregate root; no repositories for VOs or child entities.
- [ ] Lives in `<bc>/domain`; zero framework imports.
- [ ] Method names in the ubiquitous language; typed-ID parameters.
- [ ] Returns aggregates, present-or-absent idioms, or purpose-built immutable domain VOs — never mirror types, sessions, query builders, or DTOs.
- [ ] Paginated queries return a purpose-built page VO (items + total), not a framework pagination object.
- [ ] Atomic operations are intention-revealing methods, not leaked SQL.
- [ ] No transaction demarcation.
- [ ] The implementation shares the interface's name, in `<bc>/infrastructure`; no technology names.
- [ ] Stale-write conflicts surface as a domain-neutral conflict error (api maps to 409).

### Companion skills

- **`ddd-foundations`** — Rules 7–8: interface in `<bc>/domain`, implementation in `<bc>/infrastructure`; the layering this skill assumes.
- **`ddd-aggregates`** — the `rehydrate(...)` contract this skill's implementations call, and the plain `version` field the compare-and-set checks.
- **`ddd-value-objects`** — the shape rules for typed IDs and for the purpose-built query VOs of Rule 9 (immutable, constructor-validated, equality by value).
- **`ddd-services`** — the application service that owns the transaction the repository runs inside, and the load-mutate-persist orchestration.
- **`ddd-domain-events`** — the after-commit delivery that cross-aggregate consistency relies on instead of multi-aggregate saves.
- **Per-stack mechanics skills** (`quarkus-*`, `fastapi-*`, `express-*`, or your stack's equivalents) — migration tooling, session/connection management, and ORM specifics.
