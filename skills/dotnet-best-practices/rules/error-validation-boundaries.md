# error-validation-boundaries

Validate inputs at system boundaries and return early, don't validate deep in the call stack.

## Why It Matters

Validating at boundaries provides fast feedback, prevents invalid data from entering your system, and makes error handling predictable. Deep validation creates hidden dependencies and makes code harder to test and reason about.

## ❌ Incorrect

```csharp
// BAD: Validation scattered throughout the call stack
public async Task<Result<Order>> CreateOrderAsync(CreateOrderRequest request)
{
    var order = await _orderFactory.CreateAsync(request);  // Validates here?
    await _inventoryService.ReserveAsync(order);           // Validates here?
    await _paymentService.ChargeAsync(order);              // Validates here?
    await _repository.SaveAsync(order);                     // Validates here?
    
    return Result.Success(order);
}

// User has no idea validation failed until deep in processing
```

## ✅ Correct

```csharp
// GOOD: Validate once at the boundary, fail fast
public static async Task<Result<Order>> HandleAsync(
    CreateOrderRequest request,
    AppDbContext db,
    CancellationToken ct)
{
    // Validate at entry point
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    // Now we know data is valid - process confidently
    var order = ToEntity(request);
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);
    
    return Result.Success(order);
}

private static ValidationResult Validate(CreateOrderRequest request) =>
    ValidationExtensions.Validate()
        .NotEmpty(request.CustomerId, nameof(request.CustomerId))
        .NotEmpty(request.Items, nameof(request.Items))
        .GreaterThan(request.Items.Count, 0, "Must have at least one item")
        .All(request.Items, item => item.Quantity > 0, "Quantity must be positive");
```

## Context

- Validate at API endpoints, message handlers, and batch job entry points
- Use guard clauses for preconditions in internal methods
- Don't re-validate data that was already validated at the boundary
- Related: `error-guard-clauses.md`, `error-result-pattern.md`
- Validation should be synchronous and fast (no database calls)
