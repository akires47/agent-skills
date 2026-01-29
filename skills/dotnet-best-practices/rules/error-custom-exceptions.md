# error-custom-exceptions

Create custom exceptions only for exceptional programming errors, not for business logic or validation.

## Why It Matters

Custom exceptions should represent unexpected programmer errors, not expected business scenarios. Overusing custom exceptions for business logic creates hidden control flow and performance problems. Use the Result pattern for expected errors instead.

## ❌ Incorrect

```csharp
// BAD: Custom exceptions for business logic
public class InsufficientBalanceException : Exception
{
    public decimal Balance { get; }
    public decimal Required { get; }
    
    public InsufficientBalanceException(decimal balance, decimal required)
        : base($"Insufficient balance. Have: {balance}, Need: {required}")
    {
        Balance = balance;
        Required = required;
    }
}

public class OrderNotFoundException : Exception { }
public class InvalidOrderStatusException : Exception { }

// Forces callers to use exceptions for control flow
public async Task<Order> ProcessPaymentAsync(Order order)
{
    if (order.Status != OrderStatus.Pending)
        throw new InvalidOrderStatusException();
    
    if (customer.Balance < order.Total)
        throw new InsufficientBalanceException(customer.Balance, order.Total);
    
    return await CompleteOrderAsync(order);
}
```

## ✅ Correct

```csharp
// GOOD: Use Result pattern for business errors
public async Task<Result<Order>> ProcessPaymentAsync(Order order)
{
    if (order.Status != OrderStatus.Pending)
        return Result.Failure<Order>(
            Error.Conflict("Order must be in pending status"));
    
    if (customer.Balance < order.Total)
        return Result.Failure<Order>(
            Error.Conflict($"Insufficient balance. Have: {customer.Balance}, Need: {order.Total}"));
    
    var completed = await CompleteOrderAsync(order);
    return Result.Success(completed);
}

// GOOD: Custom exception for programming errors (not business logic)
public sealed class InvalidEntityStateException : InvalidOperationException
{
    public InvalidEntityStateException(string entityType, string reason)
        : base($"Entity {entityType} is in an invalid state: {reason}")
    {
    }
}

// Use for programmer mistakes, not user errors
public void EnsureNotDeleted()
{
    if (_isDeleted)
        throw new InvalidEntityStateException(
            GetType().Name,
            "Cannot operate on deleted entity");
}

// GOOD: Use built-in exceptions for programmer errors
public decimal Divide(decimal numerator, decimal denominator)
{
    if (denominator == 0)
        throw new ArgumentException("Denominator cannot be zero", nameof(denominator));
    
    return numerator / denominator;
}
```

## Context

- Use Result pattern for **business logic errors** (validation, conflicts, not found)
- Use exceptions for **programming errors** (invalid state, contract violations)
- Prefer built-in exceptions: `ArgumentException`, `InvalidOperationException`, `NotSupportedException`
- Custom exceptions should derive from appropriate base types
- Related: `error-result-pattern.md`, `error-exception-properties.md`
- Exception creation is expensive - avoid for expected flows
