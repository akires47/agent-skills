# db-notracking-default

Configure NoTracking by default in EF Core DbContext

## Why It Matters

Change tracking is expensive and unnecessary for read-only queries. Most queries are reads, so optimize for the common case by disabling tracking globally.

## ❌ Incorrect

```csharp
// BAD: Change tracking enabled for all queries by default
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
    
    public DbSet<Order> Orders => Set<Order>();
}

// Every query pays tracking overhead
var orders = await _db.Orders
    .Where(o => o.CustomerId == id)
    .ToListAsync();
```

## ✅ Correct

```csharp
// GOOD: NoTracking by default
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }
    
    public DbSet<Order> Orders => Set<Order>();
}

// Read queries are fast
var orders = await _db.Orders
    .Where(o => o.CustomerId == id)
    .ToListAsync();

// Explicitly enable tracking when needed
var order = await _db.Orders
    .AsTracking()
    .FirstAsync(o => o.Id == id);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();
```

## Context

- Significant performance improvement for read-heavy workloads
- Explicitly use AsTracking() when you need to update entities
- FindAsync() automatically enables tracking
- See also: `db-projections-over-entities`, `db-read-write-separation`
