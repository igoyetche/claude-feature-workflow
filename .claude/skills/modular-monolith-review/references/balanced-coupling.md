# Balanced Coupling (for a .NET modular monolith)

This reference applies the **Balanced Coupling** model to coupling reviews in this codebase. The model, its three dimensions, and the balance rule are the work of **Vlad Khononov**, published at **coupling.dev** (and in his book *Learning Domain-Driven Design*, O'Reilly). It is used here as an analytical lens, the way one would apply any published architectural model. Attribute it and link to coupling.dev in review output; do not present it as this project's own.

## Why this lens exists

The project's hard rules and the architecture tests catch *illegal* coupling (a module reaching past another's `Contracts`, a cross-module FK). They cannot catch coupling that is *legal but costly*: two modules that talk only through contracts can still be painful to change together if they share too much knowledge, sit far apart, and both change often. Balanced Coupling gives a vocabulary and a rule for judging that, so a review produces "this integration is imbalanced because X" rather than a gut feeling.

The core insight: coupling is not simply good or bad. It is balanced or imbalanced. The cost of a coupling depends on the interaction of three dimensions.

## The three dimensions

### Integration Strength — how much knowledge is shared

How much one component must know about another to integrate with it. From strongest (worst, most knowledge shared) to weakest (best):

- **Intrusive** — one component depends on another's *implementation details* that were never meant to be shared: reading its private data, depending on its internal behavior, reaching into its database tables. In this codebase: a module reading another module's tables directly, or depending on another module's `Domain`/`Infrastructure`. This is also a hard-rule violation.
- **Functional** — sharing knowledge of business *functionality* or rules: two components that must both know and agree on a business process or calculation. Changing the rule forces both to change.
- **Model** — sharing a *model* of the domain: components pass around the same rich domain objects, so a change to the model ripples to everyone using it. In this codebase: a shared `CartridgeVersion` entity used by several modules (the God-entity trap), or domain types leaking through contracts.
- **Contract** — sharing only an *integration contract*: a deliberately designed, minimal DTO/interface that decouples the consumer from the provider's internals. This is the weakest, most desirable strength, and exactly what this project's synchronous-contract pattern aims for. A well-designed `CartridgeVersionSummary` DTO is contract-level strength.

Lower strength = less shared knowledge = changes propagate less. The project's whole design pushes cross-module integration toward **contract** strength.

### Distance — the cost of co-evolving the components

How far apart the two components are, socio-technically, which sets how expensive it is when they must change together:

- **Code structure distance**: same class/method (near) → same module → different modules → different projects → different repositories/services (far). Further apart means more friction to change together.
- **Team distance**: same person/team (near) → different teams → different departments (far). Two modules owned by different teams are more expensive to co-change than two owned by the same person.
- **Runtime distance**: in-process (near) → separate processes/services over a network (far). In a monolith most coupling is in-process (near), which is part of why a monolith tolerates more coupling than microservices do.

In this modular monolith, distance is currently *low* for most integrations (one process, likely one or two people, possibly one project during discovery). That low distance is significant: it means the monolith can tolerate couplings that would be dangerous across services. It also means that **extracting a module raises distance sharply**, which is why a coupling that is balanced today can become imbalanced the day you extract.

### Volatility — how likely the components are to change

The probability that a component will need to change, judged from the *business domain*, not the code. This is where domain classification feeds in:

- **Core subdomain** — the competitive advantage; changes frequently as the business experiments and refines. High volatility.
- **Supporting subdomain** — necessary but not differentiating; changes occasionally. Medium volatility.
- **Generic subdomain** — a solved problem (identity via Entra ID, scheduling via the Scheduler, content delivery via the Bot/UI); changes rarely. Low volatility.

Volatility is the dimension that decides whether an imbalance actually matters. Strong coupling to something that never changes is cheap; weak coupling to something that changes constantly may still be fine; strong coupling to something volatile is where pain compounds.

## The balance rule

Khononov's rule:

```
BALANCE = (STRENGTH XOR DISTANCE) OR NOT VOLATILITY
```

Read it as: a coupling is **balanced** when strength and distance are on *opposite* ends (high strength with low distance, or low strength with high distance), **or** when volatility is low enough that it does not matter.

