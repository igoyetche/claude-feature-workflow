# Intra-module DDD & layering checklist (Lens 1b)

Lens 1a (`rule-audit-checklist.md`) catches **cross-module** boundary violations. This file catches the **intra-module** problems: a module can be perfectly isolated from its neighbours and still have an anemic domain, a fat application service, or an aggregate whose object graph is a tangle. These are DDD/clean-code conformance checks — mostly binary, like Lens 1a — and they keep each module's internals honest so the module stays cheap to understand, test, and move.

These are not all enforced by NetArchTest (some are structural and could be; many are judgment the tests cannot see). Apply them by reading the code.

## Layer mapping for this project

The source checklist assumes a `Domain / Application / Persistence / Api` layout in separate projects. This project differs — translate as you read:

- **`Persistence` → `Infrastructure`** (repository implementations, `DbContext`, EF configurations, contract implementations).
- **`Api` → `Controllers`** (thin entry points).
- Layers are **folders inside one module project**, not separate `.csproj` files (only each module's `Contracts` and `Shared.Kernel` are separate projects). So "Domain.csproj references no EF" is, here, "the `Domain` folder/namespace references no EF" — covered by the layering arch test (see Lens 1a).
- **`Contracts`** in this project is the *cross-module* public surface (its own project). An intra-module application service returns a DTO/response model; a *cross-module* contract method returns a type from the module's `Contracts` project. Either way: never a domain entity.

## On deferred infrastructure

The project **defers** domain events, integration events, message brokers, CQRS read models, and a formal Unit of Work until a concrete need forces them (see the feature-design skill's "deliberately deferred" section). So treat the source checklist's event/UoW/CQRS rules as **conditional**: review them *only if that mechanism is present*, and if it was introduced without the justifying need (a dependency cycle, a real extraction, a genuinely async process), that is itself a finding. The default cross-module mechanism is a synchronous, in-process contract call.

---

## Domain layer

### D1 — No public setters on business-relevant state
**Check:** domain state is `{ get; private set; }` or `{ get; init; }`, mutated only through intent-revealing methods. Collections are exposed `IReadOnlyList<T>`/`IReadOnlyCollection<T>` over a private backing `List<T>`.
**Violation:** `public CourseStatus Status { get; set; }`, `public List<ExperienceInstance> Experiences { get; set; }`.
**Fix:** make the setter private; add a method (`Course.Start()`) that enforces the transition; back the collection with a private field exposed read-only.

### D2 — Aggregates are created via factory methods, not public constructors
**Check:** aggregate roots expose a `public static` factory (`Course.Open(...)`, `Cartridge.Draft(...)`); constructors are `private`/`protected` (the protected one only for EF rehydration).
**Violation:** `public Course(...)` callable with `new Course(...)`, bypassing invariants.
**Fix:** private ctor + `protected Course() { }` for EF + a named factory that validates.

### D3 — Invariants enforced at creation and mutation
**Check:** factories and state-changing methods validate and throw a domain exception (or return `Result`/`Error`) when an invariant would break; there is no path to an invalid entity.
**Violation:** a factory that trusts the caller to have validated first.

### D4 — No infrastructure dependencies on entities or domain services
**Check:** no entity or domain service has a field/parameter of `DbContext`, a repository, `IUnitOfWork`, `ILogger`, `HttpClient`, or `IClock` used as ambient state.
**Violation:** `public Course(ICourseRepository repo, ...)`.

### D5 — Domain services are stateless and earn their existence
**Check:** a domain service holds no mutable state, depends only on other domain services / domain-defined interfaces, and takes/returns domain objects. It exists only when the logic spans aggregates or genuinely fits no single entity.
**Violation:** a "domain service" injecting `DbContext`/`HttpClient`/`ILogger` or returning DTOs; or a one-method service that just delegates to a factory (`CourseCreationService.Create(...) => Course.Open(...)`) — delete it.

### D6 — Domain code does not load via repositories except for genuine cross-aggregate rules
**Check:** entities never call repositories; domain services rarely do, and only for a real business rule that requires a query (e.g. a uniqueness check). Aggregates an operation needs are handed in as parameters by the application service.
**Violation:** a domain service loading aggregates by ID just to operate on them — that is the application service's job.

### D7 — Child entities do not navigate back to the aggregate root
**Check:** a child (e.g. `ExperienceInstance`) has no navigation property to its root (`Course`). The FK column exists; the inverse navigation does not. EF uses `WithOne()` with no inverse argument.
**Violation:** `public Course Course { get; private set; }` on `ExperienceInstance`.
**Rationale:** a back-reference lets callers reach the root through a child (`instance.Course.Complete(...)`), bypassing the rule that all behaviour flows through the root.

### D8 — Peer entities within an aggregate reference each other by ID
**Check:** when one child must reference another child of the same aggregate, it holds the peer's ID, not a navigation. The root coordinates lookups.
**Violation:** `public ExperienceInstance? UnlockedBy { get; private set; }`.
**Fix:** `public ExperienceInstanceId? UnlockedByExperienceId { get; private set; }`, and a lookup method on the root.

### D9 — Mutation of child state goes through the root
**Check:** state-changing methods on children are not called from outside the aggregate. The root exposes an operation that delegates inward so it can enforce invariants spanning several children (e.g. recompute Completion/Eligibility).
**Violation:** `course.Experiences[0].Complete();` in an application service.
**Fix:** `course.CompleteExperience(experienceTypeId);` — the root finds the child, mutates it, then re-evaluates aggregate-wide invariants.

### D10 — Value objects are embedded as owned types, not modelled as separate aggregates
**Check:** small value-like types with no independent identity (`DateRange`, `Money`, an address) are EF-owned (`OwnsOne`/`OwnsMany`), not entities with their own repository.
**Violation:** an `IScheduleRepository` and a `Schedule` entity with its own ID when `Schedule` only exists as part of a `Course`.

### D11 — The aggregate's object graph is a tree
**Check:** following navigations from the root never cycles; children point down to their own sub-children only — nothing sideways or upward.
**Quick test:** can the aggregate serialize to JSON without `ReferenceLoopHandling.Ignore`? If not, there is a back-reference (D7) or peer navigation (D8) that shouldn't exist.

---

## Application layer

### A1 — One application service per use case
**Check:** application services are named for use cases (`OpenCourseService`, `PublishCartridgeService`) or expose one `HandleAsync` per use case — not a kitchen-sink `CourseService` with a dozen unrelated methods.

### A2 — No business rules in application services
**Check:** no `if` in an application service enforces a domain rule. Conditionals are limited to null checks after a load (throw `NotFound`) and wiring decisions. Rules like "only a draft Cartridge can be edited" live on the entity.
**Violation:** `if (cartridge.Status == Published) throw ...;` in the service — push it into `cartridge.Edit()`.

### A3 — Application services return DTOs, never domain entities
**Check:** every public method returns a DTO/response model; cross-module methods return a type from the module's `Contracts` project. Never `Task<Course>`.

### A4 — Application services orchestrate, they don't transact ad hoc
**Check:** the shape is `load → invoke domain → persist → commit → (map to DTO)`. The service calls repositories and a single commit per use case; it does not sprinkle `SaveChanges()` calls, and ideally the `DbContext` isn't even visible at this layer. *(If no Unit of Work abstraction exists yet, that's fine during discovery — flag only ad-hoc multi-save transaction management.)*

### A5 — Application services translate primitives into value objects before calling the domain
**Check:** a command carrying `Guid CartridgeVersionId`, `decimal amount`, `string currency` is wrapped into `CartridgeVersionId`, `Money`, etc. before reaching a factory/entity.
**Violation:** `Course.Open(cmd.CartridgeVersionId /* Guid */, ...)`.

### A6 — Commands are immutable and carry only primitives or DTOs
**Check:** command types are `record`/`init`-only and contain no domain entities or value objects — they're carriers of outside input.
**Violation:** `public record OpenCourseCommand(CartridgeVersion Version);`.

### A7 — No reflection-based mapping into domain entities
**Check:** no `mapper.Map<Course>(dto)` / `dto.Adapt<Course>()` constructing an entity by reflection — it bypasses the factory and invariants. Use the factory explicitly (see A5). Source-generated mappers (Mapperly) are fine for Contract↔Contract and mechanical Domain→Contract, never anything→Domain entity.

### A8 — Read queries are separated from the write aggregate *(lightweight, not full CQRS)*
**Check:** read-only queries for display don't load write aggregates through the repository; they use a separate query type returning DTOs directly. This is the lightweight read/write split the design skill already sanctions — not a separate CQRS read model (which stays deferred).
**Rationale:** the write side protects invariants; the read side serves shapes. Mixing them bloats the aggregate.

---

## Controllers (Api)

### C1 — Controllers know nothing about domain types
**Check:** no `using ...Domain;` in a controller; signatures and bodies reference Commands, Queries, and DTOs only.

### C2 — Controllers are thin
**Check:** actions are ~3–10 lines: build a command/query, call the application service, return a result wrapping a DTO. No business logic, no repository or `DbContext` access (C4).

### C3 — No business rules in controllers
**Check:** no `if` enforcing a domain rule (ownership, state). Acceptable conditionals are result→HTTP-status mapping (prefer middleware) and model-validation short-circuits (prefer filters).
**Violation:** `if (course.LearnerId != User.GetLearnerId()) return Forbid();`.

### C4 — Controllers don't inject repositories or `DbContext`

### C5 — Auth split: identity/role at the controller, resource/state in the domain
**Check:** `[Authorize]`/roles/scopes/policies sit on the controller. "A Learner may act only on their own Course" and "this operation is valid only in this aggregate state" are enforced in the aggregate or application service. Identity is resolved from the access token (never ambient) and passed *in*.
**Violation:** a controller loading an aggregate just to check ownership before calling the service.

### C6 — Controllers pass the caller's identity into the command, not a pre-validated decision
**Check:** the controller pulls the caller identity from claims and includes it in the command (`new CancelCourseCommand(CourseId: id, RequestedBy: User.GetLearnerId())`); the aggregate method enforces the rule.

### C7 — Domain exceptions map to HTTP status codes in middleware, not `try/catch` in controllers
**Check:** central middleware maps `NotFound → 404`, domain-validation → 400, unauthorized-domain-operation → 403, concurrency → 409. Controllers contain no domain-exception `try/catch`.

---

## Infrastructure (persistence)

### I1 — Repository implementations live in `Infrastructure` and implement domain-defined interfaces
**Check:** each `ICourseRepository` (Domain/Application) has exactly one implementation in `Infrastructure`; the dependency points inward (Infrastructure → Domain), never the reverse.

### I2 — Repositories deal in aggregates, not rows, `IQueryable`, or DTOs
**Check:** repository methods return aggregate roots or `null`; loading a root loads the graph it needs to enforce invariants.
**Violation:** `IQueryable<Course> Query();` or `Task<CourseSummaryDto> GetSummaryAsync(...)` on the write repository.

### I3 — EF configuration enforces encapsulation
**Check:** configurations use backing fields and `PropertyAccessMode.Field` for read-only collections, and a `protected` constructor for rehydration. If EF couldn't load the entity so someone added a *public* parameterless ctor, that's a violation — use protected.

### I4 — Persistence concerns don't leak into the domain
**Check:** no `[Table]`, `[Column]`, `[Key]`, `[ForeignKey]` attributes on domain entities; mapping lives in `Infrastructure` Fluent API configurations. *(This is the attribute-level companion to the Lens 1a layering rule.)*

### I5 — Transaction commit (and any event dispatch) is centralized
**Check:** if a Unit of Work exists, it is the single place that calls `SaveChangesAsync()` and dispatches in-process domain events; application services commit once per use case. *(UoW and domain-event dispatch are deferred until needed — review this only if present.)*

---

## Cross-cutting

### X1 — Time is injected, not read from `DateTime.UtcNow` in the domain
**Check:** domain code needing the current time receives it as a parameter (preferred) or via an `IClock` abstraction defined in the domain. Direct `DateTime.UtcNow`/`DateTime.Now` in domain code is flagged.
**Rationale:** determinism and testability of time-sensitive rules (Eligibility windows, Completion deadlines).

### X2 — Aggregate IDs are generated in the domain, not by the database
**Check:** IDs come from the domain (`CourseId.New()`); the database stores but does not assign them. Reliance on identity/auto-increment columns for an aggregate's primary key is a violation. This also underpins idempotent writes (the client can supply the ID and retry safely).

### X3 — Domain events, if used, are raised inside the aggregate
**Check:** *(only if domain events are present — they're deferred by default)* the event is added to the aggregate's internal list inside the method that causes it, not by the application service. The application service publishes any cross-module/integration event after commit — and cross-module eventing should appear only to break a cycle or support an extraction.

### X4 — No logging in the domain
**Check:** no `ILogger<T>` in entities or domain services; the domain expresses outcomes by returning values or throwing. Logging sits in the application/infrastructure layers, around the domain calls.

---

## Test coverage

### T1 — Domain logic is tested without infrastructure (no `DbContext`, host, or HTTP client; construct entities directly).
### T2 — Application services are tested with fakes/mocks for repositories, contracts, and any infrastructure; assert the right domain method ran and the right DTO came back.
### T3 — Controllers are exercised through integration tests against the HTTP boundary (`WebApplicationFactory`), asserting status codes and DTO payloads — not unit-tested in isolation.

---

## Quick-scan heuristics

When skimming a PR or file, these are high-signal smells:

| Smell | Likely problem |
|---|---|
| `public Course(...)` (public ctor on an aggregate root) | Bypasses invariants — private ctor + factory. |
| `course.Status = ...;` outside the aggregate | State mutation outside the domain — use a method. |
| `using ...Modules.X.Domain;` outside module X | Cross-module boundary violation (Lens 1a). |
| `if (...) throw ...` in an application service or controller | Business rule in the wrong layer. |
| `_mapper.Map<Course>(...)` | Auto-mapping into the domain — use the factory. |
| Repository or `DbContext` injected into a controller | Bypasses the application layer. |
| `DbContext` referenced outside `Infrastructure` | Persistence leaking. |
| `[Table]`/`[Column]`/`[Key]` on a domain entity | EF attributes leaking into the domain. |
| Domain service with one method delegating to a factory | Ceremony with no value — delete it. |
| `IRepository<T>` generic interface | Loses domain expressiveness — make it aggregate-specific. |
| Application service returning a domain entity | Domain leaking past the boundary. |
| `DateTime.UtcNow` in domain code | Untestable time dependency — inject it. |
| DB-generated identity for an aggregate's PK | Persistence dictating the domain. |
| Navigation to another aggregate root (`public Cartridge Cartridge`) | Cross-aggregate reference — use the ID. |
| Navigation from child back to root (`ExperienceInstance.Course`) | Back-reference — children don't navigate up. |
| Navigation between peer children | Lateral navigation — use an ID, let the root coordinate. |
| `public List<T>` on an aggregate | Mutable collection exposed — private field + `IReadOnlyList<T>`. |
| `course.Experiences[0].DoSomething()` outside the aggregate | Reaching into a child — add a method on the root. |
| Need `ReferenceLoopHandling` to serialize an aggregate | Object graph isn't a tree — a back-reference or peer nav. |

---

## Layering cheat sheet

The **only** layer that should own each concern (anything elsewhere is a smell). Layers here are folders within the module; `Contracts` and `Shared.Kernel` are separate projects.

| Concern | Domain | Application | Infrastructure | Controllers |
|---|---|---|---|---|
| Entity / value-object definition | ✅ | | | |
| Factory methods, invariant enforcement | ✅ | | | |
| Domain services | ✅ | | | |
| Repository interface | ✅ | | | |
| Domain events (raise) — *if used* | ✅ | | | |
| Use-case orchestration | | ✅ | | |
| Commands / DTOs / mapping (domain → DTO) | | ✅ | | |
| Cross-module contract call | | ✅ | | |
| Transaction commit | | ✅ (intent) | ✅ (impl) | |
| Repository implementation, EF config, migrations | | | ✅ | |
| Contract-interface implementation | | | ✅ | |
| HTTP routing, request/response shape | | | | ✅ |
| Auth (identity/role/scope) | | | | ✅ |
| Auth (resource/state) | ✅ | ✅ | | |
| Exception → HTTP status mapping | | | | ✅ (middleware) |

Cross-module surface (interfaces + DTOs) lives in the module's `Contracts` **project**, implemented in `Infrastructure`. Universal primitives (`Result`/`Error`, ID base types, `Money`, `DateRange`) live in `Shared.Kernel`.
