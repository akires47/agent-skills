# async-parallel-foreach

Use Parallel.ForEachAsync for concurrent async operations instead of Task.WhenAll with Select.

## Why It Matters

Parallel.ForEachAsync provides controlled concurrency, better cancellation support, and clearer intent than Task.WhenAll. It prevents overwhelming downstream services with unbounded parallelism and manages resources more efficiently.

## ❌ Incorrect

```csharp
// BAD: Unbounded parallelism with Task.WhenAll
public async Task ProcessOrdersAsync(List<Order> orders)
{
    // Launches 10,000 tasks if orders.Count = 10,000!
    await Task.WhenAll(orders.Select(order => ProcessOrderAsync(order)));
}

// BAD: Sequential processing when parallelism would help
public async Task ProcessOrdersAsync(List<Order> orders)
{
    foreach (var order in orders)
    {
        await ProcessOrderAsync(order);  // One at a time - slow!
    }
}

// BAD: Manual batching is complex
public async Task ProcessOrdersAsync(List<Order> orders)
{
    const int batchSize = 10;
    for (int i = 0; i < orders.Count; i += batchSize)
    {
        var batch = orders.Skip(i).Take(batchSize);
        await Task.WhenAll(batch.Select(ProcessOrderAsync));
    }
}
```

## ✅ Correct

```csharp
// GOOD: Controlled parallelism with Parallel.ForEachAsync
public async Task ProcessOrdersAsync(
    List<Order> orders,
    CancellationToken ct = default)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = 10,  // Max 10 concurrent operations
        CancellationToken = ct
    };
    
    await Parallel.ForEachAsync(orders, options, async (order, ct) =>
    {
        await ProcessOrderAsync(order, ct);
    });
}

// GOOD: Use Task.WhenAll when you need all results
public async Task<List<OrderResult>> GetOrderDetailsAsync(
    List<string> orderIds,
    CancellationToken ct = default)
{
    // OK for small, bounded collections where you need all results
    var tasks = orderIds.Select(id => GetOrderAsync(id, ct));
    var results = await Task.WhenAll(tasks);
    return results.ToList();
}

// GOOD: Parallel with IAsyncEnumerable source
public async Task ProcessOrderStreamAsync(
    IAsyncEnumerable<Order> orders,
    CancellationToken ct = default)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = 5,
        CancellationToken = ct
    };
    
    await Parallel.ForEachAsync(orders, options, async (order, ct) =>
    {
        await ProcessOrderAsync(order, ct);
    });
}

// GOOD: Dynamic concurrency based on environment
public async Task ProcessOrdersAsync(
    List<Order> orders,
    CancellationToken ct = default)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = Environment.ProcessorCount * 2,
        CancellationToken = ct
    };
    
    await Parallel.ForEachAsync(orders, options, async (order, ct) =>
    {
        await ProcessOrderAsync(order, ct);
    });
}
```

## Context

- Use Parallel.ForEachAsync for side effects (processing, updating)
- Use Task.WhenAll when you need all results from small, bounded collections
- Set `MaxDegreeOfParallelism` to avoid overwhelming downstream services
- Always pass `CancellationToken` through to support cancellation
- Related: `async-cancellation-token.md`, `async-semaphore.md`, `async-iasyncenumerable.md`
- For CPU-bound work, use `Parallel.ForEach` (not async version)
