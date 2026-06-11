# Clean code and idiomatic C# (for this codebase)

Design practices for feature code in this modular monolith. These complement the boundary rules: the boundary rules keep modules apart; these keep the code inside a module clean and the domain model honest.

## Strongly typed IDs

Represent every identifier as a dedicated type per concept, never a bare `Guid`, `string`, or `int`.

```csharp
public readonly record struct CartridgeVersionId(Guid Value);
public readonly record struct CourseId(Guid Value);
```

Bare primitives let the compiler accept a swapped argument (`OpenCourse(cartridgeVersionId, learnerId)` compiles and ships even if the arguments are reversed). Strongly typed IDs make that a build error, moving a class of bug from runtime to build time. In this architecture the ID also *is* the cross-module reference, a named pointer to a concept another module owns, so it documents relationships in code.

`readonly record struct` gives value equality, no heap allocation, and immutability. Use a source generator (StronglyTypedId by Andrew Lock, or Vogen if you also want guarded value objects with validation) to eliminate the EF/JSON/model-binding boilerplate. Adopt from the first entity; retrofitting touches every signature, query, and DTO.

## Rich domain, not anemic models

Put invariants and behavior in the domain entities and value objects, not in services operating on data bags. A service that mutates public setters on a struct-like entity is a smell; the entity should expose intent-revealing methods that protect its own invariants.

```csharp
// Prefer: behavior and invariants live in the entity
public sealed class Nudge
{
    public NudgeStatus Status { get; private set; }

    public void Complete()
    {
        if (Status is not NudgeStatus.Pending)
            throw new DomainException("Only a pending Nudge can be completed.");
        Status = NudgeStatus.Completed;
    }
}

// Avoid: anemic entity with public setters, logic stranded in a service
```

## Make illegal states unrepresentable

Use constructors, factory methods, and value objects so an invalid object cannot be created. Validate at the boundary of construction, then trust the type thereafter. Value objects (`EmailAddress`, `Money`, `DateRange`) encapsulate their own validation and remove primitive obsession from the domain.

```csharp
public readonly record struct EmailAddress
{
    public string Value { get; }
    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email address.", nameof(value));
        Value = value;
    }
}
```

## Domain-Driven Design practices

DDD is the backbone of this architecture: a **bounded context maps to a module**, and the boundary rules are DDD's context boundaries made enforceable. The practices below are the tactical and strategic patterns that keep each module's model coherent.

### Ubiquitous language

Within a module, name types, methods, and database columns after the language the domain experts actually use, and use that same language consistently in code, tests, and conversation. If experts say "complete an Experience," the code says `Experience.Complete()`, not `LearningUnit.SetStatusDone()`. The payoff is direct: when the same word means two different things to two groups of stakeholders, or two words mean the same thing, you have likely found a context boundary. Diverging language between two areas of one module is early evidence that the module is really two and should split (the coarse-module discipline). Treat a name you have to qualify ("the ExperienceType the Cartridge defines, not the ExperienceInstance the Learner runs") as a seam.

### Entities vs value objects

Decide deliberately which of the two each concept is, because it drives identity, equality, and mutability:

- **Entity** — has a distinct identity that persists through change; equality is by ID, not by attributes. Two Cartridges with the same title are different Cartridges. Model as a `class` with a strongly typed ID, state changed only through intent-revealing methods.
- **Value object** — defined entirely by its attributes; no identity; immutable; equality by value. `Money`, `DateRange`, `EmailAddress`, `Address`. Two `Money(10, "USD")` are interchangeable. Model as a `readonly record struct` or `record`, validated at construction.

Most primitive obsession is a missing value object. When you see a `decimal amount` paired with a `string currency` threaded through methods, that is a `Money` value object waiting to be extracted.

### Aggregates and aggregate roots

An **aggregate** is a cluster of entities and value objects treated as a single consistency boundary, with one entity designated the **aggregate root**. This is the most important tactical pattern and the one most often skipped. The rules:

