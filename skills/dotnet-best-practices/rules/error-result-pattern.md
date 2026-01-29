# error-result-pattern

Use the Result pattern to return success or failure instead of throwing exceptions for expected errors.

## Why It Matters

Exceptions for expected errors hide control flow and hurt performance. The Result pattern makes error handling explicit, forces callers to handle failures, and improves performance by avoiding expensive stack unwinding for business logic errors.

## ❌ Incorrect

```csharp
// BAD: Throwing exceptions for business logic errors
public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
{
    if (request.Items.Count == 0)
        throw new ValidationException("Order must have at least one item");
    
    var customer = await _repository.GetCustomerAsync(request.CustomerId);
    if (customer == null)
        throw new NotFoundException($"Customer {request.CustomerId} not found");
    
    if (customer.Balance < request.Total)
        throw new BusinessException("Insufficient balance");
    
    return await _repository.CreateAsync(order);
}

// Caller has no idea what can go wrong without reading implementation
var order = await CreateOrderAsync(request);
```

## ✅ Correct

```csharp
// GOOD: Result pattern makes errors explicit
public async Task<Result<Order>> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct = default)
{
    if (request.Items.Count == 0)
        return Result.Failure<Order>(Error.Validation("Order must have at least one item"));
    
    var customer = await _repository.GetCustomerAsync(request.CustomerId, ct);
    if (customer == null)
        return Result.Failure<Order>(Error.NotFound("Customer", request.CustomerId));
    
    if (customer.Balance < request.Total)
        return Result.Failure<Order>(Error.Conflict("Insufficient balance"));
    
    var order = await _repository.CreateAsync(request, ct);
    return Result.Success(order);
}

// Caller must handle both success and failure
var result = await CreateOrderAsync(request, ct);
return result.Match(
    onSuccess: order => Results.Created($"/orders/{order.Id}", order),
    onFailure: error => error.ToHttpResult());
```

## Context

- Use Result pattern for **expected errors** (validation, not found, business rules)
- Use exceptions for **unexpected errors** (system failures, programming errors)
- Error types: Validation, NotFound, Conflict, Unauthorized, Forbidden
- Related: `error-validation-boundaries.md`, `error-guard-clauses.md`
- See dotnet-vertical-slice skill for full Result implementation
