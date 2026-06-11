---
name: modular-monolith-review
description: Use when auditing, reviewing, assessing, or checking existing C#/ASP.NET Core code, a module, a pull request, or the whole service for modularity problems, architectural drift, coupling issues, or boundary violations, or when asking why a change in one place keeps breaking another, whether two modules are too entangled, whether a boundary is in the right place, or whether the architecture is decaying as the codebase grows. Triggers on "is this coupled too tightly", "review this module", "what's wrong with our boundaries", "why does X break when I change Y", and on intra-module DDD/layering checks — whether an aggregate is well-encapsulated, whether the domain is anemic, whether business rules leaked into a controller or application service, whether Domain/Application/Infrastructure/Controllers layering holds. Counterpart to modular-monolith-feature-design: that one designs new features to fit the rules; this one audits existing code against the rules and the Balanced Coupling model.
---

# Modular Monolith Review (C#)

## Purpose

Audit existing code in this C#/ASP.NET Core modular monolith for two kinds of problem: **hard rule violations** (the project's boundary rules, which the architecture tests are meant to catch) and **coupling imbalances** (subtler problems the tests cannot see, where modules are technically legal but co-evolve painfully). The output is a prioritized, actionable review, not a pass/fail.

The project is in **domain discovery**, so the goal of a review is not to enforce a final architecture but to keep boundaries cheap to move and to surface seams the domain is revealing. A review that says "these two modules keep changing together" is discovery data, not just a defect report.

## When NOT to use this skill

- **Designing or placing a *new* feature** — that is the `modular-monolith-feature-design` skill. This skill judges code that already exists.
- **A purely mechanical rule check with nothing on disk to read** — if the only question is "does this compile against the boundary rules", the NetArchTest suite answers it on build. Use this skill when there is code to read and a coupling-quality or seam judgment to make.
- **Reviewing for things outside modularity** — general bug hunting, security, or performance. Those are other reviews; this one is about boundaries and coupling balance.

## Two lenses

Apply both. They catch different things. Lens 1 has two checklists — cross-module boundaries (1a) and intra-module DDD/layering hygiene (1b).

### Lens 1a: The project's cross-module hard rules

These are binary; a violation is a defect to fix (and ideally a missing architecture test). Check each:

- **Module isolation** — does any module reference another module's `Domain`, `Application`, or `Infrastructure` instead of only its `Contracts`?
- **Contract purity** — does any `Contracts` namespace depend on its own `Domain`/`Application`/`Infrastructure` or on EF? Does it expose anything other than interfaces and DTOs?
- **No cross-module foreign keys** — does any database mapping define an FK across a module boundary? (References across modules must be plain ID columns, no constraint.)
- **Cross-module calls through contracts** — does any cross-module interaction bypass the contract interface (direct entity access, shared mutable state, reaching into a DbContext that isn't the module's own)?
- **Layering** — does any `Domain` depend on `Application`, `Infrastructure`, or EF? Does the `Shared.Kernel` depend on any module?
- **Public surface** — is anything outside `Contracts` `public` when it should be `internal` (excepting controllers and module registration)? Are there domain entities leaking through a contract?
- **Synchronous cycles** — do two modules call each other synchronously (A→B and B→A)? This is the weld equivalent of a cross-module FK.

See `references/rule-audit-checklist.md` for what each violation looks like in C# and how to confirm it.

### Lens 1b: Intra-module DDD & layering hygiene

A module can be perfectly isolated from its neighbours and still rot inside: an anemic domain with public setters, an aggregate whose children navigate back to the root, a fat application service holding business rules, a controller reaching into a repository. These checks are also mostly binary, but they look *within* a module rather than across boundaries. Cover domain encapsulation (factories, no public setters, aggregate navigation discipline — no child→root back-references, peers by ID, tree-shaped graph, mutation through the root), the application layer (no business rules, returns DTOs, translates primitives to value objects, no auto-mapping into entities), controllers (thin, no business rules, identity passed in, auth split), infrastructure (EF encapsulation, no persistence attributes on the domain), and cross-cutting (injected time, domain-generated IDs, no logging in the domain).

See `references/intra-module-review-checklist.md` for the full set, each with what a violation looks like in C# and how to confirm it, plus a quick-scan smell table and a layering cheat sheet. It respects the project's deferrals (events/CQRS/Unit of Work are reviewed only if present).

### Lens 2: Balanced Coupling (Vlad Khononov's model)

