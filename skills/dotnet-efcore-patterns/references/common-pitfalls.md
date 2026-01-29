# Common Pitfalls

## 1. Forgetting to Save Changes

```csharp
// ❌ BAD: No SaveChangesAsync()
var order = await _db.Orders.FindAsync(id);
order.Status = OrderStatus.Shipped;
// Changes not persisted!

// ✅ GOOD: Always call SaveChangesAsync()
var order = await _db.Orders.FindAsync(id);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();
```

## 2. Modifying NoTracking Entities

```csharp
// ❌ BAD: Modifying entity fetched with AsNoTracking()
var order = await _db.Orders.AsNoTracking().FirstAsync(o => o.Id == id);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync(); // Does nothing!

// ✅ GOOD: Use FindAsync or attach entity
var order = await _db.Orders.FindAsync(id);
order.Status = OrderStatus.Shipped;
await _db.SaveChangesAsync();
```

## 3. N+1 Query Problem

```csharp
// ❌ BAD: N+1 queries (1 for orders, N for customers)
var orders = await _db.Orders.ToListAsync();
foreach (var order in orders)
{
    var customer = await _db.Customers.FindAsync(order.CustomerId);
    // This runs N queries!
}

// ✅ GOOD: Use Include() for eager loading
var orders = await _db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
// Single query with JOIN
```

## 4. Loading Too Much Data

```csharp
// ❌ BAD: Loading entire table into memory
var orders = await _db.Orders.ToListAsync();
var total = orders.Sum(o => o.Total);

// ✅ GOOD: Use server-side aggregation
var total = await _db.Orders.SumAsync(o => o.Total);
```

## 5. Using Dispose Instead of DisposeAsync

```csharp
// ❌ BAD: Synchronous dispose
public void Dispose()
{
    _db.Dispose(); // Blocks thread!
}

// ✅ GOOD: Async dispose
public async ValueTask DisposeAsync()
{
    await _db.DisposeAsync();
}
```

## 6. Tracking Entities Across Requests

```csharp
// ❌ BAD: Singleton or static DbContext
public class OrderService
{
    private static DbContext _db = new AppDbContext(); // DON'T!
}

// ✅ GOOD: Scoped DbContext per request
public class OrderService
{
    private readonly AppDbContext _db;

    public OrderService(AppDbContext db)
    {
        _db = db; // Injected, scoped per request
    }
}
```

## 7. Client-Side Evaluation

```csharp
// ❌ BAD: Client-side evaluation (loads all orders!)
var orders = await _db.Orders
    .Where(o => MyCustomMethod(o.Status))
    .ToListAsync();

// ✅ GOOD: Server-side comparison
var orders = await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();
```

## 8. Incorrect Transaction Handling

```csharp
// ❌ BAD: Nested SaveChanges without transaction
await _db.Orders.AddAsync(order);
await _db.SaveChangesAsync();

await _db.OrderItems.AddAsync(item);
await _db.SaveChangesAsync(); // Order saved even if this fails!

// ✅ GOOD: Use transaction
using var transaction = await _db.Database.BeginTransactionAsync();
try
{
    await _db.Orders.AddAsync(order);
    await _db.SaveChangesAsync();

    await _db.OrderItems.AddAsync(item);
    await _db.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## 9. Not Handling Concurrency

```csharp
// ❌ BAD: No concurrency check
public class Order
{
    public int Id { get; set; }
    public string Status { get; set; }
}

// ✅ GOOD: Use RowVersion for concurrency
public class Order
{
    public int Id { get; set; }
    public string Status { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

## 10. Inefficient Exists Checks

```csharp
// ❌ BAD: Loads data just to check existence
var exists = (await _db.Orders.Where(o => o.Id == id).ToListAsync()).Any();

// ✅ GOOD: Use AnyAsync
var exists = await _db.Orders.AnyAsync(o => o.Id == id);
```

## 11. Not Using Compiled Queries

```csharp
// ❌ BAD: Query compiled every time
public async Task<Order> GetOrderAsync(int id)
{
    return await _db.Orders.FirstAsync(o => o.Id == id);
}

// ✅ GOOD: Compiled query (for hot paths)
private static readonly Func<AppDbContext, int, Task<Order>> GetOrderQuery =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Orders.First(o => o.Id == id));

public Task<Order> GetOrderAsync(int id)
{
    return GetOrderQuery(_db, id);
}
```

## 12. Detached Entity Updates

```csharp
// ❌ BAD: Updating detached entity properties
var order = new Order { Id = 123, Status = "Shipped" };
_db.Orders.Update(order); // Overwrites ALL properties!

// ✅ GOOD: Attach and mark specific properties modified
var order = new Order { Id = 123, Status = "Shipped" };
_db.Orders.Attach(order);
_db.Entry(order).Property(o => o.Status).IsModified = true;
await _db.SaveChangesAsync();
```
