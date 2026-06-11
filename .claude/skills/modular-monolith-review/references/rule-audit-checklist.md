# Rule audit checklist (Lens 1)

Each of the project's hard boundary rules, what a violation looks like in C#, how to confirm it, and the fix. These are binary defects, not judgment calls. Many should be caught by the NetArchTest suite; when a review finds one the suite missed, that is also a gap in the suite to flag.

## Module isolation

**Rule:** a module references another module only through its `Contracts`.

**Violation looks like:** a `using LearningManagementService.Modules.Cartridges.Domain;` (or `.Application`, `.Infrastructure`) inside the LearningEngagement module. A constructor or method taking another module's domain entity, repository, or DbContext.

**How to confirm:** search the module's `using` directives and type references for any other module's non-`Contracts` namespaces. Check DI registrations and constructor parameters for cross-module concrete types.

**Fix:** depend on the other module's contract interface instead; if data is needed, get a DTO through it. If behavior is needed, the owning module should expose it via its contract.

## Contract purity

**Rule:** a module's `Contracts` contains only interfaces and DTOs and depends on nothing heavy.

**Violation looks like:** an EF `using` in `Contracts`; a `Contracts` type referencing the module's own `Domain` entity; a concrete service class (with logic, not just a record) living in `Contracts`; a DTO carrying a domain entity as a property.

**How to confirm:** inspect every type in the `Contracts` namespace. Each should be an interface or an immutable DTO (`record`). No method bodies with logic, no EF/persistence references, no domain entity references.

**Fix:** move implementations to `Infrastructure` (marked `internal sealed`); replace domain-entity properties on DTOs with primitive/value-object/ID fields; remove EF dependencies.

## No cross-module foreign keys

**Rule:** FKs within a module are fine; FKs across modules are forbidden. Cross-module references are plain ID columns with no constraint.

**Violation looks like:** an EF mapping with `.HasForeignKey(...)` or a navigation property pointing at an entity owned by another module; a migration creating an FK constraint between two modules' tables; a LINQ query joining across two modules' table sets.

**How to confirm:** inspect EF configurations and migrations for FK constraints and navigation properties crossing module boundaries. Check that a cross-module reference is stored as a bare `CartridgeVersionId` column, not a `CartridgeVersion` navigation.

**Fix:** drop the cross-module FK; store the strongly typed ID as a plain column; validate existence through the owner's contract on write rather than relying on the database constraint.

## Cross-module calls through contracts

**Rule:** all cross-module interaction goes through the contract interface, synchronously and in-process.

**Violation looks like:** one module resolving or constructing another module's internal service; shared mutable static state used as a back channel; a module querying another module's DbContext; reflection used to reach internals.

**How to confirm:** trace how data/behavior actually crosses the boundary at runtime. The only legitimate channel is an injected contract interface returning DTOs.

**Fix:** route the interaction through the owner's contract interface; if no suitable contract method exists, add one to the owner's `Contracts` and implement it internally.

## Layering within a module

**Rule:** `Domain` depends on neither `Application`, `Infrastructure`, nor EF. `Shared.Kernel` depends on no module.

**Violation looks like:** a `Domain` entity with an EF attribute or a `using Microsoft.EntityFrameworkCore`; a `Domain` type referencing an `Application` handler or `Infrastructure` repository; a `Shared.Kernel` type referencing any `Modules.*` namespace.

**How to confirm:** inspect `using` directives and type references in `Domain` and in `Shared.Kernel`.

**Fix:** invert the dependency, the `Domain` defines interfaces, outer layers implement them; move persistence concerns to `Infrastructure`; remove any module-specific type from the Kernel (it belongs in the owning module).

## Public surface (internal vs contract)

**Rule:** types outside `Contracts` are `internal` (controllers and the module registration class excepted). `Contracts` exposes only interfaces and DTOs. No domain entity escapes the module.

**Violation looks like:** a `public` application service or domain entity outside `Contracts`; a contract method returning a `Domain` entity instead of a DTO; an `internal` type that should be the contract but isn't declared in `Contracts`.

**How to confirm:** list `public` non-controller, non-`Module` types in the module outside `Contracts`. Check every contract interface's return types, all must be DTOs/value objects/IDs, never domain entities.

**Fix:** mark internal what should be internal; map domain entities to DTOs at the contract boundary; if a type is genuinely contract surface, move its interface into `Contracts`.

## Synchronous cycles

**Rule:** cross-module dependencies point one direction; A↔B synchronous cycles are forbidden (the weld equivalent of a cross-module FK).

**Violation looks like:** LearningEngagement depends on `ICartridgeCatalog` (Cartridges) *and* Cartridges depends on `ICourseUsageQuery` (LearningEngagement), with both resolved synchronously. Often shows up as a DI resolution that only works by accident of ordering, or a latent stack-overflow risk.

**How to confirm:** build the directed graph of cross-module contract dependencies. Any cycle is a violation.

**Fix:** break the cycle by reversing one direction with a domain event, the owner of the fact publishes it, the other module reacts, so it no longer needs a synchronous call back. This is the specific, sanctioned case for introducing eventing (otherwise deferred). Alternatively, the cycle may indicate the responsibility is on the wrong side of the boundary, move it.

## Using this checklist

Go rule by rule for the scope under review. For each violation: record the rule, the location, the concrete fix, and whether an architecture test should have caught it. A clean Lens 1 pass does not mean the architecture is healthy, it means it is *legal*; the coupling quality is Lens 2's job (`balanced-coupling.md`).
