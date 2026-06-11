# Domain-Driven Design Persistence Patterns for EF Core

These patterns reconcile DDD's encapsulation with EF Core's mapping needs.
Apply them ONLY when the project follows Domain-Driven Design — do not impose
them on simple CRUD models.

## 1. Aggregate encapsulation via private backing fields

An Aggregate Root controls its children; external code must mutate them only
through validation-backed domain methods, never the collection directly. Use a
private backing field exposed as a read-only collection, and map EF Core to the
field.

```csharp
public class Order // Aggregate Root
{
    public Guid Id { get; private set; }
    public DateTime OrderDate { get; private set; }

    private readonly List<OrderItem> _orderItems = new();
    public IReadOnlyCollection<OrderItem> OrderItems => _orderItems.AsReadOnly();

    private Order() { } // EF Core materialization

    public Order(Guid id)
    {
        Id = id;
        OrderDate = DateTime.UtcNow;
    }

    public void AddProduct(Guid productId, decimal price, int quantity)
    {
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive.");
        _orderItems.Add(new OrderItem(productId, price, quantity));
    }
}
```

```csharp
builder.Entity<Order>(b =>
{
    b.HasKey(o => o.Id);
    b.Navigation(o => o.OrderItems)
     .HasField("_orderItems")
     .UsePropertyAccessMode(PropertyAccessMode.Field);
});
```

## 2. Map value objects cleanly

Value objects are immutable and identity-less (e.g. `Address`, `Money`).

- **EF Core 8+:** prefer Complex Types (`ComplexProperty`). Fields embed inline
  in the owner's table, with no foreign/shadow keys and no tracking overhead.
- **EF Core 7 and earlier:** use Owned Entities (`OwnsOne`). Functional but
  modeled as a pseudo-entity with shadow keys that can hurt performance.

```csharp
public record Address(string Street, string City, string ZipCode);

public class Customer
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public Address ShippingAddress { get; private set; }
}
```

```csharp
builder.Entity<Customer>(b =>
{
    b.HasKey(c => c.Id);
    b.ComplexProperty(c => c.ShippingAddress); // flattens into Customer table
});
```

## 3. Keep infrastructure metadata in shadow properties

Technical fields (`CreatedOn`, `LastModifiedBy`, `RowVersion`) don't belong on
pure domain classes. Configure them as shadow properties — they live in the
change tracker and database only — and populate them by overriding
`SaveChangesAsync`.

```csharp
builder.Entity<Order>().Property<DateTime>("CreatedOn");
builder.Entity<Order>().Property<byte[]>("RowVersion").IsRowVersion();
```

```csharp
public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    foreach (var entry in ChangeTracker.Entries().Where(e => e.State == EntityState.Added))
        entry.Property("CreatedOn").CurrentValue = DateTime.UtcNow;

    return base.SaveChangesAsync(cancellationToken);
}
```

## 4. Define repositories per aggregate root, never leak `IQueryable<T>`

Build one repository per aggregate root, not per table. It acts as an in-memory
collection facade returning complete, consistent aggregates. Return materialized
results (`Task<Order?>`), never `IQueryable<T>` — exposing the query lets callers
inject joins, break tracking boundaries, or bypass invariants. Do not call
`SaveChangesAsync` inside repositories; commit once in the application-layer
handler (see §5 below and SKILL.md §1.7).

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Order Add(Order order);
}

public class OrderRepository : IOrderRepository
{
    private readonly ApplicationDbContext _context;
    public OrderRepository(ApplicationDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await _context.Orders
            .Include(o => o.OrderItems)   // pull the full aggregate boundary
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    // Synchronous Add: Order has an app-generated Guid key, not HiLo (see SKILL.md §1.1)
    // Returns the tracked entity (see SKILL.md §1.11); commit once in the handler (see §5)
    public Order Add(Order order) => _context.Orders.Add(order).Entity;
}
```

## 5. `SaveChangesAsync` placement: repositories register intent, the application layer commits

Never call `SaveChangesAsync` inside repository `Add`/`Update`/`Remove` methods.
Repositories only attach/track entities; the application layer owns the commit
boundary. This is the DDD elaboration of the unit-of-work rule in SKILL.md §1.7.

**Repository layer**
- `Add`, `Update`, `Remove` only attach or track entities via the `DbContext`.
- Do not call `SaveChangesAsync` inside them.
- `Update` is often unnecessary for entities already loaded by EF Core — they
  are change-tracked, so mutating them is enough (see also SKILL.md §2.3).

**Application layer (command handlers)**
- Commit exactly once per use case via an injected `IUnitOfWork.SaveChangesAsync`.
- Treat the `DbContext` as the Unit of Work, hidden behind an `IUnitOfWork`
  abstraction.
- Prefer centralizing the commit in a pipeline behavior (e.g. a MediatR
  `TransactionBehavior`) so handlers don't repeat the call.

**Rationale**
- *Atomicity:* one use case may touch multiple aggregates/repositories — a
  single transaction keeps it all-or-nothing.
- *Performance:* one round-trip per command instead of N.
- *Domain events / outbox:* a single `SaveChangesAsync` is the natural dispatch
  point, within the same transaction.
- *Separation of concerns:* repositories express *what* changed; the application
  layer decides *when* to persist.

**Modular monolith boundaries**
- Each module gets its own `DbContext` / Unit of Work, even when sharing one
  physical database.
- Do not span a single transaction across modules. Use domain/integration
  events for cross-module consistency — this preserves module boundaries and
  eases a future extraction into separate services.

```csharp
// Repository — registers intent only, no save (returns the entity, see SKILL.md §1.11)
public Order Add(Order order) => _context.Orders.Add(order).Entity;

// Handler — single commit per use case
public async Task Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Create(cmd.CustomerId, cmd.Items);
    _orderRepository.Add(order);
    await _unitOfWork.SaveChangesAsync(ct);
}
```
