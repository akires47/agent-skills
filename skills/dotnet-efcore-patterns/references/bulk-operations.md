# Bulk Operations (EF Core 7+)

## ExecuteUpdate

Update multiple rows without loading them into memory:

```csharp
// Update all pending orders to processing status
await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, OrderStatus.Processing)
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow));

// Update with computed values
await _db.Orders
    .Where(o => o.Total > 1000)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.DiscountApplied, o => o.Total * 0.1m));
```

## ExecuteDelete

Delete multiple rows without loading them:

```csharp
// Delete all cancelled orders
await _db.Orders
    .Where(o => o.Status == OrderStatus.Cancelled)
    .ExecuteDeleteAsync();

// Delete with complex conditions
await _db.Orders
    .Where(o => o.CreatedAt < DateTime.UtcNow.AddMonths(-6) &&
                o.Status == OrderStatus.Completed)
    .ExecuteDeleteAsync();
```

## Performance Benefits

```csharp
// ❌ BAD: Load all entities, modify, save (slow!)
var orders = await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

foreach (var order in orders)
{
    order.Status = OrderStatus.Processing;
}

await _db.SaveChangesAsync();

// ✅ GOOD: Single SQL statement (fast!)
await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Processing));
```

## Limitations

ExecuteUpdate/Delete:
- Cannot use navigation properties
- Cannot use certain LINQ methods (GroupBy, etc.)
- Bypasses change tracking and events
- No automatic audit logging

```csharp
// ❌ This won't work
await _db.Orders
    .Include(o => o.Customer)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Customer.Name, "Updated"));

// ✅ Use join instead
await _db.Orders
    .Where(o => o.Customer.Id == customerId)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.VIP));
```

## Use Cases

### Mass Status Updates

```csharp
public async Task MarkExpiredOrdersAsCancelled()
{
    await _db.Orders
        .Where(o => o.Status == OrderStatus.Pending &&
                    o.CreatedAt < DateTime.UtcNow.AddDays(-30))
        .ExecuteUpdateAsync(s => s
            .SetProperty(o => o.Status, OrderStatus.Cancelled)
            .SetProperty(o => o.CancellationReason, "Expired"));
}
```

### Archival

```csharp
public async Task ArchiveOldOrders()
{
    // Copy to archive table
    await _db.Orders
        .Where(o => o.CompletedAt < DateTime.UtcNow.AddYears(-1))
        .Select(o => new ArchivedOrder
        {
            OrderId = o.Id,
            CustomerId = o.CustomerId,
            Total = o.Total,
            ArchivedAt = DateTime.UtcNow
        })
        .ExecuteInsertAsync();

    // Delete from main table
    await _db.Orders
        .Where(o => o.CompletedAt < DateTime.UtcNow.AddYears(-1))
        .ExecuteDeleteAsync();
}
```

### Soft Deletes

```csharp
public async Task SoftDeleteOrdersAsync(List<int> orderIds)
{
    await _db.Orders
        .Where(o => orderIds.Contains(o.Id))
        .ExecuteUpdateAsync(s => s
            .SetProperty(o => o.IsDeleted, true)
            .SetProperty(o => o.DeletedAt, DateTime.UtcNow));
}
```
