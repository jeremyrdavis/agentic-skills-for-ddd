---
name: ddd-value-objects
description: Use when designing or modifying a value object — modeling Money, EmailAddress, Address, typed ID wrappers (OrderId, CustomerId), or any type with no identity and equality by value. Trigger on raw amount + currency field pairs on a domain type, raw string email/sku/phone fields, mutable classes with setters used as value objects, *Validator companions for value-shaped types, mutating methods on a type that should be immutable, hand-written equality where the language generates it (or missing equality where it doesn't), an identity field on a value object, or ORM imports/mapping annotations on a value type in <bc>/domain. The named failure mode is primitive obsession. Excludes layering (ddd-foundations), aggregate-root design — identity, factories, state machines (ddd-aggregates), the persistence-layer mapping that turns a value object into columns or a mirror type (ddd-repositories), and event naming/dispatch (ddd-domain-events). This skill is about the shape of the value object, not its persistence wiring.
---

# DDD Value Objects Skill

How to model value objects, in any language. A value object has no identity, is immutable, equals by value, and encodes its invariants in its construction. Each language has a native construct for this shape — use it (see the per-language reference files).

**Foundational principle.** A value object encodes its invariants in its constructor. If validation lives anywhere except the type's construction path, the type is just a container for primitives wearing a class label. **Value objects are plain domain types — no ORM imports, no mapping annotations, no identity field.** Domain depends on nothing in your code (per `ddd-foundations`); the persistence layer maps value objects to columns or to a mirror type in `<bc>/infrastructure` (covered by `ddd-repositories`). The named failure mode this skill prevents is **primitive obsession** — modeling a domain concept (money, email, postal address, order ID) as raw decimal/string/integer fields scattered across the aggregate.

**Red Flags — STOP if you find yourself thinking:**

- About to declare an amount field and a currency field as **separate** fields on a domain type — that's a `Money` value object trying to be born.
- About to use a raw string for email, SKU, phone number, or address line on a domain type — that's `EmailAddress`, `Sku`, `PhoneNumber`, or `Address` trying to be born.
- About to write a mutable class with setters and call it a "value object."
- About to hand-write equality/hash/string methods in a language whose value construct generates them — or about to *skip* equality in a language that doesn't generate it and compare values by identity.
- About to add a `*Validator` companion class for a value-shaped type.
- About to add a method that mutates the value's fields in place.
- About to reach for a codegen/annotation library to build a value type the language's native construct already provides.
- About to add an identity field to a value object — identity makes it an entity.
- About to import an ORM or add a mapping annotation on a value type in `<bc>/domain`.
- "I'll put the mapping annotation on the domain type directly — saves a mapper."
- "The ORM needs a no-arg constructor and setters, so immutable values are awkward."
- "A raw decimal and string are simpler — a wrapper is over-engineering."

If any of these surface, re-read Core Rules and Excuse / Reality before typing.

---

## When to Use

- Designing a new value object: `Money`, `EmailAddress`, `Address`, `PhoneNumber`, `Coordinates`.
- Designing a typed ID wrapper used at API surfaces: `OrderId`, `CustomerId`, `Sku`.
- Adding behavior to a value object (`Money.plus`, `Money.times`, `EmailAddress.domain()`).
- Reviewing a diff with raw primitives where a value object would cohere (amount + currency as two fields), a mutable "value object," missing or hand-rolled equality on a value-shaped type, a `*Validator` companion, mutating methods on a type that should be immutable, or ORM imports/mapping annotations on a value type in `<bc>/domain`.

**Out of scope**: layering and module placement (use `ddd-foundations`); aggregate-root design — identity, factory methods, state machines (use `ddd-aggregates`); the persistence-layer mapping that turns a value object into columns or a mirror type in `<bc>/infrastructure` (use `ddd-repositories`); domain-event naming and dispatch (use `ddd-domain-events` — events are built with the same value-construct shape, and this skill's construction-validation rule applies to their payloads, but their conventions live there); domain services (use `ddd-services`).

---

## Core Rules

1. **Use the language's native immutable value construct.** Java: `record`. Python: `@dataclass(frozen=True)`. TypeScript: class with `readonly` fields. It gives you immutability and a compact declaration without boilerplate. Don't hand-roll a mutable class for a value object unless you have a measured reason the native construct can't satisfy.
2. **Validate in the constructor — the single construction path.** The validation runs every time the value is constructed, including when the repository implementation rebuilds it from stored state. There is no "valid path" and "invalid path"; there is one path. (Relationship to `ddd-aggregates` Rule 6: aggregates' `rehydrate` skips creation *side effects* like raising events; value objects have no side effects, so their constructor validation always runs — the two rules are consistent, not in conflict.)
3. **Normalize during construction.** Trim, lowercase, scale decimals — before the value is stored in its fields. Each language has an idiom for normalizing inside an immutable construct (see the reference files); use it rather than normalizing at call sites.
4. **Equality is by value.** Use the language's generated value equality where one exists (Java records, Python frozen dataclasses) — and don't override it. Where the language provides none (TypeScript classes), define one canonical `equals()` on the type — and never compare value objects by identity. If two value objects with the same components must compare *unequal*, you have an entity (it has identity); use `ddd-aggregates` instead.
5. **Value objects are immutable.** No setters. No methods that change the receiver. Operations that "modify" the value return a *new* value: `Money.plus(other)` returns a new `Money`, never mutates.
6. **The value object is a plain domain type.** No ORM imports, no mapping annotations, no identity field. The persistence layer maps it to columns or to a mirror type in `<bc>/infrastructure`; that responsibility belongs to the repository implementation (covered by `ddd-repositories`), not to the value object.
7. **Never put an identity field on a value object.** Identity makes it an entity, not a value — different equality semantics, different lifecycle. If you want to identify a value across rows, you don't have a value object; apply `ddd-aggregates`.
8. **Behavior lives on the value object, not in a service.** `Money.plus`, `Money.times`, `EmailAddress.domain()` — operations on a value belong to the value's own type. A `MoneyService.add(amount1, amount2, currency1, currency2)` taking primitives is the smell that the type should have been born.
9. **Typed ID wrappers are value objects — and must be nominally distinct.** `OrderId` wrapping a UUID follows the same rules: native construct, constructor validation, equality by value. The wrapper's entire purpose is making `placeOrder(OrderId, CustomerId)` impossible to call with arguments swapped — so in structurally-typed languages, use a construct the type system treats as distinct (see the TypeScript reference), or the protection is illusory.
10. **Prefer the native construct over codegen and annotation libraries.** If the language's own value construct provides immutability and equality, a code-generation library (Lombok-style annotations, and similar) adds a build dependency to reproduce a language feature — and its "data class" variants often generate setters, which is the opposite of what a value needs. Per-language specifics in the reference files.
11. **The named failure mode is *primitive obsession*.** When you find yourself declaring an amount and a currency as two fields, or a raw string email, or threading (decimal, string) pairs through signatures — you have an unborn value object. The fix is always the same: introduce the type.

For worked examples — `Money`, `EmailAddress`, and typed ID wrappers with idiomatic construction validation, normalization, equality, and the persistence-mirror sketch — read the reference file matching the working language: `references/canonical-example-java.md`, `references/canonical-example-python.md`, or `references/canonical-example-typescript.md`. Load exactly one, and only when creating a new value object or converting primitives into one; the rules and tables in this file are sufficient for placement and review decisions. If the working language has no file, read the closest listed one and adapt the idioms — the value-object contract is identical in every language.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| An amount field and a currency field as two separate fields on the aggregate | Primitive obsession. The two fields are conceptually one value (`Money`), and the type system has no way to enforce that they're consistent or that the amount is non-negative. | A `Money` value type in `<bc>/domain` with validation in the constructor; the aggregate holds a single `total: Money` field. |
| A raw string email field on `Customer` | Same shape. Every code path touching the email re-implements (or forgets) the format check, the trim, the lowercasing. | An `EmailAddress` value type with normalization + format check in the constructor. |
| A mutable class with setters and hand-rolled equality used as a "value object" | The native value construct exists for exactly this. The mutable form lets a setter call elsewhere break the value; a manual equality method drifts from the field set on the next merge. | Convert to the language's value construct. Delete the hand-written equality where the language generates it; keep one canonical `equals()` where it doesn't. |
| ORM mapping annotations (or any ORM import) on a value type in `<bc>/domain` | Domain depends on the ORM; the value's design is now constrained by what the framework can map. | Drop the annotation. The persistence mirror in `<bc>/infrastructure` carries the mapping (embedded type or flat columns) — covered by `ddd-repositories`. |
| `MoneyService.add(amount1, amount2, currency1, currency2)` | The method takes primitives because the value type doesn't exist. The "service" is a container for behavior the value should own. | Define `Money`; move the operation onto it as `Money.plus(other)`. Delete the service. |
| `EmailValidator.validate(email)` called before constructing the owner | Validation lives outside the type. Anyone forgetting the validator gets an invalid value past the boundary. | Move validation into `EmailAddress`'s constructor. "Caller forgets to call" becomes "the type cannot be constructed invalid." |
| A method that mutates the value's fields in place | Breaks the value contract: two equal-but-aliased instances become unequal after one mutates, and consumers that cached a hash get a stale bucket. | Operations return new instances: `plus(other)` constructs and returns a new `Money`. |
| A codegen/annotation library to produce the value type | Adds a build dependency to reproduce a language feature; the "data" variants generate setters, defeating immutability. | Use the native construct. (Per-language specifics in the reference files.) |
| An identity field on a value object | Identity makes it an entity — different equality semantics, different lifecycle. | If the type genuinely has identity, it's an entity/aggregate — apply `ddd-aggregates`. If not, drop the identity field. |
| Raw UUID/string parameters in pairs (`placeOrder(id, customerId)` both raw) | Two same-typed arguments in a row is a swap waiting to happen; the compiler can't tell them apart. | Typed wrappers, nominally distinct: `placeOrder(OrderId, CustomerId)` makes the swap a compile error. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in a PR comment. The right column is what defeats it.

| Excuse | Reality |
| --- | --- |
| "A raw decimal and string are simpler — a wrapper is over-engineering. The validation can live in a service." | This is *primitive obsession*, named exactly. Validation that lives somewhere other than the type is validation that will be bypassed — every caller that forgets the validator gets a negative amount past the boundary. The wrapper is not over-engineering; it's the only way the type system can enforce what the comment says. Simplicity at the field declaration becomes complexity-by-vigilance everywhere else. |
| "The ORM needs a no-arg constructor and setters, so immutable values are awkward / impossible." | The domain value doesn't talk to the ORM at all. The persistence mirror in `<bc>/infrastructure` handles whatever shape the framework requires; the domain value is constructed by the repository's translation code, not by the ORM. The framework's constraints are the mirror's problem, not the value's. |
| "I'll put the mapping annotation on the domain type directly — saves a mapper." | The "saved" mapper is a few lines in the repository implementation; the cost of saving it is a domain type that imports the ORM. Once that import exists, framework upgrades, mapping-strategy changes, and column-naming policies cascade through the model. The mapper is bounded; the coupling is not. |
| "A codegen library removes the boilerplate without using the value construct." | The native construct already removes the boilerplate — without an annotation processor, plugin, or build dependency. The codegen "data" variants are actively wrong for values: they generate setters and equality shapes that drift from intent. Use the language feature. |
| "I'll write the value as a plain class so I can subclass it later if requirements change." | Value objects are not subclassed. Inheritance breaks value equality (a subclass instance that equals a base instance with the same components creates asymmetric or inconsistent comparisons). If requirements change, add a new value object or new methods on the existing one — don't subclass. |
| "Value constructs are just data carriers — they can't hold validation or behavior." | False in every language covered here. The construction path is the standard place for validation and normalization, and value types can carry static factories, instance methods, and derived accessors. The "just data" framing is a misreading of the construct. |
| "Adding a typed `OrderId` wrapper around a UUID is over-engineering — the UUID already prevents collision." | The wrapper isn't about collision (UUIDs handle that). It's about the *position* of the argument at the call site. Two raw UUIDs in a row accept swapped arguments at no compile-time cost; `placeOrder(OrderId, CustomerId)` makes the swap a compile error. The cost is a three-line type; the benefit is permanent. |

---

## Quick Reference

### Value-object decision tree

```
Has identity (two instances with the same fields are still distinct)?
├── Yes → it's an entity / aggregate root → use ddd-aggregates
└── No  → it's a value object → use the rules in this skill
```

### Value-object commit checklist

When you commit a new value object, every line below should be true:

- [ ] Declared with the language's native immutable value construct (record / frozen dataclass / readonly-field class).
- [ ] Validation in the constructor — the single construction path.
- [ ] Normalization (trim, lowercase, scale) happens during construction, via the language's idiom.
- [ ] No setters. No mutating methods. Operations return new instances.
- [ ] Equality by value: generated equality untouched where the language provides it; one canonical `equals()` where it doesn't; never identity comparison.
- [ ] No codegen/annotation library standing in for the native construct.
- [ ] No identity field. (If it has identity, it's not a value object.)
- [ ] No ORM imports or mapping annotations. The persistence mapping (flat columns or a mirror type in `<bc>/infrastructure`) is the repository's responsibility.
- [ ] If a typed ID wrapper: still a value object; constructor validates the wrapped value; nominally distinct from other ID wrappers.
- [ ] Behavior (arithmetic, derivation) lives as methods on the value type itself.

### Value object vs entity

| You see | It's a |
| --- | --- |
| Two instances with the same data should compare equal | Value object |
| Two instances with the same data should still be distinguishable | Entity (use `ddd-aggregates`) |
| You want to track changes over time / a state machine on the type | Entity |
| The type is a "frozen" measurement, descriptor, or label | Value object |
| The type has an identity field | Entity (by definition; remove the field if it shouldn't be one) |
| The type is referenced by other types' fields but never identified independently | Value object |

### Companion skills

- **`ddd-foundations`** — the layering map; value objects live in `<bc>/domain` (or a shared domain module if used across BCs).
- **`ddd-aggregates`** — aggregates that own and embed value objects, and the entity side of the decision tree.
- **`ddd-domain-events`** — events use the same value-construct shape; this skill's construction-validation rule applies to event payloads, and their naming (`*Event`, past tense) and dispatch live there.
- **`ddd-repositories`** — the persistence-layer mapping that turns a value object into columns or a mirror type in `<bc>/infrastructure`.
- **Per-stack mechanics skills** (`quarkus-*`, `fastapi-*`, `express-*`, or your stack's equivalents) — the ORM mechanics of the persistence-layer types that *do* carry mapping concerns. Those live in `<bc>/infrastructure`, not on the domain value.