- **All external access goes through the root.** Other code holds a reference to the `Course` root, never directly to an `ExperienceInstance` inside it. The root enforces the aggregate's invariants.
- **Reference other aggregates by ID, not by object.** A `Course` holds a `LearnerId`, not a `Learner`. This is the same rule as cross-module ID references, applied within a module: it keeps aggregates as independent consistency units and stops the object graph from sprawling into one tangled whole.
- **One transaction, one aggregate.** A single use case modifies one aggregate and saves it atomically. When a change must span aggregates, that is eventual consistency between them (a domain event), not one giant transaction. This keeps transactions small and aggregates genuinely independent.
- **Keep aggregates small.** A large aggregate becomes a contention and loading-cost problem. Prefer several small aggregates linked by ID over one that owns everything. If you are unsure whether something belongs inside the aggregate, the test is: must it be consistent with the root *at all times, within the same transaction*? If not, it is its own aggregate, referenced by ID.

```csharp
public sealed class Course   // aggregate root
{
    private readonly List<ExperienceInstance> _experiences = [];
    public CourseId Id { get; }
    public LearnerId LearnerId { get; }                   // another aggregate / external identity, by ID
    public CartridgeVersionId CartridgeVersionId { get; }  // the published source, by ID
    public IReadOnlyList<ExperienceInstance> Experiences => _experiences.AsReadOnly();

    public void CompleteExperience(ExperienceTypeId experienceType)
    {
        var experience = _experiences.SingleOrDefault(e => e.IsOfType(experienceType))
            ?? throw new DomainException("That Experience is not part of this Course.");
        experience.Complete();   // invariant enforced by the root
    }
}
```

The aggregate boundary is also the natural unit a repository loads and saves, and it usually maps to a module's per-schema table cluster.

#### Navigation discipline inside an aggregate

The object graph rooted at the aggregate root must be a **tree**: navigations point downward from the root to its children only — never sideways between peers, never back up to the root. This is what keeps "all behaviour flows through the root" true in practice.

- **Children do not navigate back to the root.** An `ExperienceInstance` holds no `Course` navigation property. The FK column exists in the database; the inverse navigation does not (configure EF with `WithOne()` and no inverse argument). A back-reference lets a caller reach the root through a child (`instance.Course.Complete(...)`) and bypass the root's invariants.
- **Peer children reference each other by ID, not by navigation.** When one child must point at another in the same aggregate (an Experience unlocked by completing another), it holds the peer's ID (`ExperienceInstanceId? UnlockedByExperienceId`), and the root provides any lookup. A direct `ExperienceInstance UnlockedBy` navigation is a lateral edge that breaks the tree.
- **Mutate children through the root.** Don't call a child's state-changing method from outside; expose an operation on the root that delegates inward, so the root can enforce invariants spanning several children:

```csharp
// On the root — finds the child, mutates it, then re-checks aggregate-wide state
public void CompleteExperience(ExperienceTypeId experienceType)
{
    var experience = _experiences.SingleOrDefault(e => e.IsOfType(experienceType))
        ?? throw new DomainException("That Experience is not part of this Course.");
    experience.Complete();
    RecalculateEligibility();   // invariant that spans children lives on the root
}

// Avoid: course.Experiences[0].Complete();  — reaching into a child from an application service
```

Quick self-check: if you cannot serialize the aggregate to JSON without `ReferenceLoopHandling.Ignore`, the graph has a back-reference or peer navigation that shouldn't be there.

### Repositories per aggregate

Provide one repository per aggregate root, expressed as an interface owned by the domain/application layer and implemented in `Infrastructure`. The repository deals in whole aggregates (load the root with the data needed to enforce its invariants; save the root), not in arbitrary partial queries. Read-side queries that cross aggregates or need projections do not belong on the repository; use a dedicated query (or, only when justified, a read model) rather than bending the repository into a general DAO.

