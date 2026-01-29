# error-global-handlers

Use global exception handlers for unexpected errors and logging, but don't use them for business logic errors.

## Why It Matters

Global handlers catch unexpected exceptions and prevent 500 errors from leaking implementation details. However, overusing them for expected errors hides control flow and makes debugging difficult. Expected errors should use the Result pattern.

## ❌ Incorrect

```csharp
// BAD: Using global handler for business logic
app.UseExceptionHandler(builder =>
{
    builder.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        
        // Don't handle business logic errors globally!
        if (exception is ValidationException ve)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { error = ve.Message });
        }
        else if (exception is NotFoundException nfe)
        {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new { error = nfe.Message });
        }
        // This hides error handling from the endpoint code
    });
});

// Endpoint has no idea it can throw ValidationException
app.MapPost("/orders", async (CreateOrderRequest request) =>
{
    var order = await orderService.CreateAsync(request);  // Can throw!
    return Results.Created($"/orders/{order.Id}", order);
});
```

## ✅ Correct

```csharp
// GOOD: Global handler only for unexpected errors
app.UseExceptionHandler(builder =>
{
    builder.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        
        logger.LogError(exception, "Unhandled exception occurred");
        
        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(new
        {
            error = "An unexpected error occurred",
            traceId = Activity.Current?.Id ?? context.TraceIdentifier
        });
    });
});

// Endpoint explicitly handles expected errors with Result pattern
app.MapPost("/orders", async (CreateOrderRequest request, AppDbContext db) =>
{
    var result = await CreateOrder.HandleAsync(request, db);
    
    return result.Match(
        onSuccess: order => Results.Created($"/orders/{order.Id}", order),
        onFailure: error => error.ToHttpResult());
});

// Handler returns Result for expected errors
public static async Task<Result<Order>> HandleAsync(
    CreateOrderRequest request,
    AppDbContext db)
{
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    // Business logic...
    return Result.Success(order);
}
```

## Context

- Global handlers are for **unexpected** exceptions (database down, out of memory)
- Use Result pattern for **expected** errors (validation, not found, business rules)
- Always log unexpected exceptions with correlation IDs
- Don't leak internal error details to clients in production
- Related: `error-result-pattern.md`, `error-never-catch-all.md`
- Use ProblemDetails for structured error responses
