---
name: efcore-conventions
description: >
  Enforces Entity Framework Core best practices, conventions, and common-gotcha
  avoidance when writing, generating, scaffolding, reviewing, or refactoring
  C# / .NET data-access code. Use whenever the user works with EF Core:
  DbContext, DbSet, entities, migrations, LINQ queries, change tracking, saving
  data, or persisting domain models. Also covers Domain-Driven Design (DDD)
  persistence patterns when the project follows DDD. Triggers on keywords such
  as "Entity Framework", "EF Core", "DbContext", "DbSet", "migration",
  "LINQ to Entities", "Add/AddAsync", "SaveChanges", "AsNoTracking", "Include",
  "tracking", "aggregate", "value object", "repository", or any EF Core
  persistence code.
---

# EF Core Conventions

Apply these to all Entity Framework Core code. Sections 1 (Core Conventions) and
2 (Gotchas) apply unconditionally. When the project follows Domain-Driven Design
(aggregates, value objects, repositories over domain models), also read
`references/ddd-patterns.md` — do not impose private backing fields, aggregate
repositories, or shadow properties on simple CRUD code. When generating code,
follow the applicable rules without being asked; when reviewing, flag
violations using the anti-patterns checklist at the end.

---

## Section 1 — Core Conventions (always apply)

### 1.1 Prefer `Add` over `AddAsync`

Use synchronous `Add` / `AddRange` by default. Only use `AddAsync` /
`AddRangeAsync` when the entity's key uses a `HiLo` (or other
database-dependent) value generator.

**Rationale:** `Add` performs no I/O — it only marks the entity as `Added` in
the in-memory change tracker. No database round-trip happens until `SaveChanges`
/ `SaveChangesAsync`. `AddAsync` exists solely so value generators like `HiLo`
can query the database asynchronously to reserve key values. Without `HiLo`,
`AddAsync` adds `Task` overhead for no benefit.

```csharp
// Do — identity / auto-increment / app-generated keys (Guid, etc.)
var entry = context.Customers.Add(customer).Entity;

// Don't — unnecessary async; entity does not use a HiLo key generator
await context.Customers.AddAsync(customer);

// Exception — entity key is configured with a HiLo sequence generator
await context.Orders.AddAsync(order);
```

This applies only to `Add`. Keep using `SaveChangesAsync`, `ToListAsync`,
`FindAsync`, etc. — those perform real database I/O. When wrapping `Add` in a
repository/service method, return the entity rather than discarding it (see 1.11).

### 1.2 Use async for all real database I/O

Any operation that hits the database must be awaited: `SaveChangesAsync`,
`ToListAsync`, `FirstOrDefaultAsync`, `SingleOrDefaultAsync`, `AnyAsync`,
`CountAsync`, `FindAsync`. Synchronous calls block a thread for the round-trip.

```csharp
// Do
var user = await context.Users.FirstOrDefaultAsync(u => u.Id == id);

// Don't
var user = context.Users.FirstOrDefault(u => u.Id == id);
```

### 1.3 Use `AsNoTracking` for read-only queries

By default EF Core tracks every returned entity, caching original values and
spending CPU on snapshot comparisons at `SaveChanges`. For data that is only
displayed or processed (never modified in the same context), append
`.AsNoTracking()`, or set `QueryTrackingBehavior.NoTracking` as the context
default. This skips change-tracking overhead entirely.

```csharp
// Do — read-only lookup
var activeUsers = await context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .ToListAsync();
```

### 1.4 Push filtering, sorting, and paging to the database

Compose the full LINQ query before materializing. Never pull a whole table into
memory and filter with LINQ-to-Objects. Apply `Where`, `OrderBy`, `Skip`/`Take`
on the `IQueryable` so they translate to SQL.

### 1.5 Scope `DbContext` correctly

`DbContext` is not thread-safe; it holds an internal change-tracking state
machine and a single transaction context. In ASP.NET Core register it Scoped
(the `AddDbContext` default) — one context per request. Never register it as a
Singleton, never share it across threads, never run concurrent operations on a
single instance. For background workers, parallel loops, or Blazor Server
components, inject `IDbContextFactory<TContext>` to create isolated, short-lived
instances on demand.

