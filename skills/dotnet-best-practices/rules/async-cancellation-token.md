# async-cancellation-token

Always accept CancellationToken parameters in async methods to support request cancellation and timeouts.

## Why It Matters

Cancellation tokens allow long-running operations to be cancelled gracefully, freeing resources and preventing wasted work. This is critical for web applications where clients may disconnect or for batch operations that need to be stopped.

## ❌ Incorrect

```csharp
// BAD: No cancellation support
public async Task<Order> GetOrderAsync(string orderId)
{
    var order = await _repository.GetAsync(orderId);
    var items = await _repository.GetItemsAsync(orderId);
    var customer = await _repository.GetCustomerAsync(order.CustomerId);
    
    // Cannot cancel even if client disconnects
    return BuildOrderDetails(order, items, customer);
}

// BAD: CancellationToken not passed through
public async Task<Result<Order>> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct)
{
    var order = ToEntity(request);
    await _repository.SaveAsync(order);  // Doesn't pass ct!
    await _eventBus.PublishAsync(new OrderCreated(order.Id));  // No ct!
    
    return Result.Success(order);
}
```

## ✅ Correct

```csharp
// GOOD: Accept and pass through CancellationToken
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken ct = default)
{
    var order = await _repository.GetAsync(orderId, ct);
    var items = await _repository.GetItemsAsync(orderId, ct);
    var customer = await _repository.GetCustomerAsync(order.CustomerId, ct);
    
    return BuildOrderDetails(order, items, customer);
}

// GOOD: Thread through entire call chain
public async Task<Result<Order>> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct = default)
{
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    var order = ToEntity(request);
    await _repository.SaveAsync(order, ct);
    await _eventBus.PublishAsync(new OrderCreated(order.Id), ct);
    
    return Result.Success(order);
}

// GOOD: Check cancellation in loops
public async Task ProcessBatchAsync(
    IEnumerable<Order> orders,
    CancellationToken ct = default)
{
    foreach (var order in orders)
    {
        ct.ThrowIfCancellationRequested();
        await ProcessOrderAsync(order, ct);
    }
}

// GOOD: Use linked token for timeout
public async Task<Order> GetOrderWithTimeoutAsync(
    string orderId,
    CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(30));
    
    return await GetOrderAsync(orderId, cts.Token);
}
```

## Context

- Make CancellationToken the last parameter with `= default`
- Pass token to all async calls: database, HTTP, file I/O
- Call `ct.ThrowIfCancellationRequested()` in CPU-bound loops
- Use `CancellationTokenSource.CreateLinkedTokenSource()` for composite cancellation
- Related: `async-all-the-way.md`, `async-avoid-async-void.md`
- In ASP.NET Core, token is auto-cancelled when client disconnects
