---
name: modular-monolith-feature-design
description: Use when designing, planning, adding, scaffolding, or structuring a new feature, capability, module, endpoint, or use case in this C#/ASP.NET Core service, or when deciding where code or data should live, how two modules should talk, whether a foreign key or cross-module reference is allowed, or how to model an entity several modules need. Triggers on any "how should I build/structure/place X" question about this codebase, even without the words "modular monolith". Apply before writing feature code so module boundaries, contracts, data ownership, and clean code are respected from the start rather than retrofitted.
---

# Modular Monolith Feature Design (C#)

## Purpose

Design features for this C#/ASP.NET Core modular monolith so they respect module boundaries, data ownership, and clean code from the first line, while the codebase is still in **domain discovery** (boundaries are provisional and meant to stay cheap to move).

The job here is *design*, not just code generation: decide which module owns what, how modules interact, where data lives, and how the public surface is shaped, then produce code that encodes those decisions. A feature placed correctly costs nothing later; a feature that quietly couples two modules is a weld that is expensive to undo.

## The prime directive

Optimize for **cheap reshaping**, not for guessing boundaries correctly up front. The domain is still being discovered, so the architecture's job is to make moving a boundary a refactor, not a migration. Every rule below serves that goal. When a rule and "what's quickest right now" conflict, the rule wins, because the quick version is what becomes the expensive weld.

## When NOT to use this skill

- **Auditing or reviewing code that already exists** for coupling or boundary drift — that is the `modular-monolith-review` skill.
- **A change wholly inside one module** that touches no boundary, no contract, and no data ownership (a bug fix, a rename, a tweak to internal logic). Just make it; no design pass needed.
- **Work that belongs to another MindShift service** — authentication/identity, deciding *when* learning happens, calendars, scheduling/cron, delivering content to learners. If the request implies one of those, the answer is usually "not here" rather than a design within this service.

## Workflow

Follow these steps in order when designing a feature. Do not skip to code.

### 1. Clarify the feature and find its owner

State what the feature does in one or two sentences. Then identify which single module **owns** it, the module responsible for its invariants and its source-of-truth data. If the feature seems to span modules, that is a signal: either it belongs to one module that the others merely reference, or you have found a seam (see step 6). Resist creating a new module on a hunch; prefer placing the feature in an existing coarse module until friction proves a split is needed.

### 2. Classify every cross-module need

For each piece of data or behavior the feature needs from **another** module, classify it as exactly one of:

- **Reference** — the feature only needs to point at the other module's entity. Store its strongly typed ID. Nothing more.
- **Read a slice** — the feature needs some of the other module's data. Define a small local view and obtain it through the owner's contract.
- **Own concept** — the feature has its own model that happens to share an identity with the other module's. Model it locally; share only the ID.

Most needs are *Reference*. Conflating these three is what produces a shared God entity. See `references/cross-module-references.md` for the full decision guide.

### 3. Design the public surface (internal vs contract)

Decide what, if anything, this feature must expose to other modules. The default is **nothing**: most features are internal to their module. Anything exposed goes through an interface in the owning module's `Contracts`, returning only DTOs, never domain entities.

The internal-vs-public rule, which you must encode in the code you write:

- A type that escapes the module returns **only** contract types. A type that does not escape can use domain entities freely.
- Contract **interfaces and DTOs** live in `Contracts`. Their **implementations** live in the module's `Infrastructure` (or `Application`) and are marked `internal sealed`.
- Everything outside `Contracts` is `internal` (controllers and the module registration class excepted). `public` is a deliberate act that means "this is contract surface."

### 4. Decide data ownership and references

Apply the data rules (these are firm):

- The owning module owns the feature's tables. Give the module its own schema.
- **Foreign keys within a module: yes.** Model relationships normally; let the database enforce integrity inside the context.
- **Foreign keys across modules: never.** Store the other module's identifier as a plain column (a strongly typed ID in code) with **no** FK constraint. Cross-context consistency is the application's job, validate through the owner's contract, not the database's.

### 5. Apply clean code, DDD, and C# practices

Design the types and methods to clean code and Domain-Driven Design standards. The essentials, with the full list in `references/clean-code-csharp.md`:

