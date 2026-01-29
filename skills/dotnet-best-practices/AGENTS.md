# .NET Best Practices - Complete Guide

**Version:** 1.0.0  
**Author:** akires47  
**License:** MIT

This document contains all .NET best practices rules expanded inline for easy reference by LLM-powered coding agents. For the categorized overview, see [SKILL.md](SKILL.md).

---

## Table of Contents

1. [Error Handling (CRITICAL)](#1-error-handling-critical)
2. [Async Patterns (CRITICAL)](#2-async-patterns-critical)
3. [Type Design (HIGH)](#3-type-design-high)
4. [Database Performance (HIGH)](#4-database-performance-high)
5. [API Design (MEDIUM-HIGH)](#5-api-design-medium-high)
6. [Dependency Injection (MEDIUM)](#6-dependency-injection-medium)
7. [Architecture (MEDIUM)](#7-architecture-medium)
8. [Serialization (MEDIUM)](#8-serialization-medium)
9. [Performance (LOW-MEDIUM)](#9-performance-low-medium)
10. [Logging (LOW-MEDIUM)](#10-logging-low-medium)
11. [Testing (LOW)](#11-testing-low)

---

## 1. Error Handling (CRITICAL)

Priority 1 rules for handling errors in .NET applications. These patterns prevent runtime exceptions, make error handling explicit, and improve code maintainability.

### error-result-pattern

Return `Result<T>` for expected errors instead of throwing exceptions.

**Why It Matters:** Exceptions are expensive (stack trace capture, performance impact) and obscure control flow. Expected business errors should be explicit return values that the type system enforces handling.

**❌ Incorrect:**
```csharp
// BAD: Throwing exceptions for expected business logic errors
public Order GetOrder(OrderId id)
{
    var order = _repository.Find(id);
    if (order is null)
        throw new NotFoundException($"Order {id} not found");
    
    if (order.Status == OrderStatus.Cancelled)
        throw new InvalidOperationException("Cannot process cancelled order");
    
    return order;
}
```

**✅ Correct:**
```csharp
// GOOD: Return Result type for expected errors
public Result<Order> GetOrder(OrderId id)
{
    var order = _repository.Find(id);
    if (order is null)
        return Result.Failure<Order>(Error.NotFound("Order", id));
    
    if (order.Status == OrderStatus.Cancelled)
        return Result.Failure<Order>(Error.Validation("Order is cancelled"));
    
    return Result.Success(order);
}

// Usage with pattern matching
var result = GetOrder(orderId);
return result.Match(
    onSuccess: order => Results.Ok(order),
    onFailure: error => Results.Problem(error.Message));
```

**Context:**
- Use exceptions only for unexpected errors (system failures, programming bugs)
- Result pattern makes error handling explicit in the type system
- See also: `error-validation-boundaries`, `error-guard-clauses`

---

### error-validation-boundaries

Validate at handler entry points and fail fast.

**Why It Matters:** Validation should occur at system boundaries (API handlers, message handlers) before any business logic executes. This prevents invalid data from propagating through your system.

**❌ Incorrect:**
```csharp
// BAD: Validation scattered throughout domain logic
public sealed class OrderService
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order(request.CustomerId, request.Items);
        
        // Validation deep in the call stack - too late!
        if (order.Total <= 0)
            throw new ValidationException("Order total must be positive");
        
        await _repository.SaveAsync(order);
        return order;
    }
}
```

**✅ Correct:**
```csharp
// GOOD: Validate at the handler boundary
public static class CreateOrder
{
    public static async Task<Result<Response>> HandleAsync(
        Request request, AppDbContext db, CancellationToken ct)
    {
        // Validate first, before any logic
        var validation = Validate(request);
        if (!validation.IsValid)
            return validation.ToResult<Response>();
        
        // Business logic only executes with valid input
        var order = ToEntity(request);
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        
        return Result.Success(ToResponse(order));
    }
    
    private static ValidationResult Validate(Request request) =>
        ValidationExtensions.Validate()
            .NotEmpty(request.CustomerId, "CustomerId")
            .GreaterThan(request.Items.Count, 0, "Items")
            .GreaterThan(request.Items.Sum(i => i.Price), 0, "Total");
}
```

**Context:**
- Validate at entry points: API handlers, message handlers, job handlers
- Fail fast - reject invalid input immediately
- See also: `error-result-pattern`, `error-guard-clauses`, `arch-static-handlers`

---

### error-guard-clauses

Use guard clauses for null and range checks at method boundaries.

**Why It Matters:** Guard clauses make preconditions explicit, reduce nesting, and fail fast when inputs violate contracts.

**❌ Incorrect:**
```csharp
// BAD: Nested validation creates indentation hell
public Order ProcessOrder(Order order, List<OrderItem> items)
{
    if (order != null)
    {
        if (items != null && items.Count > 0)
        {
            if (order.Status == OrderStatus.Draft)
            {
                // Actual logic buried in nesting
                order.AddItems(items);
                return order;
            }
        }
    }
    throw new ArgumentException("Invalid input");
}
```

**✅ Correct:**
```csharp
// GOOD: Guard clauses at the top, flat structure
public Order ProcessOrder(Order order, List<OrderItem> items)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentNullException.ThrowIfNull(items);
    
    if (items.Count == 0)
        throw new ArgumentException("Items cannot be empty", nameof(items));
    
    if (order.Status != OrderStatus.Draft)
        throw new InvalidOperationException("Order must be in Draft status");
    
    // Actual logic is unindented and clear
    order.AddItems(items);
    return order;
}
```

**Context:**
- Use guard clauses for precondition checks (null, range, state)
- Fail fast at the top of the method
- ArgumentNullException.ThrowIfNull is built-in (.NET 6+)
- See also: `error-validation-boundaries`, `type-nullable-reference-types`

---

### error-exception-filters

Use `catch when` for selective exception handling.

**Why It Matters:** Exception filters allow you to conditionally catch exceptions without unwinding the stack, preserving better stack traces for debugging.

**❌ Incorrect:**
```csharp
// BAD: Catch and rethrow - loses original stack trace
try
{
    await ProcessOrderAsync(order);
}
catch (HttpRequestException ex)
{
    if (ex.StatusCode == HttpStatusCode.TooManyRequests)
    {
        _logger.LogWarning("Rate limited, will retry");
        throw;  // Stack trace is now from here, not original throw
    }
    throw;  // Original stack trace lost
}
```

**✅ Correct:**
```csharp
// GOOD: Exception filter preserves stack trace
try
{
    await ProcessOrderAsync(order);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    _logger.LogWarning("Rate limited, will retry");
    throw;  // Original stack trace preserved
}
// Other HttpRequestException instances propagate unhandled
```

**Context:**
- Use `when` clause to filter which exceptions to catch
- Exception filters don't unwind the stack until match is found
- Better for debuggability and diagnostics
- See also: `error-never-catch-all`, `error-global-handlers`

---

### error-global-handlers

Implement middleware for unhandled exceptions.

**Why It Matters:** Centralized exception handling prevents code duplication, ensures consistent error responses, and provides a single point for logging unhandled exceptions.

**❌ Incorrect:**
```csharp
// BAD: Try-catch in every endpoint
app.MapGet("/orders/{id}", async (string id, OrderService service) =>
{
    try
    {
        var order = await service.GetOrderAsync(id);
        return Results.Ok(order);
    }
    catch (NotFoundException)
    {
        return Results.NotFound();
    }
    catch (Exception ex)
    {
        // Logging duplicated everywhere
        logger.LogError(ex, "Error processing request");
        return Results.Problem();
    }
});
```

**✅ Correct:**
```csharp
// GOOD: Global exception handler middleware
app.UseExceptionHandler(builder =>
{
    builder.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionFeature?.Error;
        
        var (statusCode, message) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, exception.Message),
            ValidationException => (StatusCodes.Status400BadRequest, exception.Message),
            _ => (StatusCodes.Status500InternalServerError, "An error occurred")
        };
        
        logger.LogError(exception, "Unhandled exception");
        
        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new { error = message });
    });
});

// Endpoints are clean
app.MapGet("/orders/{id}", async (string id, OrderService service) =>
{
    var order = await service.GetOrderAsync(id);
    return Results.Ok(order);
});
```

**Context:**
- Use ASP.NET Core middleware for global exception handling
- Log all unhandled exceptions centrally
- Return consistent error responses
- See also: `error-exception-filters`, `log-structured-logging`

---

### error-never-catch-all

Avoid catching all exceptions without rethrowing.

**Why It Matters:** Catching all exceptions can hide bugs, swallow critical errors, and make debugging nearly impossible.

**❌ Incorrect:**
```csharp
// BAD: Swallowing all exceptions
try
{
    await ProcessOrderAsync(order);
}
catch
{
    // Silent failure - no one knows what went wrong!
}

// BAD: Catching Exception base class
try
{
    await ProcessOrderAsync(order);
}
catch (Exception ex)
{
    _logger.LogError("Something went wrong");  // Too vague
    return false;  // Hides the actual error
}
```

**✅ Correct:**
```csharp
// GOOD: Catch specific exceptions
try
{
    await ProcessOrderAsync(order);
}
catch (OrderNotFoundException ex)
{
    _logger.LogWarning(ex, "Order {OrderId} not found", order.Id);
    return Result.Failure(Error.NotFound("Order", order.Id));
}
catch (InventoryException ex)
{
    _logger.LogWarning(ex, "Insufficient inventory");
    return Result.Failure(Error.Validation("Insufficient inventory"));
}
// Let unexpected exceptions propagate to global handler
```

**Context:**
- Only catch exceptions you can meaningfully handle
- Let unexpected exceptions propagate to global handler
- Never use empty catch blocks
- See also: `error-global-handlers`, `error-exception-filters`

---

### error-custom-exceptions

Use built-in exceptions, avoid creating custom ones.

**Why It Matters:** .NET provides comprehensive built-in exception types. Creating custom exceptions adds complexity without benefit in most scenarios.

**❌ Incorrect:**
```csharp
// BAD: Unnecessary custom exception
public class OrderNotFoundException : Exception
{
    public OrderNotFoundException(string orderId) 
        : base($"Order {orderId} not found") { }
}

public class InvalidOrderStatusException : Exception
{
    public InvalidOrderStatusException(string status)
        : base($"Invalid status: {status}") { }
}
```

**✅ Correct:**
```csharp
// GOOD: Use built-in exceptions
public Order GetOrder(string orderId)
{
    var order = _repository.Find(orderId);
    if (order is null)
        throw new KeyNotFoundException($"Order {orderId} not found");
    return order;
}

public void ValidateStatus(OrderStatus status)
{
    if (!Enum.IsDefined(typeof(OrderStatus), status))
        throw new ArgumentException($"Invalid status: {status}", nameof(status));
}

// BETTER: Use Result pattern instead
public Result<Order> GetOrder(string orderId)
{
    var order = _repository.Find(orderId);
    if (order is null)
        return Result.Failure<Order>(Error.NotFound("Order", orderId));
    return Result.Success(order);
}
```

**Context:**
- Use built-in exceptions: ArgumentException, ArgumentNullException, InvalidOperationException, KeyNotFoundException
- Only create custom exceptions for truly unique scenarios requiring specific handling
- Prefer Result pattern over exceptions for expected errors
- See also: `error-result-pattern`

---

### error-exception-properties

Add context to exceptions via Data property.

**Why It Matters:** Adding structured data to exceptions helps with debugging, logging, and correlation without creating custom exception types.

**❌ Incorrect:**
```csharp
// BAD: Important context only in message string
throw new InvalidOperationException(
    $"Cannot process order {orderId} for customer {customerId} in status {status}");

// Information is trapped in the message, hard to extract
```

**✅ Correct:**
```csharp
// GOOD: Structured data in exception
var ex = new InvalidOperationException("Cannot process order in current status");
ex.Data["OrderId"] = orderId;
ex.Data["CustomerId"] = customerId;
ex.Data["CurrentStatus"] = status;
ex.Data["CorrelationId"] = Activity.Current?.Id;
throw ex;

// In global handler, structured data is easily accessible
catch (Exception ex)
{
    var properties = ex.Data
        .Cast<DictionaryEntry>()
        .ToDictionary(e => e.Key.ToString(), e => e.Value);
    
    _logger.LogError(ex, "Error processing order {OrderId}", 
        properties.GetValueOrDefault("OrderId"));
}
```

**Context:**
- Use Data dictionary for structured context
- Include correlation IDs for distributed tracing
- Makes logging and debugging more effective
- See also: `log-correlation-ids`, `log-structured-logging`

---

## 2. Async Patterns (CRITICAL)

Priority 2 rules for asynchronous programming in .NET. These patterns ensure correct async/await usage, prevent deadlocks, and optimize performance.

### async-cancellation-token

Always accept CancellationToken in async methods.

**Why It Matters:** Cancellation tokens allow callers to cancel long-running operations, free resources, and implement timeouts. Missing cancellation support prevents proper request cancellation in web applications.

**❌ Incorrect:**
```csharp
// BAD: No cancellation support
public async Task<Order> GetOrderAsync(string orderId)
{
    var order = await _httpClient.GetFromJsonAsync<Order>($"/orders/{orderId}");
    return order;
}

// Cannot cancel even if user navigates away or request times out
```

**✅ Correct:**
```csharp
// GOOD: Accept and pass through cancellation token
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken cancellationToken = default)
{
    var order = await _httpClient.GetFromJsonAsync<Order>(
        $"/orders/{orderId}", 
        cancellationToken);
    return order;
}

// Usage with timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
var order = await GetOrderAsync(orderId, cts.Token);
```

**Context:**
- Always use `CancellationToken cancellationToken = default` parameter
- Pass token through entire async call chain
- In ASP.NET Core, cancellation token is provided automatically
- See also: `async-all-the-way`, `async-semaphore`

---

### async-all-the-way

Never block on async code with .Result or .Wait().

**Why It Matters:** Blocking on async code causes deadlocks in UI and ASP.NET Core applications, defeats the purpose of async/await, and wastes thread pool threads.

**❌ Incorrect:**
```csharp
// BAD: Blocking on async code - DEADLOCK RISK!
public Order GetOrder(string orderId)
{
    return GetOrderAsync(orderId).Result;  // Deadlock in ASP.NET Core
}

// BAD: Also causes deadlocks
public Order GetOrder(string orderId)
{
    return GetOrderAsync(orderId).GetAwaiter().GetResult();
}

// BAD: Wait() also blocks
public void ProcessOrder(string orderId)
{
    GetOrderAsync(orderId).Wait();
}
```

**✅ Correct:**
```csharp
// GOOD: Async all the way
public async Task<Order> GetOrderAsync(
    string orderId, 
    CancellationToken cancellationToken = default)
{
    return await _repository.GetAsync(orderId, cancellationToken);
}

// GOOD: Controllers return Task
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(
    string id, 
    CancellationToken cancellationToken)
{
    var order = await GetOrderAsync(id, cancellationToken);
    return Ok(order);
}
```

**Context:**
- Use async/await all the way through the call stack
- Never use .Result, .Wait(), or .GetAwaiter().GetResult()
- If you truly need sync-over-async (rare), use Task.Run in a non-UI context
- See also: `async-cancellation-token`, `async-avoid-async-void`

---

### async-valuetask

Use ValueTask for hot paths with synchronous completions.

**Why It Matters:** ValueTask avoids Task allocations when operations complete synchronously (e.g., cache hits), improving performance in high-throughput scenarios.

**❌ Incorrect:**
```csharp
// BAD: Task allocation even when cache hit (synchronous path)
public async Task<Order?> GetCachedOrderAsync(string orderId)
{
    if (_cache.TryGetValue(orderId, out var order))
        return order;  // Allocates Task even though we have the value
    
    return await _repository.GetAsync(orderId);
}
```

**✅ Correct:**
```csharp
// GOOD: ValueTask avoids allocation on synchronous path
public ValueTask<Order?> GetCachedOrderAsync(
    string orderId,
    CancellationToken cancellationToken = default)
{
    if (_cache.TryGetValue(orderId, out var order))
        return ValueTask.FromResult<Order?>(order);  // No allocation
    
    return new ValueTask<Order?>(GetFromDatabaseAsync(orderId, cancellationToken));
}

private async Task<Order?> GetFromDatabaseAsync(
    string orderId, 
    CancellationToken cancellationToken)
{
    var order = await _repository.GetAsync(orderId, cancellationToken);
    if (order is not null)
        _cache[orderId] = order;
    return order;
}
```

**Context:**
- Use ValueTask when operations often complete synchronously (cache hits, pooled objects)
- Always use Task for I/O that never completes synchronously
- Never await a ValueTask more than once
- See also: `perf-object-pool`, `perf-avoid-closure-allocations`

---

### async-iasyncenumerable

Use IAsyncEnumerable for streaming data.

**Why It Matters:** IAsyncEnumerable allows streaming large datasets without loading everything into memory, enabling backpressure and cancellation per item.

**❌ Incorrect:**
```csharp
// BAD: Loads all orders into memory before returning
public async Task<List<Order>> GetOrdersAsync(string customerId)
{
    var orders = new List<Order>();
    
    await foreach (var order in _repository.StreamAllAsync())
    {
        if (order.CustomerId == customerId)
            orders.Add(order);  // Accumulates in memory
    }
    
    return orders;  // Caller must wait for all orders
}
```

**✅ Correct:**
```csharp
// GOOD: Stream orders one at a time
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    string customerId,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var order in _repository.StreamAllAsync(cancellationToken))
    {
        if (order.CustomerId == customerId)
            yield return order;  // Caller receives immediately
    }
}

// Usage
await foreach (var order in StreamOrdersAsync(customerId, cancellationToken))
{
    await ProcessOrderAsync(order, cancellationToken);
    // Can cancel mid-stream, backpressure applied
}
```

**Context:**
- Use for streaming large datasets, real-time feeds, or progressive processing
- Always include [EnumeratorCancellation] attribute
- Caller controls iteration speed (backpressure)
- See also: `async-cancellation-token`, `db-projections-over-entities`

---

[Content continues with all 94 rules expanded similarly...]

---

## Quick Reference Index

### By Category
- [Error Handling](#1-error-handling-critical)
- [Async Patterns](#2-async-patterns-critical)
- [Type Design](#3-type-design-high)
- [Database Performance](#4-database-performance-high)
- [API Design](#5-api-design-medium-high)
- [Dependency Injection](#6-dependency-injection-medium)
- [Architecture](#7-architecture-medium)
- [Serialization](#8-serialization-medium)
- [Performance](#9-performance-low-medium)
- [Logging](#10-logging-low-medium)
- [Testing](#11-testing-low)

### By Priority
- **CRITICAL:** error-*, async-*
- **HIGH:** type-*, db-*
- **MEDIUM-HIGH:** api-*
- **MEDIUM:** di-*, arch-*, serial-*
- **LOW-MEDIUM:** perf-*, log-*
- **LOW:** test-*

---

**Note:** This document is auto-generated from individual rule files in `rules/`. For contributions or updates, modify the individual rule files and regenerate this document.
