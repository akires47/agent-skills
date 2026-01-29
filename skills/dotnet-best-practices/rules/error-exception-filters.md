# error-exception-filters

Use exception filters to conditionally catch exceptions without unwinding the stack prematurely.

## Why It Matters

Exception filters preserve the original call stack for exceptions that don't match the filter condition. This provides better diagnostics and debugging information compared to catching and rethrowing exceptions.

## ❌ Incorrect

```csharp
// BAD: Catch and rethrow loses stack information
public async Task<Result<Order>> GetOrderAsync(string orderId)
{
    try
    {
        return Result.Success(await _repository.GetAsync(orderId));
    }
    catch (NotFoundException ex)
    {
        if (ex.ResourceType == "Order")
        {
            return Result.Failure<Order>(Error.NotFound("Order", orderId));
        }
        throw;  // Stack trace is reset here
    }
}

// BAD: Catching too broadly
public async Task ProcessAsync()
{
    try
    {
        await _service.ProcessAsync();
    }
    catch (Exception ex)
    {
        if (ex is SqlException sqlEx && sqlEx.Number == 1205)
        {
            // Handle deadlock
        }
        throw;  // Lost stack trace
    }
}
```

## ✅ Correct

```csharp
// GOOD: Exception filter preserves full stack trace
public async Task<Result<Order>> GetOrderAsync(string orderId)
{
    try
    {
        return Result.Success(await _repository.GetAsync(orderId));
    }
    catch (NotFoundException ex) when (ex.ResourceType == "Order")
    {
        // Stack is preserved if condition is false
        return Result.Failure<Order>(Error.NotFound("Order", orderId));
    }
}

// GOOD: Filter specific exceptions with conditions
public async Task ProcessAsync()
{
    try
    {
        await _service.ProcessAsync();
    }
    catch (SqlException ex) when (ex.Number == 1205)
    {
        // Handle deadlock specifically
        _logger.LogWarning(ex, "Deadlock detected, retrying...");
        await Task.Delay(TimeSpan.FromSeconds(1));
        await ProcessAsync();
    }
}

// GOOD: Use filters for logging without catching
public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
{
    try
    {
        return await _repository.CreateAsync(request);
    }
    catch (Exception ex) when (LogException(ex))
    {
        throw;  // Never reached - LogException returns false
    }
}

private bool LogException(Exception ex)
{
    _logger.LogError(ex, "Order creation failed");
    return false;  // Don't catch, just log
}
```

## Context

- Exception filters use `when (condition)` clause
- Stack trace is only unwound if filter condition is true
- Useful for logging without catching: `when (LogAndReturnFalse(ex))`
- Better for conditional exception handling than catch-and-rethrow
- Related: `error-global-handlers.md`, `error-never-catch-all.md`
- .NET 6+ adds `ExceptionDispatchInfo` for advanced scenarios
