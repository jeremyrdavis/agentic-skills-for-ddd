---
name: ddd-domain-events
description: Use when designing a domain event — naming it, deciding what its payload carries, choosing between a domain event and an integration event, or deciding how and when it is delivered to handlers. Trigger on an event named in the present tense or imperative (CreateOrder, OrderCreate, PlaceOrderEvent), an event whose payload is the whole aggregate or a mirror/DTO type, an event with no identity or timestamp, an event publisher, broker client, Emitter, or Event<> injected into a <bc>/domain type, a handler that mutates a second aggregate inside the publisher's transaction, publishing that happens before commit or "right after the call returns" with no mechanism tying it to the commit, a handler with no idempotency guard, or a domain event type serialized straight onto a message topic. Excludes raising the event inside the aggregate (ddd-aggregates), the application service's drain-and-hand-off (ddd-services), and the delivery mechanism of a specific framework — observer phases, outbox tables, broker configuration (<stack>-ddd).
---

# DDD Domain Events Skill

What a domain event is, what it carries, and when it reaches a handler — in any language. An event is a **fact**: something that already happened to one aggregate, named in the past tense, carrying the identifiers and changed values another part of the system needs to react. It is raised inside the aggregate, drained by the application service, and delivered **after the transaction that produced it commits** — by a mechanism, never by call ordering.

**Foundational principle.** An event records a fact; it is not a message to a broker, not a command to another aggregate, and not a copy of the aggregate's state. Three things follow. The payload is small and domain-typed (typed IDs plus the facts that changed), because anything more couples every consumer to the aggregate's shape. Delivery is after commit, because a handler that sees an uncommitted fact acts on something that may never have happened. And what crosses a bounded-context boundary is a *separate, versioned* integration event, because the domain event is free to change with the model and a published contract is not.

**Red Flags — STOP if you find yourself thinking:**

- About to name an event `CreateOrderEvent`, `OrderCreate`, or `PlaceOrder` — an event is a past-tense fact: `OrderPlacedEvent`.
- About to put the aggregate (or its mirror, or a DTO) in the event payload "so handlers have everything."
- About to inject an event publisher, message emitter, or broker client into a type under `<bc>/domain`.
- About to publish inside the transaction and let handlers run synchronously — or publish "right after commit returns" with nothing tying the two together.
- About to update a second aggregate inside the handler's caller's transaction "so it's consistent."
- About to serialize the domain event type itself onto a topic another service reads.
- About to write a handler that assumes it will see each event exactly once.
- "The event needs a return value so the caller knows the handler succeeded."
- "It's one service — the outbox is over-engineering; we'll publish after the save."

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing a new domain event: its name, payload, identity, and timestamp.
- Deciding whether a fact stays inside the bounded context (domain event) or is published to other contexts (integration event).
- Designing a handler: its transaction, its idempotency, what it is allowed to call.
- Choosing the delivery guarantee a use case needs (in-process after-commit vs outbox).
- Reviewing a diff with a present-tense event name, an aggregate in a payload, broker types in `domain` or `application`, a handler that joins the publisher's transaction, or publishing that is not tied to the commit.

