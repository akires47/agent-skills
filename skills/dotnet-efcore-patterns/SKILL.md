---
name: dotnet-efcore-patterns
description: Entity Framework Core best practices including NoTracking by default, migration management, dedicated migration services, and common pitfalls to avoid.
---

# Entity Framework Core Patterns

## When to Use This Skill

Use this skill when:
- Setting up EF Core in a new project
- Optimizing query performance
- Managing database migrations
- Integrating EF Core with .NET Aspire
- Debugging change tracking issues

## Core Principles

1. **NoTracking by Default** - Most queries are read-only; opt-in to tracking
2. **Never Edit Migrations Manually** - Always use CLI commands
3. **Dedicated Migration Service** - Separate migration execution from application startup
4. **ExecutionStrategy for Retries** - Handle transient database failures
5. **Explicit Updates** - When NoTracking, explicitly mark entities for update

---

## Quick Start

### NoTracking by Default

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
        // Disable change tracking by default for read-only queries
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }

    public DbSet<Order> Orders => Set<Order>();
}
```

### Read-Only Queries

```csharp
// Works normally with NoTracking
var orders = await _db.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync();
```

### Updates with NoTracking

```csharp
// Must explicitly enable tracking or use Entry API
var order = await _db.Orders.FindAsync(id); // FindAsync tracks automatically
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();

// Or use Entry API
var order = await _db.Orders.AsNoTracking().FirstAsync(o => o.Id == id);
order.Status = OrderStatus.Shipped;
_db.Entry(order).State = EntityState.Modified;
await _db.SaveChangesAsync();
```

---

## Migration Management

### Create Migration

```bash
dotnet ef migrations add AddOrderStatusColumn --project MyApp.Infrastructure
```

### Apply Migrations

```bash
# Development
dotnet ef database update --project MyApp.Infrastructure

# Production - use dedicated migration service
```

### Remove Last Migration

```bash
dotnet ef migrations remove --project MyApp.Infrastructure
```

---

## References

See detailed patterns and examples in the `references/` folder:

- [NoTracking Pattern](references/notracking-pattern.md) - Configuration, read/write operations, pitfalls
- [Migration Management](references/migrations.md) - CLI commands, workflow, best practices
- [Migration Service](references/migration-service.md) - .NET Aspire integration, dedicated service pattern
- [Bulk Operations](references/bulk-operations.md) - ExecuteUpdate, ExecuteDelete for performance
- [Common Pitfalls](references/common-pitfalls.md) - Mistakes to avoid with change tracking
- [Testing Patterns](references/testing.md) - In-memory vs TestContainers

## Resources

- **EF Core Documentation**: https://learn.microsoft.com/en-us/ef/core/
- **Migrations**: https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/
- **Change Tracking**: https://learn.microsoft.com/en-us/ef/core/change-tracking/
