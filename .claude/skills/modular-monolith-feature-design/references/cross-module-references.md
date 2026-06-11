# Cross-module references (decision guide + patterns)

When a feature in module B needs something owned by module A, B needs exactly one of three things. Conflating them produces a shared God entity that couples everything. Identify the case first, then apply the matching pattern. The running example below is LearningEngagement (B) needing things from Cartridges (A).

## The decision: which of three?

Ask what B actually needs from A's entity:

| If B needs to... | It is a... | Mechanism |
| --- | --- | --- |
| Point at A's entity (store the link) | **Reference** | Store a strongly typed ID. No object, no FK. |
| Read some of A's data | **Read a slice** | Local view type + call A's contract interface. |
| Maintain its own model that shares an identity | **Own concept** | Model locally; share only the ID. |

Most needs are **Reference**. The instinct to create one canonical entity every module shares is the thing to resist: the same real-world thing is usually modeled differently per context, linked only by a shared ID.

## Pattern: Reference by ID

B stores A's identifier, not A's object. Use a strongly typed ID so the reference is named and the wrong ID cannot be passed.

```csharp
// In LearningEngagement's domain. Cartridges' types are never imported.
public sealed class Course
{
    public CourseId Id { get; }
    public LearnerId LearnerId { get; }                     // the learner this Course belongs to (identity resolved from the token, owned outside this service)
    public CartridgeVersionId CartridgeVersionId { get; }   // the published version this Course instantiates, owned by the Cartridges module
    // ...Course's own state and invariants
}
```

This dissolves most "cross-module reference" problems: usually you do not need the entity, only to point at it.

## Pattern: Read a slice through a contract

When B needs A's *data* at request time, A exposes an interface in its `Contracts` returning a DTO. B defines its own lean view of the concept and depends only on the interface.

```csharp
// Cartridges.Contracts — the public surface (interfaces + DTOs only)
public interface ICartridgeCatalog
{
    Task<CartridgeVersionSummary?> GetVersionSummaryAsync(CartridgeVersionId id, CancellationToken ct);
    Task<bool> VersionExistsAsync(CartridgeVersionId id, CancellationToken ct);
}

public sealed record CartridgeVersionSummary(CartridgeVersionId Id, string Title, CartridgeVersionStatus Status);
```

```csharp
// Cartridges.Infrastructure — implementation, internal, never escapes the module
internal sealed class CartridgeCatalog : ICartridgeCatalog
{
    private readonly CartridgesDbContext _db;
    public CartridgeCatalog(CartridgesDbContext db) => _db = db;

    public async Task<CartridgeVersionSummary?> GetVersionSummaryAsync(CartridgeVersionId id, CancellationToken ct)
    {
        var v = await _db.CartridgeVersions.FindAsync([id], ct);
        return v is null ? null : new CartridgeVersionSummary(v.Id, v.Title, v.Status);
    }
    // ...
}
```

```csharp
// CartridgesModule.cs — the owner wires its own implementation
services.AddScoped<ICartridgeCatalog, CartridgeCatalog>();
```

LearningEngagement depends on `ICartridgeCatalog` and `CartridgeVersionSummary` only; it never sees `Cartridges.Domain.CartridgeVersion`. The lean `CartridgeVersionSummary` is not duplication, it is the consumer's context modeling the concept in its own language.

## Pattern: Own concept

B's model and A's model are genuinely different things that share an identity. The authored `ExperienceType` that Cartridges owns (its definition within a Cartridge, its structural validation) is a different model from the `ExperienceInstance` that LearningEngagement owns (a Learner's actual run of it, carrying Completion state). Each module owns its own model; they share only the `ExperienceTypeId`. No contract call may even be needed.

## The mechanism is synchronous and in-process

Cross-module calls are plain method calls through the contract interface, resolved by DI. No event bus, no separate read model. Consumers do **not** write their own client class while in-process; the injected interface is the direct call. A consumer-side HTTP/gRPC client appears only if the owning module is later extracted, at which point the interface is unchanged and only the registered implementation swaps. Consumers do not change. That is the payoff for routing every cross-module call through a contract.

## Keep dependencies one-directional

If A calls B and B calls A synchronously, that is a cycle, two modules that cannot be understood or extracted independently. A cycle is the moment a **domain event** is the right tool: the owner publishes a fact, the consumer reacts, and the dependency points one way. This is the specific, concrete trigger for introducing an event. Absent a cycle (or an extraction, or a genuinely async process), events stay deferred.

## Consistency without cross-module FKs

Because there is no database FK across modules, the database will not stop B from referencing a deleted A entity. Cross-context consistency becomes explicit and application-level:

- **Validate on write** through the owner's contract (`await _catalog.VersionExistsAsync(id, ct)` before opening a Course on that CartridgeVersion).
- **React to lifecycle changes** through the owner when needed (when "CartridgeVersion retired" matters to in-flight Courses, model that as a reaction, eventually via an event if a cycle would otherwise form, rather than a database cascade across a boundary).

Consistency stays strong *within* a context (FKs, transactions) and becomes eventual and explicit *across* contexts. That is the correct trade for keeping boundaries real.
