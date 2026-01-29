# Async/Await Best Practices

**Always use async for I/O-bound operations.**

## Async All the Way

```csharp
// ✅ GOOD: Async all the way
public async Task<Order> GetOrderAsync(string orderId, CancellationToken cancellationToken)
{
    var order = await _repository.GetAsync(orderId, cancellationToken);
    var customer = await _customerService.GetCustomerAsync(order.CustomerId, cancellationToken);
    return order;
}

// ❌ BAD: Blocking on async code
public Order GetOrder(string orderId)
{
    return _repository.GetAsync(orderId).Result;  // DEADLOCK RISK!
}
```

## ValueTask for Hot Paths

```csharp
// ✅ GOOD: ValueTask for frequently-called, often-synchronous methods
public ValueTask<Order?> GetCachedOrderAsync(string orderId, CancellationToken cancellationToken)
{
    if (_cache.TryGetValue(orderId, out var order))
        return ValueTask.FromResult<Order?>(order);  // Synchronous path, no allocation

    return GetFromDatabaseAsync(orderId, cancellationToken);  // Async path
}

private async ValueTask<Order?> GetFromDatabaseAsync(string orderId, CancellationToken cancellationToken)
{
    var order = await _repository.GetAsync(orderId, cancellationToken);
    if (order is not null)
        _cache[orderId] = order;
    return order;
}
```

## IAsyncEnumerable for Streaming

```csharp
// ✅ GOOD: IAsyncEnumerable for streaming
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    string customerId,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var order in _repository.StreamAllAsync(cancellationToken))
    {
        if (order.CustomerId == customerId)
            yield return order;
    }
}
```

## ConfigureAwait in Library Code

```csharp
// ✅ GOOD: ConfigureAwait(false) in library code (not application code)
public async Task<string> ProcessDataAsync(string input, CancellationToken cancellationToken)
{
    var data = await FetchDataAsync(cancellationToken).ConfigureAwait(false);
    var result = await TransformDataAsync(data, cancellationToken).ConfigureAwait(false);
    return result;
}
```

## Always Accept CancellationToken

```csharp
// ✅ GOOD: CancellationToken parameter with default
public async Task<List<Order>> GetOrdersAsync(
    string customerId,
    CancellationToken cancellationToken = default)
{
    var orders = await _repository.GetOrdersByCustomerAsync(customerId, cancellationToken);
    return orders;
}

// Pass cancellation through the call stack
public async Task<OrderSummary> GetOrderSummaryAsync(
    string customerId,
    CancellationToken cancellationToken = default)
{
    var orders = await GetOrdersAsync(customerId, cancellationToken);
    var total = orders.Sum(o => o.Total);
    return new OrderSummary(customerId, orders.Count, total);
}

// Link cancellation tokens when composing operations
public async Task<ProcessResult> ProcessWithTimeoutAsync(
    string data,
    TimeSpan timeout,
    CancellationToken cancellationToken = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(timeout);

    return await ProcessAsync(data, cts.Token);
}
```
