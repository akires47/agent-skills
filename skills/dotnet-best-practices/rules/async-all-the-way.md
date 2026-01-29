# async-all-the-way

Use async/await throughout the entire call chain; never block on async code with .Result or .Wait().

## Why It Matters

Blocking on async code causes deadlocks in UI and ASP.NET contexts, wastes thread pool threads, and prevents proper cancellation. Async must flow all the way from the entry point to the I/O operation.

## ❌ Incorrect

```csharp
// BAD: Blocking on async code causes deadlocks
public Order GetOrder(string orderId)
{
    // DEADLOCK RISK in ASP.NET/UI contexts
    return _repository.GetOrderAsync(orderId).Result;
}

public IActionResult CreateOrder(CreateOrderRequest request)
{
    var result = _service.CreateOrderAsync(request).GetAwaiter().GetResult();
    return Ok(result);
}

// BAD: Mixing sync and async in call chain
public async Task<Order> ProcessOrderAsync(string orderId)
{
    var order = GetOrder(orderId);  // Sync call that blocks internally
    await SaveOrderAsync(order);     // Now async?
    return order;
}

// BAD: Async wrapper around blocking code
public async Task<string> ReadFileAsync(string path)
{
    return await Task.Run(() => File.ReadAllText(path));  // Still blocks a thread!
}
```

## ✅ Correct

```csharp
// GOOD: Async all the way
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken ct = default)
{
    return await _repository.GetOrderAsync(orderId, ct);
}

public async Task<IActionResult> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct)
{
    var result = await _service.CreateOrderAsync(request, ct);
    return Ok(result);
}

// GOOD: Fully async call chain
public async Task<Order> ProcessOrderAsync(
    string orderId,
    CancellationToken ct = default)
{
    var order = await GetOrderAsync(orderId, ct);
    await ValidateInventoryAsync(order, ct);
    await SaveOrderAsync(order, ct);
    return order;
}

// GOOD: Use native async APIs
public async Task<string> ReadFileAsync(
    string path,
    CancellationToken ct = default)
{
    return await File.ReadAllTextAsync(path, ct);
}

// GOOD: For truly CPU-bound work, use Task.Run at the entry point
public async Task<Report> GenerateReportAsync(
    ReportRequest request,
    CancellationToken ct = default)
{
    var data = await LoadDataAsync(request, ct);
    
    // CPU-intensive work offloaded to thread pool
    return await Task.Run(() => ProcessReport(data, ct), ct);
}
```

## Context

- Never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`
- Use async signatures all the way: `Task<T>` instead of `T`
- ASP.NET Core supports async controllers and minimal API handlers
- For legacy sync code, create async versions instead of wrapping
- Related: `async-cancellation-token.md`, `async-avoid-async-void.md`
- If you must block (console app main), use `.GetAwaiter().GetResult()`
