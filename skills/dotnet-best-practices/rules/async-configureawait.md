# async-configureawait

Use ConfigureAwait(false) in library code to avoid capturing synchronization context, but not in application code.

## Why It Matters

ConfigureAwait(false) prevents deadlocks in libraries used from UI or ASP.NET contexts and improves performance by avoiding context capture. However, in ASP.NET Core and modern application code, it's usually unnecessary.

## ❌ Incorrect

```csharp
// BAD: ConfigureAwait(false) in ASP.NET Core controllers
public async Task<IActionResult> GetOrderAsync(string orderId)
{
    var order = await _repository.GetOrderAsync(orderId)
        .ConfigureAwait(false);  // Loses HttpContext!
    
    // Can't access HttpContext here - wrong thread
    var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier);
    return Ok(order);
}

// BAD: Inconsistent ConfigureAwait usage
public async Task<Order> ProcessOrderAsync(string orderId)
{
    var order = await GetOrderAsync(orderId);  // No ConfigureAwait
    await ValidateAsync(order).ConfigureAwait(false);  // Has ConfigureAwait
    await SaveAsync(order);  // No ConfigureAwait
    // Inconsistency makes it hard to reason about context
}

// BAD: Using ConfigureAwait(true) - it's the default!
public async Task ProcessAsync()
{
    await DoWorkAsync().ConfigureAwait(true);  // Redundant - this is default
}
```

## ✅ Correct

```csharp
// GOOD: Library code uses ConfigureAwait(false)
public sealed class OrderRepository
{
    public async Task<Order?> GetOrderAsync(
        string orderId,
        CancellationToken ct = default)
    {
        // Library code - no UI or ASP.NET context needed
        var connection = await _dataSource.OpenConnectionAsync(ct)
            .ConfigureAwait(false);
        
        var order = await connection.QuerySingleOrDefaultAsync<Order>(
            "SELECT * FROM orders WHERE id = @Id",
            new { Id = orderId })
            .ConfigureAwait(false);
        
        return order;
    }
}

// GOOD: Application code - no ConfigureAwait needed
public async Task<IActionResult> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct)
{
    var result = await CreateOrder.HandleAsync(request, _db, ct);
    
    return result.Match(
        onSuccess: order => Results.Created($"/orders/{order.Id}", order),
        onFailure: error => error.ToHttpResult());
}

// GOOD: ASP.NET Core doesn't need ConfigureAwait
public static async Task<Result<Order>> HandleAsync(
    CreateOrderRequest request,
    AppDbContext db,
    CancellationToken ct)
{
    // ASP.NET Core has no sync context - ConfigureAwait unnecessary
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    var order = ToEntity(request);
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);
    
    return Result.Success(order);
}

// GOOD: Consistent ConfigureAwait in library
public sealed class HttpApiClient
{
    public async Task<T> GetAsync<T>(
        string url,
        CancellationToken ct = default)
    {
        var response = await _httpClient.GetAsync(url, ct)
            .ConfigureAwait(false);
        
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync(ct)
            .ConfigureAwait(false);
        
        return JsonSerializer.Deserialize<T>(content)!;
    }
}
```

## Context

- **Library code**: Use `ConfigureAwait(false)` everywhere
- **ASP.NET Core**: Don't use ConfigureAwait - there's no synchronization context
- **Desktop UI (WPF/WinForms)**: Don't use ConfigureAwait - need UI thread
- **Blazor Server**: Don't use ConfigureAwait - need synchronization context
- Related: `async-all-the-way.md`, `async-avoid-async-void.md`
- Be consistent: all or nothing in a given method