```csharp
// Domain or Application — the abstraction
public interface ICourseRepository
{
    Task<Course?> GetByIdAsync(CourseId id, CancellationToken ct);
    Task AddAsync(Course course, CancellationToken ct);
    // no GetAll, no arbitrary filters; this deals in the aggregate
}
```

### Domain services

When a piece of domain logic does not naturally belong to a single entity or value object, because it coordinates several aggregates or expresses a domain operation that is inherently between things, put it in a **domain service**: a stateless type, named in the ubiquitous language, living in the domain layer. Use this sparingly. Logic in a domain service that *could* live on an entity usually should; the domain service is for the genuine in-between operations, not a dumping ground that recreates the anemic-model smell.

### Domain events as a tactical pattern

A **domain event** records something meaningful that happened in the domain ("ExperienceCompleted"), expressed in past tense in the ubiquitous language. Within a module they let an aggregate announce a fact without knowing who reacts, which decouples a use case from its side effects and keeps each transaction focused on one aggregate. Note the boundary rule still holds: cross-*module* eventing is deferred until a dependency cycle or extraction forces it. Within a module, lightweight domain events are a reasonable way to keep aggregates decoupled, but do not over-reach for them during discovery, a direct method call is fine until the decoupling earns its keep.

### Factories for complex creation

When constructing a valid aggregate involves more than a constructor can cleanly express (invariants spanning several inputs, creation of contained entities), use a static factory method on the aggregate root (`Course.Open(...)`) or a dedicated factory. The goal is unchanged: it must be impossible to obtain an invalid aggregate.

### Strategic note: the module is the bounded context

Keep one model per context. Resist a single shared model of "the Experience" across modules; each module models only what it needs (the entity/own-concept distinction — Cartridges owns the authored `ExperienceType`, LearningEngagement owns the Learner's `ExperienceInstance`). When two contexts must relate, that relationship is a deliberate mapping at the contract boundary (a DTO translating the owner's model into terms the consumer understands), which is exactly what the synchronous-contract pattern provides. This is anti-corruption-layer thinking: a consumer never absorbs another context's model wholesale; it translates at the edge so its own model stays clean.

## Error handling across boundaries

For expected failures (not found, validation, conflict), prefer a `Result`/`Error` type returned from application services and contract methods over throwing, and over returning bare `null` that callers may forget to check. Reserve exceptions for truly exceptional or programmer-error cases. Never let an internal exception type leak across a module boundary; map to a contract-level result or a documented contract exception.

```csharp
public async Task<Result<CourseId>> Handle(OpenCourseCommand cmd, CancellationToken ct)
{
    if (!await _catalog.VersionExistsAsync(cmd.CartridgeVersionId, ct))
        return Result.Failure<CourseId>(Error.NotFound("CartridgeVersion", cmd.CartridgeVersionId.Value));
    // ...happy path
}
```

## Application services (use-case orchestration)

The application layer orchestrates a use case; it holds no business rules. Keep each service focused on one use case (`OpenCourseService.HandleAsync`, not a kitchen-sink `CourseService` with a dozen methods), and keep the method to this shape:

```
load aggregate(s) → invoke domain behaviour → persist → commit → map to a DTO
```

Concretely:

- **No business rules here.** The only conditionals are null checks after a load (`?? return Error.NotFound(...)`) and wiring decisions. A rule like "only a draft Cartridge can be edited" lives on the entity (`cartridge.Edit()` throws), not as an `if` in the service.
- **Translate primitives into value objects before calling the domain.** A command carries `Guid`/`decimal`/`string` from the outside world; the service wraps them (`new CartridgeVersionId(cmd.VersionId)`, `Money.Of(cmd.Amount, cmd.Currency)`) before passing them to a factory or method.
- **Return a DTO, never a domain entity.** Intra-module callers get an internal response type; cross-module callers get a type from the module's `Contracts` project.
- **Never reflection-map into a domain entity.** `mapper.Map<Course>(cmd)` bypasses the factory and its invariants — construct through the factory explicitly. Source-generated mappers (Mapperly) are fine for Contract↔Contract and mechanical Domain→Contract, never anything→Domain entity.
- **Commands are immutable carriers.** `record` types holding primitives or DTOs, no domain entities or value objects.
- **Read vs write.** Display queries don't load write aggregates through the repository; use a separate query type returning DTOs. This is a lightweight read/write split, not a CQRS read model (which stays deferred until justified).

