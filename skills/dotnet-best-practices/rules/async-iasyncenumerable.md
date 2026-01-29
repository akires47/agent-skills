# async-iasyncenumerable

Use IAsyncEnumerable<T> for streaming results instead of loading entire collections into memory.

## Why It Matters

IAsyncEnumerable<T> enables streaming large datasets without loading everything into memory first. This improves memory usage, reduces latency to first result, and supports cancellation during iteration.

## ❌ Incorrect

```csharp
// BAD: Loading entire result set into memory
public async Task<List<Order>> GetOrdersAsync(CustomerId customerId)
{
    // Loads ALL orders into memory at once
    return await _db.Orders
        .Where(o => o.CustomerId == customerId.Value)
        .ToListAsync();
}

// BAD: Materializing then iterating
public async Task ProcessOrdersAsync(CustomerId customerId)
{
    var orders = await GetOrdersAsync(customerId);  // All in memory
    
    foreach (var order in orders)
    {
        await ProcessOrderAsync(order);
    }
}

// BAD: Reading entire file into memory
public async Task<string[]> ReadLinesAsync(string path)
{
    var content = await File.ReadAllTextAsync(path);
    return content.Split('\n');  // Entire file in memory
}
```

## ✅ Correct

```csharp
// GOOD: Stream results as they're fetched
public async IAsyncEnumerable<Order> GetOrdersAsync(
    CustomerId customerId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in _db.Orders
        .Where(o => o.CustomerId == customerId.Value)
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return order;
    }
}

// GOOD: Process as you stream
public async Task ProcessOrdersAsync(
    CustomerId customerId,
    CancellationToken ct = default)
{
    await foreach (var order in GetOrdersAsync(customerId, ct))
    {
        await ProcessOrderAsync(order, ct);
        // Only one order in memory at a time
    }
}

// GOOD: Stream file lines
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(path);
    
    while (!reader.EndOfStream)
    {
        ct.ThrowIfCancellationRequested();
        var line = await reader.ReadLineAsync();
        if (line != null)
            yield return line;
    }
}

// GOOD: Transform stream without materialization
public async IAsyncEnumerable<OrderDto> GetOrderDtosAsync(
    CustomerId customerId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in GetOrdersAsync(customerId, ct))
    {
        yield return new OrderDto(
            order.Id.Value,
            order.Total,
            order.Status);
    }
}

// GOOD: Filter stream
public async IAsyncEnumerable<Order> GetActiveOrdersAsync(
    CustomerId customerId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in GetOrdersAsync(customerId, ct))
    {
        if (order.Status == OrderStatus.Active)
            yield return order;
    }
}
```

## Context

- Use `[EnumeratorCancellation]` attribute for proper cancellation support
- Streams results on-demand: first result arrives faster, lower memory usage
- Perfect for: large result sets, paginated APIs, file processing, real-time data
- EF Core supports `AsAsyncEnumerable()` for streaming queries
- Related: `async-cancellation-token.md`, `perf-defer-enumeration.md`
- Can use LINQ-like operations with `System.Linq.Async` package