```csharp
// Do — scoped registration
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString));

// Do — factory for background/parallel/Blazor Server work
builder.Services.AddDbContextFactory<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString));

// Don't — static/shared context (not thread-safe, leaks)
public static readonly AppDbContext Db = new();
```

### 1.6 Use `AddDbContextPool` in high-throughput apps

Allocating context instances and setting up the pipeline has runtime cost. In
high-throughput services, `AddDbContextPool<TContext>` pools and recycles
context instances to cut allocation overhead. Note pooled contexts must not hold
per-request mutable state in fields.

```csharp
// Do — pooled contexts for high request volume
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString));
```

### 1.7 Batch saves; treat `DbContext` as the unit of work

Make all changes, then save once. `SaveChangesAsync` inside a loop produces one
round-trip per iteration. The `DbContext` is itself a Unit of Work: run business
logic across repositories/services and call `SaveChangesAsync` exactly once at
the end of the operation so multi-entity updates commit atomically.

```csharp
// Do
foreach (var item in items)
    context.Items.Add(item);
await context.SaveChangesAsync();

// Don't — round-trip every iteration
foreach (var item in items)
{
    context.Items.Add(item);
    await context.SaveChangesAsync();
}
```

### 1.8 Flow a `CancellationToken` through async data access

Async EF Core methods accept a `CancellationToken`. Pass the request/operation
token through so queries cancel when the caller goes away.

```csharp
// Do
public async Task<User?> GetAsync(int id, CancellationToken ct) =>
    await context.Users.FirstOrDefaultAsync(u => u.Id == id, ct);
```

**Placement:** put the `CancellationToken` as the **last required parameter — before any
optional parameters**, and keep it required (no `= default`). Do *not* reflexively move it
to the very end of the signature: C# forbids a required parameter after an optional one, so
placing `ct` last *forces* it to become optional (`CancellationToken ct = default`), which
lets callers silently drop the token and lose cancellation. A required parameter is allowed
to precede an optional one, so "last required, before the optionals" keeps the compiler
guarantee that every caller passes a token.

```csharp
// Do — ct is the last required parameter, sitting before the optional `tracked`
Task<Cartridge?> GetByIdAsync(CartridgeId id, CancellationToken ct, bool tracked = false);

// Don't — ct shoved to the end is forced optional; callers can omit it and lose cancellation
Task<Cartridge?> GetByIdAsync(CartridgeId id, bool tracked = false, CancellationToken ct = default);
```

### 1.9 Configure the model explicitly with the Fluent API

Prefer `IEntityTypeConfiguration<T>` classes over scattered data annotations,
especially for keys, relationships, indexes, precision, and constraints. Keep
configuration close to the model and testable.

```csharp
// Do
public class OrderConfig : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.HasIndex(o => o.CustomerId);
        builder.Property(o => o.Total).HasPrecision(18, 2);
        builder.HasMany(o => o.Lines).WithOne(l => l.Order);
    }
}
```

### 1.10 Migrations: review before applying, never edit applied migrations

Generate migrations with descriptive names, inspect the generated `Up`/`Down`
before applying, and keep them in source control. Once a migration is applied to
a shared environment, never edit it — add a new migration instead.

```bash
# Do
dotnet ef migrations add AddOrderStatusColumn
# review the generated file, then:
dotnet ef database update
```

### 1.11 Return the entity from add/update methods

Have repository/service add and update methods return the affected entity rather
than `void`. It enables fluent call sites and gives the caller a handle to the
tracked instance. `Add` returns an `EntityEntry<T>`; expose its `.Entity`.

**Important — what this does and does not give you.** `Add(order).Entity` is the
*same* instance you passed in, so returning it adds convenience, not new data at
that moment. Database-generated values (identity/auto-increment keys, computed
columns, `RowVersion`) are populated only after `SaveChangesAsync` runs — and EF
Core writes them back onto that same tracked reference. So the returned object is
fully populated only once the unit of work has been saved (see 1.7). Do not lead
callers to expect a generated `Id` immediately after `Add`. The save belongs in
the caller, never in the add/update method itself (in DDD projects, see the
save-placement rule in `references/ddd-patterns.md` §5).

