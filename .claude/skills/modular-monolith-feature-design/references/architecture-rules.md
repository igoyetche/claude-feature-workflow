# Architecture rules (full reference)

The complete boundary rule set for this modular monolith, with rationale. The project is in domain discovery; the rules exist to keep boundaries cheap to move, not to lock them in.

## Why a modular monolith

A single deployable, internally partitioned into modules with strong logical boundaries. Chosen because the domain is not yet understood well enough to know where service boundaries belong. Committing to network boundaries prematurely encodes boundary guesses into infrastructure, where they are expensive to move and impose distributed-systems costs on a decomposition that is probably wrong. The monolith keeps logical separation while deferring those costs until a boundary has earned a network gap. A modular monolith is also a legitimate permanent destination, not merely a waiting room for microservices.

## Module-first organization

The top-level unit is the **module** (a candidate bounded context). Technical layers live **inside** each module. This is the inverse of the layer-first layout (Controllers/Services/Repositories at the root) that ASP.NET MVC nudges toward. Layer-first scatters a capability across every folder and provides no boundaries; module-first makes a capability legible in one place and its boundaries visible and enforceable.

### Folder layout

```
src/
  LearningManagementService.Host/      Thin composition root; wires modules together
    Program.cs
    appsettings.json
  Modules/
    <ModuleName>/
      Controllers/                     Thin entry points (public, by MVC necessity)
      Domain/                          Entities, value objects, domain logic
      Application/                     Use cases, handlers (internal)
      Contracts/                       The ONLY surface other modules may reference
      Infrastructure/                  Persistence, external clients, contract implementations (internal)
      <ModuleName>Module.cs            This module's DI + endpoint registration
  Shared/
    Kernel/                            Universal primitives only
```

The host stays thin: `Program.cs` calls one registration method per module (`builder.Add<ModuleName>Module()`). Each module owns its own DI, configuration, and endpoint registration in one place.

Project vs folder boundaries (ADR 3). Make the boundaries already trusted into compile-time boundaries, and keep the unsettled ones as folders. Each module's `Contracts` is its **own project**, and `Shared.Kernel` is its **own project**, so cross-module access to anything but contracts, and any module dependency from the Kernel, does not compile. A coarse module whose boundary is still uncertain may share a project with its siblings (split expressed as folders/namespaces), and a module's internal layers (`Domain`, `Application`, `Infrastructure`) stay as folders, policed by the architecture tests. Promote a full module to its own project once its boundary stabilizes. This matters more under agent-driven development: an agent pattern-matches on what compiles, so a compiler-enforced reference is a far stronger guardrail than a test-time namespace check for the boundaries that are already decided, while folders keep the genuinely unsettled boundaries cheap to reshape.

## The boundary rules

1. **Reference only through Contracts.** A module may reference another module only through that module's `Contracts`. Never reach into another module's `Domain`, `Application`, or `Infrastructure`.

2. **Reads go through a contract interface returning a DTO.** Cross-module reads call a public interface defined in the owning module's `Contracts`, which returns a DTO, never an internal domain entity.

3. **Foreign keys within a module: encouraged. Across modules: forbidden.** Inside a module the entities form one connected model; enforce referential integrity with the database normally. Across boundaries, store the other module's identifier as a plain column (a strongly typed ID in code) with no FK constraint pointing at the owning module's tables. A cross-module FK physically entangles two schemas, blocks per-module schema separation, and must be unpicked before a module can ever be extracted. Cross-context consistency is the application's job, not the database's.

4. **Cross-module calls are synchronous, in-process, one-directional.** The consumer depends only on the interface; the owner provides the implementation. Keep dependencies pointing one way. A synchronous cycle between two modules is a weld, the equivalent of a cross-module FK, and is the signal to introduce a domain event so one dependency can reverse direction.

## Coarse modules, split on evidence

Start with few, deliberately coarse modules. Splitting an overlarge module is local (find the seam, cut); merging two drawn too small means reconciling hardened contracts and schemas. Erring coarse is the cheaper error. Split when concrete friction appears: a module acquires two distinct reasons to change, two vocabularies for the same term, or two sub-areas that never touch each other's data. That friction is the domain revealing the boundary, more reliable than an up-front guess.

## The Shared Kernel

Holds only universal, stable primitives: strongly typed ID base types, `Result`/`Error` types, base entity/aggregate types, domain-agnostic value objects (`Money`, `DateRange`). It must never hold a domain entity, business behavior, or anything two modules would evolve in different directions. The governing test: if two modules would push a type in different directions, it does not belong in the Kernel. A domain entity in the Kernel couples every module to every other module's reasons for changing it.

## Enforcement

A NetArchTest suite in CI enforces: module isolation (no reaching past another module's `Contracts`), contract purity (`Contracts` depends on no `Domain`/`Application`/`Infrastructure`/EF), layering (`Domain` depends on no `Application`/`Infrastructure`/EF; Kernel depends on no module), and public surface (types outside `Contracts` are `internal`; `Contracts` holds only interfaces and DTOs). The tests reason about namespaces and visibility, so they are a strong safety net, not a proof; namespace discipline matters.
