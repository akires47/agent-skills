# NoTracking Pattern

## Configuration

Disable change tracking by default in your DbContext:

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
        // Disable change tracking by default
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
}
```

## Read-Only Queries

With NoTracking enabled, read queries work normally:

```csharp
// No tracking - fast, read-only
var orders = await _db.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync();

// Join queries work fine
var orderDetails = await _db.Orders
    .Include(o => o.Items)
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();
```

## Updates with NoTracking

When NoTracking is the default, use these patterns for updates:

### Pattern 1: FindAsync (Automatically Tracks)

```csharp
// FindAsync enables tracking automatically
var order = await _db.Orders.FindAsync(orderId);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();
```

### Pattern 2: AsTracking() Explicitly

```csharp
// Explicitly enable tracking for this query
var order = await _db.Orders
    .AsTracking()
    .FirstOrDefaultAsync(o => o.Id == orderId);

order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();
```

### Pattern 3: Entry API (Attach and Modify)

```csharp
// Fetch without tracking
var order = await _db.Orders
    .AsNoTracking()
    .FirstAsync(o => o.Id == orderId);

// Modify entity
order.Status = OrderStatus.Shipped;

// Explicitly mark as modified
_db.Entry(order).State = EntityState.Modified;
await _db.SaveChangesAsync();
```

### Pattern 4: Update Specific Properties

```csharp
var order = await _db.Orders.AsNoTracking().FirstAsync(o => o.Id == orderId);
order.Status = OrderStatus.Shipped;

var entry = _db.Entry(order);
entry.State = EntityState.Modified;
entry.Property(o => o.Status).IsModified = true;
await _db.SaveChangesAsync();
```

## Inserts with NoTracking

NoTracking doesn't affect inserts:

```csharp
var newOrder = new Order
{
    CustomerId = "C123",
    Total = 99.99m,
    Status = OrderStatus.Pending
};

_db.Orders.Add(newOrder);
await _db.SaveChangesAsync();
```

## Deletes with NoTracking

### Pattern 1: Load Then Delete

```csharp
var order = await _db.Orders.FindAsync(orderId);
_db.Orders.Remove(order);
await _db.SaveChangesAsync();
```

### Pattern 2: Attach Then Delete

```csharp
var order = new Order { Id = orderId };
_db.Orders.Attach(order);
_db.Orders.Remove(order);
await _db.SaveChangesAsync();
```

### Pattern 3: ExecuteDelete (EF Core 7+)

```csharp
// Bulk delete without loading entities
await _db.Orders
    .Where(o => o.Status == OrderStatus.Cancelled)
    .ExecuteDeleteAsync();
```

## Common Pitfalls

### ❌ BAD: Modifying Without Attaching

```csharp
// This does nothing!
var order = await _db.Orders.AsNoTracking().FirstAsync(o => o.Id == orderId);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync(); // No update!
```

### ✅ GOOD: Attach and Mark Modified

```csharp
var order = await _db.Orders.AsNoTracking().FirstAsync(o => o.Id == orderId);
order.Status = OrderStatus.Shipped;
_db.Entry(order).State = EntityState.Modified;
await _db.SaveChangesAsync(); // Update happens
```

## When to Enable Tracking

Enable tracking when:
- Updating multiple entities in a single transaction
- Complex change tracking scenarios
- Working with navigation properties that need change detection

```csharp
// Enable tracking for complex operations
var order = await _db.Orders
    .AsTracking()
    .Include(o => o.Items)
    .FirstAsync(o => o.Id == orderId);

// Modify order and items
order.Status = OrderStatus.Processing;
order.Items.First().Quantity = 10;

// SaveChanges tracks all changes automatically
await _db.SaveChangesAsync();
```