```csharp
// Do — return the tracked entity; caller saves, then reads generated values
public Order Add(Order order) => _context.Orders.Add(order).Entity;

public Order Update(Order order) => _context.Orders.Update(order).Entity;

// Caller (application-layer handler): generated key is available after save
var added = _orderRepository.Add(order);   // registers intent only
await _unitOfWork.SaveChangesAsync(ct);    // single commit per use case
var newId = added.Id; // populated now, not before SaveChangesAsync
```

Note the `Update` example follows the standard `Update` semantics — see the
disconnected-overwrite gotcha (2.3) before using `Update` on detached payloads.

---

## Section 2 — Critical Gotchas (always apply)

### 2.1 The N+1 query problem

One query fetches the parents; then a query fires per parent to load children
inside a loop (often silently, via lazy-loading proxies).

```csharp
// Gotcha — 1 query for customers, then N queries inside the loop
var customers = context.Customers.ToList();
foreach (var customer in customers)
    Console.WriteLine(customer.Orders.Count);
```

**Fix:** eager-load with `.Include(c => c.Orders)`, or project into a DTO so the
work flattens into a single query.

### 2.2 Cartesian product explosion

Multiple parallel `.Include()` of collection navigations generate a cross-product
of JOINs, multiplying returned rows.

```csharp
// Gotcha — massive row multiplication over the wire
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Subscribers)
    .ToListAsync();
```

**Fix:** append `.AsSplitQuery()` so EF Core issues separate SQL commands and
stitches the relationships back together in memory.

```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Subscribers)
    .AsSplitQuery()
    .ToListAsync();
```

### 2.3 Disconnected-entity overwrites with `Update()`

Calling `context.Update(entity)` on a detached payload marks **every** property
as modified. The generated SQL overwrites all columns — including ones the
caller never set — which can wipe values with defaults/nulls.

**Fix:** load the tracked instance first, copy only the changed fields onto it,
and let the change tracker emit an update for just the modified columns.

```csharp
// Do
var existing = await context.Orders.FirstOrDefaultAsync(o => o.Id == dto.Id);
if (existing is null) return;
existing.Status = dto.Status;       // assign only what changed
existing.Notes  = dto.Notes;
await context.SaveChangesAsync();    // updates only modified columns

// Don't — overwrites every column from a detached payload
context.Update(incomingOrder);
await context.SaveChangesAsync();
```

---

## Section 3 — Domain-Driven Design Patterns (conditional)

When the project follows DDD — aggregate roots, value objects, repositories
wrapping domain models — read `references/ddd-patterns.md` for the persistence
patterns that reconcile DDD encapsulation with EF Core mapping: private backing
fields for aggregate children, value-object mapping (Complex Types vs Owned
Entities), shadow properties for infrastructure metadata,
repository-per-aggregate design, and `SaveChangesAsync` placement (repositories
register intent, the application layer commits). Do not apply those patterns to
simple CRUD models.

---

## Anti-patterns to reject on review

- `AddAsync` where the key is not `HiLo`-generated (use `Add`).
- Synchronous DB calls (`ToList`, `First`, `SaveChanges`) in async code paths.
- Missing `AsNoTracking` on read-only queries.
- Lazy loading or per-iteration queries inside loops (N+1).
- `ToListAsync()` followed by client-side `Where`/`Select`.
- Multiple collection `Include`s without `AsSplitQuery` (Cartesian explosion).
- `context.Update(detachedEntity)` that overwrites every column.
- Shared, static, or singleton `DbContext`; concurrent use of one instance.
- Plain `AddDbContext` in a high-throughput path where pooling fits.
- `SaveChangesAsync` called inside a loop or inside a repository.
- Missing `CancellationToken` on async data-access methods.
- `CancellationToken ct = default` made optional only because it was placed after an optional
  parameter — put it as the last *required* parameter instead, before the optionals (see §1.8).
- Editing a migration already applied to a shared database.
- (DDD) Public mutable child collections on aggregates; repositories per table;
  repository methods returning `IQueryable<T>`; infrastructure metadata leaking
  into domain classes; `SaveChangesAsync` inside a repository instead of the
  handler; a single transaction spanning multiple modules. See
  `references/ddd-patterns.md`.