This lens catches coupling that is *legal under the rules but still costly*. Two modules can talk only through contracts and still be badly coupled if they share too much knowledge, sit far apart, and change often together. Assess each significant integration across three dimensions and apply the balance rule. This is the heart of the review; see `references/balanced-coupling.md` for the full method, then record findings in the coupling table format described there.

The model and its terminology are Vlad Khononov's, published at coupling.dev. Attribute it and link concepts there in the output; do not present it as this project's invention.

## Workflow

### 1. Establish scope and context

Determine what is being reviewed (one module, an integration between two, a PR, or the whole service). Then gather the context the coupling analysis needs, ask the user, do not assume:

- **Domain classification** of the areas in scope: is each a *core* subdomain (the competitive advantage, changes often), a *supporting* subdomain (necessary but not differentiating), or a *generic* subdomain (solved problems, buy/adopt rather than build)? Volatility largely follows from this.
- **Team and ownership** structure: who owns which modules? (Distance is partly socio-technical.)
- **Known pain points**: where have changes recently rippled across boundaries? What broke when something else changed? These are the symptoms the model explains.

If reviewing code that exists on disk, read it to map the actual dependencies before judging them. Do not review from the user's description alone when the code is available.

### 2. Map the integrations

For each pair of components (modules, or notable types across a boundary) that interact, identify how they integrate: what knowledge passes between them, through what mechanism (contract call, shared ID, shared Kernel type, shared database), and in which direction. Build the dependency picture before scoring it.

### 3. Run Lens 1 (structural rules: 1a cross-module, 1b intra-module)

Go through both checklists — `rule-audit-checklist.md` for cross-module boundaries, then `intra-module-review-checklist.md` for DDD/layering hygiene within each module in scope. For each violation, record what it is, where, and the fix. Note whether an architecture test should have caught it; a rule violation the NetArchTest suite missed is also a gap in the suite worth flagging. The quick-scan smell table in the intra-module checklist is a fast first pass over a PR before the rule-by-rule walk.

### 4. Run Lens 2 (Balanced Coupling)

For each significant integration, score Integration Strength, Distance, and Volatility, then apply the balance rule. Flag imbalances. Translate each imbalance into a concrete recommendation in this project's vocabulary (e.g. "weaken strength: replace the shared `CartridgeVersion` type with a `CartridgeVersionSummary` contract DTO" or "the volatility is high and these sit far apart, so reduce strength to contract level"). Full method in the reference.

### 5. Synthesize and prioritize

Produce the review document. Lead with the highest-leverage findings, usually a high-strength, high-distance, high-volatility coupling, or a hard-rule violation that's actively causing the pain points the user named. Distinguish:

- **Fix now** — hard rule violations, and imbalances on high-volatility integrations (these will hurt repeatedly).
- **Watch** — imbalances on currently low-volatility integrations (fine for now; revisit if volatility rises).
- **Seam signal** — coupling that suggests a module boundary is misplaced (two modules always changing together may be one module; one module with two independent clusters may be two). This is discovery information; surface it as such.

End with what, if anything, suggests the current boundaries should move, since that is the most valuable output during discovery.

## Output format

Produce a Markdown review document with: a short context summary (scope, domain classification, pain points), a coupling overview table (the format in `references/balanced-coupling.md`), a structural-findings list (Lens 1a cross-module boundary violations and Lens 1b intra-module DDD/layering violations), the prioritized recommendations (Fix now / Watch / Seam signal), and a closing note on boundary health. Keep recommendations concrete and in the project's terms. When referencing a Balanced Coupling concept, link it to coupling.dev and attribute the model to Vlad Khononov.

## Relationship to the other skill and the tests

This skill audits; the `modular-monolith-feature-design` skill builds. The NetArchTest suite enforces the hard rules mechanically on every build. This review covers what the tests cannot: the coupling-quality judgment, the domain-classification context, and the question of whether the boundaries themselves are right. A finding here is often best resolved by either a redesign (hand to the design skill) or a new architecture test (so the rule violation can never recur silently).

## Reference files

- `references/rule-audit-checklist.md` — Lens 1a: each cross-module hard rule, what its violation looks like in C#, and how to confirm and fix it.
- `references/intra-module-review-checklist.md` — Lens 1b: the intra-module DDD/layering checks (domain encapsulation and aggregate navigation, application, controller, infrastructure, cross-cutting, tests), plus a quick-scan smell table and a layering cheat sheet. Adapted to this project's layer names and its deferrals.
- `references/balanced-coupling.md` — Lens 2: the Balanced Coupling model in full: the three dimensions, how to score each for a .NET modular monolith, the balance rule, and the coupling table format. Attributed to Vlad Khononov (coupling.dev).