## Controllers and identity

Controllers are thin entry points: build a command/query from the request, call the application service, return a result wrapping a DTO. No business logic, no repository or `DbContext` injection, no `using ...Domain;`.

Identity is resolved **from the access token** and passed *into* the command — never read as ambient state, and never used by the controller to pre-validate a domain decision:

```csharp
// Controller: extract the caller from claims, hand it to the command
var command = new CancelCourseCommand(CourseId: id, RequestedBy: User.GetLearnerId());
await _cancelCourse.HandleAsync(command, ct);
```

Split authorization by kind: identity/role/scope checks (`[Authorize(...)]`) belong on the controller; "a Learner may act only on their own Course" and "this operation is valid only in this aggregate state" are domain rules enforced in the aggregate or application service (`course.CancelBy(requesterId)`). A controller that loads an aggregate just to check ownership has put a domain rule in the wrong layer. Map domain exceptions to HTTP status codes in central middleware (`NotFound → 404`, validation → 400, unauthorized-operation → 403, concurrency → 409), not in per-action `try/catch`.

## Cross-cutting: time, identifiers, logging

- **Inject time; don't read the clock in the domain.** Domain code that needs "now" takes it as a parameter (preferred) or via an `IClock` abstraction defined in the domain — never `DateTime.UtcNow` inline. Eligibility windows and Completion deadlines must be deterministic under test.
- **Generate IDs in the domain, not the database.** `CourseId.New()` mints the identity; the database stores but never assigns it. This keeps writes idempotent (the caller can supply the ID and safely retry) and avoids the persistence layer dictating the model.
- **No logging in the domain.** No `ILogger<T>` in entities or domain services; the domain reports outcomes by returning values or throwing. Logging wraps the domain call in the application/infrastructure layers.

## Dependencies and DI

Depend on abstractions, not concretions: inject interfaces, register implementations in the module's registration method. Implementations are `internal sealed`; the interface they implement is `public` only when it is genuine contract surface, otherwise it too is `internal`. Constructor injection only; avoid service-locator patterns. Keep constructors free of work, just assignment.

## Naming and method shape

Name types and methods by intent (`CompleteExperience`, not `Process`). Keep methods small and single-purpose; a method that needs sections with comment headers usually wants to be several methods. Keep the cyclomatic load low, prefer guard clauses and early returns over deep nesting. One level of abstraction per method.

## Async and cancellation

Use `async`/`await` end to end for I/O; do not block on async (`.Result`, `.Wait()`). Flow a `CancellationToken` through every async method, including contract methods, and honor it. Suffix async methods with `Async`.

## Immutability and records

Prefer immutable types where practical: `record` for DTOs and value objects, `private set` or init-only on entity state changed through methods. DTOs in `Contracts` should be immutable `record` types carrying data only, no behavior, no dependency on `Domain`.

## Testing

Unit-test domain logic directly against entities and value objects, no mocks needed for pure domain. Test application services against contract interfaces, mocking other modules' contracts. Because cross-module dependencies are interfaces returning DTOs, a module's tests never need another module's internals, which is the same property that makes extraction cheap. The architecture tests (NetArchTest) cover the structural rules; feature tests cover behavior.

## Keep it consistent with the boundary rules

Clean code here always serves the same end as the boundary rules: a module that is internally clean and externally minimal (small contract surface, everything else `internal`) is both pleasant to work in and cheap to move. When a clean-code instinct (e.g., "extract this shared helper") would place domain logic in `Shared/Kernel`, stop, that is the coupling trap; shared *behavior* across contexts usually means the concept is modeled in the wrong place, not that it belongs in the Kernel.
