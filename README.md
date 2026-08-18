# Agentic Skills for DDD

A curated set of domain-driven design (DDD) skills for working with AI coding assistants. This repository packages a small, opinionated playbook for modeling DDD systems with clear boundaries, explicit language, and consistent layering.

Each skill is written to help an assistant reason about a specific DDD concern without collapsing everything into generic CRUD, service blobs, or framework-first design. The goal is to make Claude and Codex better at staying inside the right layer, choosing the right abstractions, and avoiding the common traps that turn domain code into persistence-driven code.

## What this repository contains

- `ddd-foundations-skill` — the shared layering model and terminology used by all other skills.
- `ddd-aggregates` — how to design aggregate roots, factories, rehydration, state transitions, and invariants.
- `ddd-value-objects` — how to model immutable value objects, typed IDs, and value-based equality.
- `ddd-services` — how to distinguish application services from domain services.
- `ddd-persistence` — how repositories, persistence mirrors, locking, and query results should work.

## Purpose

These skills are meant to help you and your AI assistant:

- model DDD concepts in a consistent way across a codebase,
- keep domain logic in the domain layer,
- keep infrastructure and framework concerns out of domain types,
- choose the right abstraction for each responsibility,
- and review code changes against DDD-specific rules and anti-patterns.

If you are building or reviewing a bounded context, these skills give you a shared vocabulary and a set of enforceable rules for:

- module layout,
- aggregate design,
- value object construction,
- service responsibilities,
- repository boundaries,
- and persistence translation.

## The individual skills

### `ddd-foundations-skill`
The foundation for the whole set. It defines the canonical DDD layers and where each kind of type belongs:

- `domain`
- `application`
- `infrastructure`
- `api`

Use this skill when you are deciding where a type should live, how modules should depend on one another, or whether a class is a domain service, application service, repository interface, controller, or adapter.

### `ddd-aggregates`
Use this skill when designing or modifying an aggregate root.

It covers:

- aggregate identity strategy,
- named factory methods,
- `rehydrate(...)` for persistence reconstruction,
- enforcing invariants during creation and transition,
- referencing other aggregates by ID rather than object reference,
- optimistic locking with a version field,
- and domain events raised from inside the aggregate.

### `ddd-value-objects`
Use this skill when modeling values with no identity.

It covers:

- immutable value objects,
- constructor-based validation,
- normalization during construction,
- equality by value,
- typed ID wrappers like `OrderId` and `CustomerId`,
- and behavior that lives directly on the value type.

### `ddd-services`
Use this skill when deciding whether something belongs in an application service, a domain service, or on the aggregate itself.

It covers:

- application service orchestration,
- domain service computation,
- transaction ownership,
- event draining and dispatch after commit,
- and naming conventions such as `*ApplicationService` and `*DomainService`.

### `ddd-persistence`
Use this skill when designing repositories and persistence behavior.

It covers:

- repository interfaces in the domain layer,
- repository implementations in infrastructure,
- translating between domain objects and persistence mirrors,
- optimistic locking and conflict handling,
- intention-revealing atomic database operations,
- and use-case-optimal query results that return domain value objects rather than DTOs or ORM entities.

## How the skills fit together

A useful way to think about the set is:

- `ddd-foundations-skill` tells you **where things belong**.
- `ddd-value-objects` tells you **how to model values**.
- `ddd-aggregates` tells you **how to model consistency boundaries**.
- `ddd-services` tells you **how to orchestrate use cases and domain computation**.
- `ddd-persistence` tells you **how to cross the storage boundary safely**.

In practice, you often use them together. For example:

- A new `Order` aggregate uses `ddd-aggregates`.
- Its `OrderId` and `Money` types use `ddd-value-objects`.
- The use case that creates or updates it uses `ddd-services`.
- The repository that saves it uses `ddd-persistence`.
- The module structure that keeps all of those in the right place comes from `ddd-foundations-skill`.

## How to leverage these skills with Claude

Claude works best when you give it the skill content explicitly and ask it to apply the rules while generating or reviewing code.

Recommended approach:

1. Open the relevant `SKILL.md` file for the task.
2. Paste the skill content into Claude or reference the skill directly if your Claude environment supports skills.
3. Ask Claude to follow the skill’s rules while designing, reviewing, or refactoring code.
4. If the task touches multiple concerns, combine the relevant skills.

Example prompts:

- “Use the DDD Foundations and Aggregates skills to design the module layout for this bounded context.”
- “Apply the Value Objects skill to model `EmailAddress` and `Money`.”
- “Use the Services and Persistence skills to refactor this use case so the transaction stays in the application service and the repository only translates data.”
- “Review this diff against the DDD skills and flag any boundary violations.”

Best practice: ask Claude to reason in terms of the skill’s checklist and anti-patterns, not just to generate code. That keeps the output aligned with the intended architecture.

## How to leverage these skills with Codex

Codex can use the same skill files as architectural guidance while it generates or modifies code.

Recommended approach:

1. Provide Codex with the relevant skill file or point it at this repository.
2. State the task in terms of the DDD concern you want to change.
3. Ask Codex to consult the skill before making changes.
4. For larger tasks, ask it to use multiple skills in sequence.

Example prompts:

- “Use `ddd-foundations-skill` and `ddd-services` to place these classes in the right modules.”
- “Use `ddd-aggregates` and `ddd-value-objects` to model this order workflow correctly.”
- “Use `ddd-persistence` to implement repository translation without leaking ORM entities outside infrastructure.”
- “Refactor this code to obey the DDD skills, then summarize the boundary changes you made.”

If you are using Codex in an agentic workflow, it helps to ask for a plan first and then request implementation. That makes it easier to validate that the right skill is being applied to the right layer before code is written.

## Practical workflow

A simple way to use the repository is:

1. Identify the DDD concern in your task.
2. Open the matching skill file.
3. Share that skill with Claude or Codex.
4. Ask for a design, review, or refactor aligned with the skill.
5. Validate the result against the skill’s quick-reference checklist.

## Repository structure

```text
.
├── ddd-aggregates/
├── ddd-foundations-skill/
├── ddd-persistence/
├── ddd-services/
└── ddd-value-objects/
```

## Contributing

If you add a new DDD concern, keep the format consistent with the existing skills:

- clear purpose statement,
- when-to-use guidance,
- core rules,
- anti-patterns,
- quick reference,
- and companion skills.

That consistency is what makes the collection useful to both humans and AI assistants.
