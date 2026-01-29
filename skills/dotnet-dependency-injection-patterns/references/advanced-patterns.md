# Advanced Patterns

## Layered Extensions

For larger applications, compose extensions hierarchically:

```csharp
// Top-level: Everything the app needs
public static class AppServiceCollectionExtensions
{
    public static IServiceCollection AddAppServices(this IServiceCollection services)
    {
        return services
            .AddDomainServices()
            .AddInfrastructureServices()
            .AddApiServices();
    }
}

// Domain layer
public static class DomainServiceCollectionExtensions
{
    public static IServiceCollection AddDomainServices(this IServiceCollection services)
    {
        return services
            .AddUserServices()
            .AddOrderServices()
            .AddProductServices();
    }
}

// Infrastructure layer
public static class InfrastructureServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructureServices(this IServiceCollection services)
    {
        return services
            .AddEmailServices()
            .AddPaymentServices()
            .AddStorageServices();
    }
}

// Usage in Program.cs
builder.Services.AddAppServices();
```

## Keyed Services (.NET 8+)

Register multiple implementations of the same interface:

```csharp
public static IServiceCollection AddNotificationServices(this IServiceCollection services)
{
    // Register multiple implementations with keys
    services.AddKeyedSingleton<INotificationSender, EmailNotificationSender>("email");
    services.AddKeyedSingleton<INotificationSender, SmsNotificationSender>("sms");
    services.AddKeyedSingleton<INotificationSender, PushNotificationSender>("push");

    // Resolver service
    services.AddScoped<INotificationDispatcher, NotificationDispatcher>();

    return services;
}

// Consuming keyed services
public class NotificationDispatcher : INotificationDispatcher
{
    private readonly INotificationSender _emailSender;
    private readonly INotificationSender _smsSender;

    public NotificationDispatcher(
        [FromKeyedServices("email")] INotificationSender emailSender,
        [FromKeyedServices("sms")] INotificationSender smsSender)
    {
        _emailSender = emailSender;
        _smsSender = smsSender;
    }

    public async Task SendAsync(string channel, string message)
    {
        var sender = channel switch
        {
            "email" => _emailSender,
            "sms" => _smsSender,
            _ => throw new ArgumentException($"Unknown channel: {channel}")
        };

        await sender.SendAsync(message);
    }
}
```

## Conditional Registration

```csharp
public static IServiceCollection AddEmailServices(
    this IServiceCollection services,
    IHostEnvironment environment)
{
    services.AddSingleton<IEmailComposer, MjmlEmailComposer>();

    if (environment.IsDevelopment())
    {
        // Use Mailpit in development
        services.AddSingleton<IEmailSender, MailpitEmailSender>();
    }
    else
    {
        // Use real SMTP in production
        services.AddSingleton<IEmailSender, SmtpEmailSender>();
    }

    return services;
}
```

## Options Pattern Integration

```csharp
public static IServiceCollection AddPaymentServices(
    this IServiceCollection services,
    IConfiguration configuration)
{
    // Bind and validate options
    services.AddOptions<StripeOptions>()
        .BindConfiguration("Stripe")
        .ValidateDataAnnotations()
        .ValidateOnStart();

    // Register services that depend on options
    services.AddSingleton<IPaymentProcessor, StripePaymentProcessor>();
    services.AddScoped<IPaymentService, PaymentService>();

    return services;
}
```

## TryAdd Methods

Use TryAdd to make registrations idempotent:

```csharp
public static IServiceCollection AddCoreServices(this IServiceCollection services)
{
    // Only registers if not already registered
    services.TryAddScoped<IUserRepository, UserRepository>();
    services.TryAddScoped<IOrderRepository, OrderRepository>();

    return services;
}

// Safe to call multiple times
services.AddCoreServices();
services.AddCoreServices(); // Doesn't duplicate registrations
```

## Decorator Pattern

```csharp
public static IServiceCollection AddCachedOrderService(this IServiceCollection services)
{
    // Register base service
    services.AddScoped<OrderService>();

    // Wrap with caching decorator
    services.AddScoped<IOrderService>(sp =>
    {
        var innerService = sp.GetRequiredService<OrderService>();
        var cache = sp.GetRequiredService<IMemoryCache>();
        return new CachedOrderService(innerService, cache);
    });

    return services;
}

public class CachedOrderService : IOrderService
{
    private readonly IOrderService _inner;
    private readonly IMemoryCache _cache;

    public CachedOrderService(IOrderService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        var cacheKey = $"order:{id}";

        if (_cache.TryGetValue(cacheKey, out Order? cached))
            return cached!;

        var order = await _inner.GetOrderAsync(id);
        _cache.Set(cacheKey, order, TimeSpan.FromMinutes(5));
        return order;
    }
}
```

## Composite Services

Register multiple implementations and inject all:

```csharp
public static IServiceCollection AddNotificationHandlers(this IServiceCollection services)
{
    services.AddScoped<INotificationHandler, EmailNotificationHandler>();
    services.AddScoped<INotificationHandler, SmsNotificationHandler>();
    services.AddScoped<INotificationHandler, PushNotificationHandler>();

    return services;
}

// Inject all implementations
public class NotificationService
{
    private readonly IEnumerable<INotificationHandler> _handlers;

    public NotificationService(IEnumerable<INotificationHandler> handlers)
    {
        _handlers = handlers; // All registered handlers
    }

    public async Task SendToAllChannelsAsync(string message)
    {
        foreach (var handler in _handlers)
        {
            await handler.SendAsync(message);
        }
    }
}
```
