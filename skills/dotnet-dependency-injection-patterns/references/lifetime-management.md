# Lifetime Management

Choose the right lifetime based on state and usage patterns.

## Lifetime Types

| Lifetime | Use When | Examples |
|----------|----------|----------|
| **Singleton** | Stateless, thread-safe, expensive to create | Configuration, HttpClient factories, caches |
| **Scoped** | Stateful per-request, database contexts | DbContext, repositories, user context |
| **Transient** | Lightweight, stateful, cheap to create | Validators, short-lived helpers |

## Rules of Thumb

```csharp
// SINGLETON: Stateless services, shared safely
services.AddSingleton<IMjmlTemplateRenderer, MjmlTemplateRenderer>();
services.AddSingleton<IEmailLinkGenerator, EmailLinkGenerator>();

// SCOPED: Database access, per-request state
services.AddScoped<IUserRepository, UserRepository>();  // DbContext dependency
services.AddScoped<IOrderService, OrderService>();       // Uses scoped repos

// TRANSIENT: Cheap, short-lived
services.AddTransient<CreateUserRequestValidator>();
```

## Scope Requirements

**Scoped services require a scope to exist.** In ASP.NET Core, each HTTP request creates a scope automatically. But in other contexts, you must create scopes manually.

### ASP.NET Controller (Automatic Scope)

```csharp
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;  // Scoped - works!

    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }
}
```

### Background Service (Manual Scope)

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;

    public OrderProcessingService(IServiceProvider serviceProvider)
    {
        // Inject IServiceProvider, NOT scoped services directly
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // Create scope manually for each unit of work
            using var scope = _serviceProvider.CreateScope();
            var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();

            await orderService.ProcessPendingOrdersAsync(ct);
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

## Using IServiceScopeFactory

Alternative to IServiceProvider for creating scopes:

```csharp
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

## Scope Per Message Pattern

For message handlers or actors that process multiple messages:

```csharp
public sealed class MessageHandler
{
    private readonly IServiceProvider _serviceProvider;

    public MessageHandler(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task HandleMessageAsync(OrderMessage message)
    {
        // Create scope for this message processing
        using var scope = _serviceProvider.CreateScope();

        // Resolve scoped services
        var orderRepository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var emailComposer = scope.ServiceProvider.GetRequiredService<IEmailComposer>();

        // Do work with scoped services
        var order = await orderRepository.GetAsync(message.OrderId);
        await emailComposer.SendConfirmationAsync(order);

        // DbContext commits when scope disposes
    }
}
```

## Why This Pattern Works

1. **Each unit of work gets fresh DbContext** - No stale entity tracking
2. **Proper disposal** - Connections released after each operation
3. **Isolation** - One operation's errors don't affect others
4. **Testable** - Can inject mock IServiceProvider