The practical readings:

- **High strength + low distance = balanced.** Components that share a lot of knowledge should be close together. Two tightly related types in the same module sharing a model is fine, they are near, so co-changing them is cheap. (This is also why FKs *within* a module are fine: high strength, but minimal distance.)
- **Low strength + high distance = balanced.** Components that are far apart should share little knowledge. Two services across a network talking through a thin contract is fine. (This is the target state for any module you later extract: weaken strength to contract level *before* distance rises.)
- **High strength + high distance = IMBALANCED.** Sharing lots of knowledge across a large distance is the costly case: every change forces coordinated edits across far-apart, possibly differently-owned components. This is the distributed-monolith smell. Flag it.
- **Low strength + low distance = wasteful but safe.** Lots of ceremony (elaborate contracts) between two things sitting right next to each other. Not dangerous, just over-engineered; during discovery, usually not worth flagging unless it adds real friction.
- **`OR NOT VOLATILITY`** — if volatility is low, even an imbalance is tolerable, the components rarely change, so the coordination cost is rarely paid. A high-strength, high-distance coupling to a *generic, stable* subdomain may be acceptable. Conversely, the imbalances to prioritize are always the ones on *high-volatility* (core subdomain) integrations.

## How to score an integration here

For each cross-boundary integration in scope:

1. **Strength**: what is actually shared? Tables/internals = intrusive (also a rule violation). A shared domain model/entity = model. A shared business rule both sides encode = functional. A purpose-built DTO/interface = contract. Record the level.
2. **Distance**: same module or different? Same owner or different? In-process (yes, during the monolith phase) or would this survive extraction? Record near/medium/far, and note what extraction would do to it.
3. **Volatility**: what subdomain does each side belong to, core/supporting/generic? Record high/medium/low.
4. **Apply the rule**: is strength counterbalanced by distance, or is volatility low? If neither, it is imbalanced, flag it with the specific reason.

## The coupling overview table

Record findings in a table like this in the review:

| Integration | Strength | Distance | Volatility | Balanced? | Note / recommendation |
| --- | --- | --- | --- | --- | --- |
| LearningEngagement → Cartridges | Model (shares `CartridgeVersion`) | Near (same project, same owner during discovery) | High (authoring is core) | Tolerable now | Strength is high but distance is low, so balanced today. Risk: extracting Cartridges raises distance and breaks balance. Weaken to a `CartridgeVersionSummary` contract DTO before any extraction. |
| LearningEngagement → Entra ID (identity) | Contract (`LearnerId` from token) | Far (external service) | Low (generic) | Balanced | Low strength + high distance, and low volatility. Healthy. |
| Cartridges ↔ LearningEngagement | Functional (both encode the Eligibility/unlock rule) | Different modules | High (core) | Imbalanced | The rule that decides what unlocks next is duplicated across authoring validation and runtime evaluation; a change forces coordinated edits in both. Define it once: keep authoring to pure structure and let LearningEngagement own Eligibility, exposed through its contract. |

## Translating imbalances into this project's fixes

Every recommendation should land in the project's own vocabulary:

- **Reduce strength** (the most common fix): replace a shared domain entity (model strength) with a purpose-built contract DTO (contract strength). This is exactly the "read a slice through a contract" pattern. Intrusive strength is a hard-rule violation, fix the boundary breach first.
- **Reduce distance**: rarely the right move during discovery (it usually means merging modules). But if two modules are *always* changing together and share a lot, low distance is correct, they may genuinely be one module. That is a **seam signal**: the boundary is misplaced.
- **Accept it (for now)**: if volatility is low, document why the imbalance is tolerated rather than churning to fix it. Note it as **Watch** in case volatility rises.
- **Pre-empt extraction risk**: for any module that might be extracted, weaken cross-boundary strength to contract level *now*, while distance is still low and the change is cheap, so that raising distance later keeps the coupling balanced.

## Caveats

Scoring has judgment in it, especially volatility, which depends on business knowledge the reviewer must get from the user, not invent. State the assumptions behind each score so the user can correct them. The model is a lens for structured reasoning, not a precise metric; its value is making the *reason* for a coupling concern explicit and traceable to a dimension, rather than asserting a number.
