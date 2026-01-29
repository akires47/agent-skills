# error-never-catch-all

Never catch all exceptions without rethrowing; it hides critical errors and makes debugging impossible.

## Why It Matters

Catching all exceptions silently swallows critical errors like OutOfMemoryException, StackOverflowException, and ThreadAbortException. This makes systems appear to work while failing silently, leading to data corruption and impossible debugging scenarios.

## ❌ Incorrect

```csharp
// BAD: Swallowing all exceptions
public async Task ProcessOrdersAsync()
{
    foreach (var order in orders)
    {
        try
        {
            await ProcessOrderAsync(order);
        }
        catch (Exception ex)
        {
            // Silently continues - order is lost!
            _logger.LogError(ex, "Failed to process order");
        }
    }
}

// BAD: Catching and returning default value
public Order? GetOrder(string orderId)
{
    try
    {
        return _repository.GetOrder(orderId);
    }
    catch (Exception)
    {
        return null;  // Hides what went wrong - network? not found? database down?
    }
}

// BAD: Generic catch without context
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    try
    {
        var order = await _service.CreateOrderAsync(request);
        return Ok(order);
    }
    catch (Exception ex)
    {
        return BadRequest("Order creation failed");  // Which error? Why?
    }
}
```

## ✅ Correct

```csharp
// GOOD: Don't catch - let it bubble to global handler
public async Task ProcessOrdersAsync()
{
    foreach (var order in orders)
    {
        await ProcessOrderAsync(order);  // Let exceptions propagate
    }
}

// GOOD: Catch specific exceptions only
public async Task<Result<Order>> GetOrderAsync(string orderId)
{
    try
    {
        var order = await _repository.GetOrderAsync(orderId);
        return Result.Success(order);
    }
    catch (SqlException ex) when (ex.Number == -2)  // Timeout
    {
        _logger.LogWarning(ex, "Database timeout getting order {OrderId}", orderId);
        return Result.Failure<Order>(Error.Unavailable("Database temporarily unavailable"));
    }
    // Let other exceptions propagate
}

// GOOD: Use Result pattern instead of catch-all
public async Task<Result<Order>> CreateOrderAsync(CreateOrderRequest request)
{
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    var order = await _repository.CreateAsync(request);
    return Result.Success(order);
    
    // No try-catch needed - expected errors return Result, unexpected errors throw
}

// GOOD: Catch-all only for cleanup, then rethrow
public async Task ProcessWithCleanupAsync()
{
    var resource = await AcquireResourceAsync();
    try
    {
        await ProcessAsync(resource);
    }
    catch
    {
        await CleanupAsync(resource);
        throw;  // Rethrow to preserve stack
    }
}
```

## Context

- Only catch exceptions you can meaningfully handle
- Use `catch (Exception ex) when (condition)` for filtered logging
- Let unexpected exceptions bubble to global handler
- Use Result pattern for expected errors instead of exceptions
- Related: `error-result-pattern.md`, `error-exception-filters.md`, `error-global-handlers.md`
- Never catch `OutOfMemoryException`, `StackOverflowException`, or `ThreadAbortException`