- Model identity with **strongly typed IDs** (`readonly record struct`), never bare `Guid`/`string`/`int`.
- Push invariants into the domain: rich entities and value objects, not anemic data bags with logic in services.
- Define **aggregates** with a single root as the consistency boundary; external code touches only the root, and other aggregates are referenced **by ID**, not by object. One transaction modifies one aggregate.
- Distinguish **entities** (identity, equality by ID, mutable through methods) from **value objects** (no identity, immutable, equality by value); most primitive obsession is a missing value object.
- One **repository per aggregate root**, dealing in whole aggregates; the interface lives in domain/application, the implementation in `Infrastructure`.
- Speak the **ubiquitous language** in code, tests, and column names; diverging language within a module is a seam signal.
- Make illegal states unrepresentable, constructors and factory methods that cannot produce an invalid object.
- Name by intent; keep methods small and single-purpose; depend on abstractions (interfaces injected via DI), not concretions.
- Return `Result`/`Error` types or throw domain-specific exceptions for expected failures rather than leaking nulls or generic exceptions across boundaries.
- Keep the aggregate's object graph a **tree**: children hold no back-reference to the root, peers reference each other by ID, and child state is mutated through a method on the root.
- Keep **application services free of business rules** (orchestrate: load → invoke domain → persist → commit → map to DTO; translate primitives to value objects; never reflection-map into an entity) and **controllers thin**, resolving identity from the token and passing it into the command rather than pre-validating domain decisions. Inject time via `IClock`, generate IDs in the domain, and keep logging out of the domain.

### 6. Check for boundary smells before finalizing

Before producing final code, scan the design for these signals and act on them:

- **A would-be cross-module FK** → replace with an ID reference + contract call.
- **A synchronous dependency cycle** (A calls B and B calls A) → this is a weld. Break it: usually one direction becomes a domain event (owner publishes, consumer reacts). Eventing is otherwise deferred (see below).
- **The same word meaning two things** in two modules → you have likely found a bounded-context seam. Flag it to the user; it may justify splitting a coarse module.
- **A type you want to put in `Shared/Kernel`** → only universal, stable primitives belong there (ID base types, `Result`/`Error`, base entity types, domain-agnostic value objects like `Money`). If two modules would evolve it in different directions, it does **not** belong in the Kernel.

### 7. Produce the design and code

Output the design decisions briefly (owner, cross-module needs and their classification, public surface, data ownership), then the code, organized by the module-first folder layout. Keep controllers thin. End by noting any seams or follow-ups you flagged in step 6.

## What is deliberately deferred

Do **not** introduce these unless a concrete, present need forces it; defaulting to them adds ceremony that makes discovery-phase code harder to reshape:

- **CQRS** — only when a read model genuinely cannot be served from the write model.
- **Domain/integration events and message brokers** — only when breaking a dependency cycle, supporting an actual extraction, or modeling a genuinely asynchronous process. The default for cross-module interaction is a **synchronous, in-process contract call**.
- **Separate physical databases / service extraction** — only when a module has earned a network boundary (independent scaling, deployment cadence, failure isolation, or tech stack).

Naming these as deferred is correct, not a shortcut. The synchronous-contract design is built so that introducing them later does not require touching consumers.

## Reference files

Read these when the feature touches the relevant area:

- `references/architecture-rules.md` — the full rule set with rationale and the module-first folder layout. Read when you need the complete boundary rules or are unsure how a structure should be laid out.
- `references/cross-module-references.md` — the Reference / Read-a-slice / Own-concept decision guide with C# examples, contract patterns, and the synchronous-contract mechanism. Read whenever a feature needs another module's data.
- `references/clean-code-csharp.md` — clean code, Domain-Driven Design, and idiomatic C# practices for this codebase: strongly typed IDs, rich domain modeling, aggregates and aggregate roots, aggregate navigation discipline (no back-references, peers by ID, tree-shaped graph, mutation through the root), entities vs value objects, repositories, ubiquitous language, domain services and events, error handling, application-service shape (orchestrate, no business rules, translate primitives, no auto-map into entities), thin controllers and token-based identity, cross-cutting (injected time, domain-generated IDs, no logging in the domain), DI, naming, testing. Read when designing the types and methods.

## A note on enforcement

This project enforces the boundary rules with a NetArchTest suite in CI (module isolation, contract purity, layering, public surface). Design as if the build will fail on a violation, because it will. If a design seems to require breaking a rule, that is a prompt to reconsider the design or to surface a genuine seam to the user, not to break the rule.
