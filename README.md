# Agentic Skills for DDD

A curated set of domain-driven design (DDD) skills for AI coding assistants. The set is an opinionated, stack-neutral playbook for modeling bounded contexts with clear boundaries, explicit language, and consistent layering. Each skill helps an assistant reason about one DDD concern without collapsing everything into generic CRUD, service blobs, or framework-first design.

**Version:** `ddd-skills 1.0` (see `VERSION`). Core Rule numbering is frozen within a major version; stack skills cite rule numbers against it.

## Two tiers: design here, mechanics in a stack skill

These skills decide **what**: the layers, the shape of an aggregate, the repository contract, who owns the transaction, when an event is delivered. They are the same in every language and framework, and they stay static.

**How** a particular framework realizes those decisions — the transaction annotation, the ORM mapping on the persistence mirror, the lock-conflict → 409 wiring, after-commit event delivery, the test tiers — lives in a separate **mechanics skill per stack**, named `<stack>-ddd`:

| Stack | Skill | Where |
| --- | --- | --- |
| Quarkus | `quarkus-ddd` | [agentic-skills-for-quarkus](https://github.com/jeremyrdavis/agentic-skills-for-quarkus) |
| Spring Boot | `spring-ddd` | planned |
| Python stacks | `fastapi-ddd`, … | planned |

Install the DDD skills plus the one stack skill your project uses. Every DDD skill points at `<stack>-ddd` with a soft reference: if it is installed the assistant loads it for the mechanics and the stack's worked example; if not, the DDD rules still apply and the assistant uses your framework's equivalents.

A stack skill must not redefine any DDD concept. It supplies annotations, framework idioms, configuration, and worked examples that cite the DDD Core Rules by number — nothing else.

### Sibling: the TDD skills

[agentic-skills-for-tdd](https://github.com/jeremyrdavis/agentic-skills-for-tdd) (`tdd-skills`: `tdd-test-list`, `tdd-cycle`) follows the same two-tier model and is built to run alongside these skills. Precedence: `tdd-cycle` governs the sequence of work; the `ddd-*` skills constrain the interface decisions made while writing each test and the layering; implementation decisions in the refactor step are TDD's; `<stack>-ddd` and `<stack>-tdd` supply mechanics. A DDD + TDD Quarkus project installs `ddd-skills`, `tdd-skills`, and `quarkus-skills`.

## What this repository contains

Six skills under `skills/`:

- `ddd-foundations` — the shared layering model (`domain` / `application` / `infrastructure` / `api`) and terminology every other skill references.
- `ddd-aggregates` — aggregate roots: identity, named factories, `rehydrate(...)`, state transitions, invariants, cross-aggregate references by ID, the `version` field, raising domain events.
- `ddd-value-objects` — immutable value objects, constructor validation and normalization, equality by value, typed ID wrappers.
- `ddd-services` — application services (orchestrate, own the transaction, return DTOs) versus domain services (pure computation).
- `ddd-persistence` — repository interfaces in `domain`, same-name implementations in `infrastructure`, the persistence mirror, optimistic locking, atomic operations, use-case-optimal queries.
- `ddd-domain-events` — what an event is and carries, after-commit delivery, idempotent handlers, domain versus integration events.

Each skill is a `SKILL.md` with the same structure: a foundational principle, red flags, when to use, numbered Core Rules, anti-patterns, an excuse/reality table, a quick-reference checklist, and companion skills. The `ddd-aggregates` and `ddd-value-objects` skills also carry stack-neutral worked examples in `references/` (Java, Python, TypeScript). The framework-dependent worked examples (foundations, services, persistence, events) live in the stack skills; Python and TypeScript fallbacks remain here until their stack skills exist.

## Install

There is no `claude mcp add-skill` command. Use one of the following.

### Plugin, per project (recommended)

Add the marketplace once, then install into each project that uses DDD. Project scope records the plugin in that project's `.claude/settings.json`, so nothing is loaded in projects that don't use it.

```bash
claude plugin marketplace add jeremyrdavis/agentic-skills-for-ddd
cd <your-project>
claude plugin install ddd-skills@agentic-skills-for-ddd --scope project
```

One-step form from inside a Claude Code session:

```
/plugin install ddd-skills --marketplace jeremyrdavis/agentic-skills-for-ddd
```

Use `--scope local` for a personal, uncommitted install, and `--scope user` only if you want the skills in every project. For a Quarkus project, also install `quarkus-skills` from the Quarkus repository.

### Copy, per project (no manifests)

```bash
git clone https://github.com/jeremyrdavis/agentic-skills-for-ddd.git
mkdir -p <your-project>/.claude/skills
cp -r agentic-skills-for-ddd/skills/ddd-* <your-project>/.claude/skills/
```

Claude Code loads `<project>/.claude/skills/<name>/SKILL.md` for that project. Copy all six; the skills cross-reference each other.

## How the skills fit together

- `ddd-foundations` tells you **where things belong**.
- `ddd-value-objects` tells you **how to model values**.
- `ddd-aggregates` tells you **how to model consistency boundaries**.
- `ddd-services` tells you **how to orchestrate use cases and domain computation**.
- `ddd-persistence` tells you **how to cross the storage boundary safely**.
- `ddd-domain-events` tells you **how facts leave an aggregate and reach a handler**.
- `<stack>-ddd` tells you **how your framework does each of those**.

For a new `Order` aggregate: the aggregate uses `ddd-aggregates`; its `OrderId` and `Money` use `ddd-value-objects`; the use case uses `ddd-services`; the repository uses `ddd-persistence`; `OrderPlacedEvent` uses `ddd-domain-events`; the package layout comes from `ddd-foundations`; and the `@Transactional`, Panache mirror, and Kafka outbox come from `quarkus-ddd`.

## Using the skills with Claude Code

Once installed, the skills fire on their own: each `SKILL.md` has a description listing the situations and code smells it triggers on, and Claude loads the body only when it matches. You don't need to paste skill content into the conversation. Ask in terms of the DDD concern:

- "Design the module layout for an orders bounded context."
- "Model `EmailAddress` and `Money` as value objects."
- "Refactor this use case so the transaction stays in the application service and the repository only translates."
- "Review this diff against the DDD skills and flag any boundary violations."

Asking for the skill's checklist and anti-patterns, not just code, keeps the output aligned with the architecture.

## Using the skills with Codex

Codex can use the same files as architectural guidance. Point it at the repository or copy the relevant `skills/ddd-*/SKILL.md` into its context, state the task in terms of the DDD concern, and ask it to consult the skill before changing code. For larger tasks, ask for a plan first; it is easier to check that the right skill is applied to the right layer before code is written.

## Repository structure

```text
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── VERSION
└── skills/
    ├── ddd-aggregates/
    │   ├── SKILL.md
    │   └── references/            (Java, Python, TypeScript — stack-neutral)
    ├── ddd-domain-events/
    ├── ddd-foundations/
    │   └── references/            (Python, TypeScript fallbacks)
    ├── ddd-persistence/
    │   └── references/            (Python, TypeScript fallbacks)
    ├── ddd-services/
    │   └── references/            (Python, TypeScript fallbacks)
    └── ddd-value-objects/
        └── references/            (Java, Python, TypeScript — stack-neutral)
```

## Contributing

A new DDD concern follows the existing template: purpose statement, red flags, when-to-use, numbered Core Rules, anti-patterns, excuse/reality, quick reference, companion skills. Keep `SKILL.md` stack-neutral; framework code belongs in a `<stack>-ddd` skill. Adding a rule appends to the list — numbering is frozen within a major version because stack skills cite it.

A new stack skill is named `<stack>-ddd`, defers every design decision to these skills, and carries `references/{foundations,services,persistence,domain-events}.md` plus whatever cross-cutting setup the stack needs. See `quarkus-ddd` for the shape.