**Out of scope**: raising the event inside the aggregate's state-changing method and the `pendingEvents` / `pullPendingEvents()` shape (`ddd-aggregates` Rule 12); the application service's drain-and-hand-off and its ownership of the transaction (`ddd-services` Rule 11); the layering that puts the publisher abstraction in `application` and the broker adapter in `infrastructure` (`ddd-foundations`); the value-object shape of the payload (`ddd-value-objects`); the mechanism that guarantees after-commit delivery in your framework — observer phases, post-commit hooks, outbox tables, broker configuration (use the `<stack>-ddd` skill for your stack, e.g. `quarkus-ddd`; if none is installed, apply these rules with your framework's equivalents).

---

## Core Rules

1. **An event is a past-tense fact with the `Event` suffix.** `OrderPlacedEvent`, `PaymentAuthorizedEvent`, `OrderCancelledEvent`. Never imperative or present tense — `CreateOrder` is a command, `OrderCreate` is nothing. The name says what happened, not what should happen next.
2. **The payload is typed IDs plus the facts that changed.** `OrderCancelledEvent(OrderId orderId, CustomerId customerId, Money refundDue, CancellationReason reason)`. Not the aggregate, not the mirror, not a DTO. If a handler needs more, it loads the aggregate through its own repository in its own transaction.
3. **Every event carries its own identity and time.** An `eventId` and an `occurredAt`, assigned when the aggregate raises it. The id is the idempotency key for every handler and the outbox key for every relay; the timestamp is the order of record when delivery reorders.
4. **The payload is a value object.** Immutable, constructed through the language's native value construct, validated in construction, equality by value (`ddd-value-objects` Rules 1–5). It lives in `<bc>/domain`.
5. **Raised inside the aggregate, drained by the application service.** The aggregate appends the event in the method that changes state (`ddd-aggregates` Rule 12); the application service drains `pullPendingEvents()` and hands the events to the publisher (`ddd-services` Rule 11). No service constructs an event; no aggregate publishes one.
6. **Delivered after commit, by mechanism.** Handlers see an event only once the transaction that produced it has committed, and the guarantee comes from the framework's transactional-observer phase, a post-commit hook, or an outbox — never from the application service calling `publish()` after `commit()` returns. Publishing before commit leaks facts that may be rolled back; publishing after, with no coupling, is a dual write that loses events on the first crash between the two.
7. **One handler, one use case, one transaction of its own.** A handler is an inbound adapter that calls an application service; that service owns its transaction (`ddd-services` Rule 8). A handler never joins the publisher's transaction, and never mutates a second aggregate inside it — cross-aggregate consistency is eventual (`ddd-aggregates` Rule 13).
8. **Handlers are idempotent.** Delivery is at-least-once in every mechanism worth using. Dedupe on `eventId`, or make the operation naturally idempotent. A handler that double-charges on redelivery is wrong, not unlucky.
9. **Domain events stay inside the bounded context; integration events cross it.** What another context consumes is a separate, explicitly versioned contract (`OrderCancelledV1`) with primitive-friendly fields, mapped from the domain event in `<bc>/infrastructure`. The domain event changes with the model; the integration event changes by version. The domain event type is never the wire format.
10. **No broker in `domain` or `application`.** The application service depends on a publisher abstraction (the framework's in-process event bus or a domain-defined interface); the broker client lives in one `<bc>/infrastructure` adapter. `Emitter`, producer, or channel types in `domain` or `application` are a layering violation (`ddd-foundations` Rule 6).
11. **Events return nothing.** Fire-and-forget. If the use case needs the handler's result to proceed, it isn't an event — it is a domain-service call or a second step of the same use case.
12. **Events are not for coordination inside one aggregate.** A state change that must happen together with another state change in the *same* aggregate is a method call, enforced as an invariant (`ddd-aggregates` Rule 7). Events cross aggregate boundaries; they never substitute for them.

For the worked example — the drain, an after-commit in-process handler, a transactional outbox with a relay to a broker, and a versioned integration event — read `references/domain-events.md` in the `<stack>-ddd` skill for your stack (e.g. `quarkus-ddd`). Load it only when wiring delivery; the rules and tables in this file are sufficient for naming, payload, and review decisions. If no `<stack>-ddd` skill is installed, apply the rules with your framework's equivalents — the event contract is identical in every stack.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| `CreateOrderEvent`, `OrderCreate`, `PlaceOrderEvent` | Present tense or imperative names describe intent, not fact; handlers can't tell whether the thing happened. | Past tense, `Event` suffix: `OrderPlacedEvent`. |
| The aggregate (or its mirror, or a DTO) as the payload | Every handler is now coupled to the aggregate's full shape; the event becomes a change-data-capture feed; mirrors and DTOs drag other layers into `domain`. | Typed IDs plus the facts that changed (Rule 2). Handlers that need more load it themselves. |
| No `eventId` / `occurredAt` | Handlers cannot dedupe; relays cannot key the outbox; reordered delivery has no order of record. | Assign both in the aggregate when the event is raised (Rule 3). |
| `publish()` called before commit, handlers run synchronously | Handlers act on facts that may roll back; a slow handler holds the publisher's transaction open. | After-commit delivery by mechanism (Rule 6). |
| `save(); commit(); publish();` with nothing tying them | Dual write: a crash after commit and before publish loses the event silently. | Transactional observer phase or outbox (Rule 6); the `<stack>-ddd` skill has the mechanism. |
| Handler mutates a second aggregate inside the publisher's transaction | Two aggregates, one transaction — the consistency boundary `ddd-aggregates` Rule 13 exists to prevent. | The handler calls an application service with its own transaction (Rule 7). |
| Handler assumes exactly-once | Every real mechanism redelivers on retry; the second delivery double-applies. | Dedupe on `eventId` or make the operation idempotent (Rule 8). |
| Domain event type serialized straight onto a topic | The wire contract now changes whenever the model does; every consumer breaks on a domain rename. | A versioned integration event mapped in `infrastructure` (Rule 9). |
| `Emitter` / producer / channel injected into an aggregate or application service | Broker types in `domain` or `application`; the layering map breaks. | Publisher abstraction in `application`; broker adapter in `infrastructure` (Rule 10). |
| An event whose handler must "return" a verdict to the raiser | Request/response wearing an event costume; the raiser blocks on something events don't promise. | A domain-service call, or the next step of the same use case (Rule 11). |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "We'll publish right after the save returns — the transaction has committed by then, so it's fine." | It is fine until the process dies between the two lines, and then an order is cancelled with no refund ever triggered and no trace that anything was lost. The window is small; the failure is silent and permanent. A transactional observer phase costs one annotation; an outbox costs one table. Both are cheaper than the incident, and both are what Rule 6 means by "mechanism". |
| "Putting the aggregate in the event saves every handler a repository call." | It also hands every handler the aggregate's entire shape, so a field rename in `Order` breaks inventory, billing, and notifications at once — the opposite of the decoupling events exist for. Handlers that need more than the facts load the aggregate in *their* transaction, seeing *committed* state, which is the only state they should act on anyway. |
| "The handler should run in the same transaction so the two aggregates are always consistent." | Then you have one aggregate in two files. The one-aggregate-per-transaction rule (`ddd-aggregates` Rule 13) is the definition of the consistency boundary; a handler that joins the publisher's transaction erases it, and a slow or failing handler now rolls back the original use case. Eventual consistency is not a compromise here — it is the model. |
| "Idempotency is the broker's job — we'll configure exactly-once." | No delivery mechanism gives a *handler* exactly-once semantics end to end; retries, redeliveries, and replays exist in every system that recovers from failure. An `eventId` check is two lines; without it the handler is correct only on the happy path. |
| "One event type for inside and outside the service — two types is duplication." | The two types have different change rates and different owners. The domain event follows the ubiquitous language and can be renamed tomorrow; the integration event is a contract other teams deployed against and changes by version. One type means either the model is frozen by consumers or consumers break on every refactor. The mapping is a few lines in `infrastructure`. |
| "The event needs a return value so the caller knows whether the handler accepted it." | If the raiser needs the answer, the raiser is making a request — model it as a domain-service call or as the next step in the use case, with a result type. Events announce facts; they don't negotiate. |

---

## Quick Reference

### Event vs command vs integration event

| You have | It is | Lives in | Named |
| --- | --- | --- | --- |
| A fact about one aggregate that others inside the BC react to | Domain event | `<bc>/domain` | Past tense, `*Event` |
| An instruction to change state | Command (a DTO) | `<bc>/application` | Imperative, `*Command` |
| A fact other bounded contexts consume | Integration event | `<bc>/infrastructure` (contract package) | Past tense, versioned (`*V1`) |
| A request that needs an answer before the use case continues | Not an event — a domain-service call or the next step | `<bc>/domain` / `<bc>/application` | — |

### Domain-event commit checklist

- [ ] Past tense, `Event` suffix; the name is a fact.
- [ ] Payload: typed IDs + changed facts as value objects; no aggregate, mirror, or DTO.
- [ ] `eventId` and `occurredAt` assigned when raised.
- [ ] Raised inside the aggregate's state-changing method; drained by the application service.
- [ ] Delivered after commit by a mechanism (`<stack>-ddd`), not by call order.
- [ ] Each handler calls an application service with its own transaction; no second aggregate in the publisher's transaction.
- [ ] Handlers dedupe on `eventId`.
- [ ] Anything leaving the BC is a versioned integration event mapped in `infrastructure`.
- [ ] No broker types in `domain` or `application`.

### Companion skills

- **`ddd-foundations`** — domain events live in `<bc>/domain`; publisher abstraction in `application`, broker adapter in `infrastructure`.
- **`ddd-aggregates`** — Rule 12: the aggregate raises events in its state-changing methods; Rule 13: one aggregate per transaction, which is why handlers get their own.
- **`ddd-value-objects`** — the payload's shape: immutable, constructor-validated, equality by value.
- **`ddd-services`** — Rule 11: the application service drains `pullPendingEvents()` and hands off; handlers are application services too.
- **`ddd-persistence`** — the repository a handler uses to load the second aggregate in its own transaction.
- **`<stack>-ddd`** — the mechanics skill for your stack (`quarkus-ddd`, `spring-ddd`, `fastapi-ddd`, ...), if installed: the transactional-observer phase or outbox that guarantees after-commit delivery, the broker adapter and its configuration, and the worked example (`references/domain-events.md` there). If none is installed, apply the rules above with your framework's equivalents.
