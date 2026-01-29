# Common Mistakes

## 1. Injecting Scoped into Singleton

```csharp
// ❌ BAD: Singleton captures scoped service - stale DbContext!
public class CacheService  // Registered as Singleton
{
    private readonly IUserRepository _repo;  // Scoped!

    public CacheService(IUserRepository repo)  // Captured at startup!
    {
        _repo = repo;  // This DbContext lives forever - BAD
    }
}

// ✅ GOOD: Inject factory or IServiceProvider
public class CacheService
{
    private readonly IServiceProvider _serviceProvider;

    public CacheService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task<User> GetUserAsync(string id)
    {
        using var scope = _serviceProvider.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IUserRepository>();
        return await repo.GetByIdAsync(id);
    }
}
```

## 2. No Scope in Background Work

```csharp
// ❌ BAD: No scope for scoped services
public class BadBackgroundService : BackgroundService
{
    private readonly IOrderService _orderService;  // Scoped!

    public BadBackgroundService(IOrderService orderService)
    {
        _orderService = orderService;  // Will throw or behave unexpectedly
    }
}

// ✅ GOOD: Create scope for each unit of work
public class GoodBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public GoodBackgroundService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();
        // ...
    }
}
```

## 3. Registering Everything in Program.cs

```csharp
// ❌ BAD: Massive Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
// ... 200 more lines ...
```

Use extension methods instead (see extension-method-patterns.md).

## 4. Overly Generic Extensions

```csharp
// ❌ BAD: Too vague, doesn't communicate what's registered
public static IServiceCollection AddServices(this IServiceCollection services)
{
    // Registers 50 random things
}

// ✅ GOOD: Specific, descriptive names
public static IServiceCollection AddOrderServices(this IServiceCollection services)
{
    // Clear what this registers
}
```

## 5. Hiding Important Configuration

```csharp
// ❌ BAD: Buried important settings
public static IServiceCollection AddDatabase(this IServiceCollection services)
{
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer("hardcoded-connection-string"));  // Hidden!
}

// ✅ GOOD: Accept configuration explicitly
public static IServiceCollection AddDatabase(
    this IServiceCollection services,
    string connectionString)
{
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(connectionString));
}
```

## 6. Disposing DbContext Manually

```csharp
// ❌ BAD: Manual disposal
public class OrderService
{
    public async Task ProcessOrder(int orderId)
    {
        using var db = new AppDbContext();  // DON'T!
        // ...
    }
}

// ✅ GOOD: Inject from DI container
public class OrderService
{
    private readonly AppDbContext _db;

    public OrderService(AppDbContext db)
    {
        _db = db;  // Container manages lifecycle
    }
}
```

## 7. Registering Same Type Multiple Times

```csharp
// ❌ CONFUSING: Which implementation wins?
services.AddScoped<IEmailSender, SmtpEmailSender>();
services.AddScoped<IEmailSender, SendGridEmailSender>();  // This wins!

// ✅ GOOD: Use keyed services for multiple implementations
services.AddKeyedScoped<IEmailSender, SmtpEmailSender>("smtp");
services.AddKeyedScoped<IEmailSender, SendGridEmailSender>("sendgrid");
```

## 8. Not Validating Options on Startup

```csharp
// ❌ BAD: Validation only runs when first accessed
services.AddOptions<SmtpSettings>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations();  // Missing ValidateOnStart!

// ✅ GOOD: Fails immediately if invalid
services.AddOptions<SmtpSettings>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()
    .ValidateOnStart();  // Fail fast!
```
