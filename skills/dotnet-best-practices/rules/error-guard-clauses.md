# error-guard-clauses

Use guard clauses to validate preconditions at the start of methods, failing fast with clear error messages.

## Why It Matters

Guard clauses reduce nesting, improve readability, and fail fast when preconditions aren't met. They make method contracts explicit and prevent invalid state from propagating through your system.

## ❌ Incorrect

```csharp
// BAD: Deeply nested validation logic
public decimal CalculateDiscount(Order order, Customer customer)
{
    if (order != null)
    {
        if (customer != null)
        {
            if (order.Total > 0)
            {
                if (customer.LoyaltyPoints >= 100)
                {
                    return order.Total * 0.1m;
                }
                else
                {
                    return 0;
                }
            }
            else
            {
                throw new InvalidOperationException("Order total must be positive");
            }
        }
        else
        {
            throw new ArgumentNullException(nameof(customer));
        }
    }
    else
    {
        throw new ArgumentNullException(nameof(order));
    }
}
```

## ✅ Correct

```csharp
// GOOD: Guard clauses at the top, happy path at bottom
public decimal CalculateDiscount(Order order, Customer customer)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentNullException.ThrowIfNull(customer);
    
    if (order.Total <= 0)
        throw new ArgumentException("Order total must be positive", nameof(order));
    
    if (customer.LoyaltyPoints < 100)
        return 0;
    
    // Happy path - no nesting
    return order.Total * 0.1m;
}

// BETTER: Use Result pattern for expected validation errors
public static Result<decimal> CalculateDiscount(Order order, Customer customer)
{
    if (order.Total <= 0)
        return Result.Failure<decimal>(Error.Validation("Order total must be positive"));
    
    if (customer.LoyaltyPoints < 100)
        return Result.Success(0m);
    
    return Result.Success(order.Total * 0.1m);
}
```

## Context

- Use `ArgumentNullException.ThrowIfNull()` for null checks (.NET 6+)
- Use `ArgumentException` for invalid arguments in internal methods
- Use Result pattern for validation at public boundaries
- Guard clauses prevent defensive programming deeper in the stack
- Related: `error-validation-boundaries.md`, `error-result-pattern.md`
