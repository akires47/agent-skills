# Extension Method Patterns

## Basic Structure

```csharp
namespace MyApp.Users;

public static class UserServiceCollectionExtensions
{
    public static IServiceCollection AddUserServices(this IServiceCollection services)
    {
        // Repositories
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserReadStore, UserReadStore>();
        services.AddScoped<IUserWriteStore, UserWriteStore>();

        // Services
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IUserValidationService, UserValidationService>();

        // Return for chaining
        return services;
    }
}
```

## With Configuration

```csharp
namespace MyApp.Email;

public static class EmailServiceCollectionExtensions
{
    public static IServiceCollection AddEmailServices(
        this IServiceCollection services,
        string configSectionName = "EmailSettings")
    {
        // Bind configuration
        services.AddOptions<EmailOptions>()
            .BindConfiguration(configSectionName)
            .ValidateDataAnnotations()
            .ValidateOnStart();

        // Register services
        services.AddSingleton<IMjmlTemplateRenderer, MjmlTemplateRenderer>();
        services.AddSingleton<IEmailLinkGenerator, EmailLinkGenerator>();
        services.AddScoped<IUserEmailComposer, UserEmailComposer>();
        services.AddScoped<IOrderEmailComposer, OrderEmailComposer>();
        services.AddScoped<IEmailSender, SmtpEmailSender>();

        return services;
    }
}
```

## With Dependencies on Other Extensions

```csharp
namespace MyApp.Orders;

public static class OrderServiceCollectionExtensions
{
    public static IServiceCollection AddOrderServices(this IServiceCollection services)
    {
        // This subsystem depends on email services
        // Caller is responsible for calling AddEmailServices() first

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IOrderEmailNotifier, OrderEmailNotifier>();

        return services;
    }
}
```

## File Organization

Place extension methods near the services they register:

```
src/
  MyApp.Api/
    Program.cs                    # Composes all Add* methods
  MyApp.Users/
    Services/
      UserService.cs
      IUserService.cs
    Repositories/
      UserRepository.cs
    UserServiceCollectionExtensions.cs   # AddUserServices()
  MyApp.Orders/
    Services/
      OrderService.cs
    OrderServiceCollectionExtensions.cs  # AddOrderServices()
```

**Convention**: `{Feature}ServiceCollectionExtensions.cs` next to the feature's services.

## Naming Conventions

| Pattern | Use For |
|---------|---------|
| `Add{Feature}Services()` | General feature registration |
| `Add{Feature}()` | Short form when unambiguous |
| `Configure{Feature}()` | When primarily setting options |
| `Use{Feature}()` | Middleware (on IApplicationBuilder) |

```csharp
// Feature services
services.AddUserServices();
services.AddEmailServices();
services.AddPaymentServices();

// Third-party integrations
services.AddStripePayments();
services.AddSendGridEmail();

// Configuration-heavy
services.ConfigureAuthentication();
services.ConfigureAuthorization();
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
        services.AddSingleton<IEmailSender, MailpitEmailSender>();
    }
    else
    {
        services.AddSingleton<IEmailSender, SmtpEmailSender>();
    }

    return services;
}
```

## Factory-Based Registration

```csharp
public static IServiceCollection AddPaymentServices(
    this IServiceCollection services,
    string configSection = "Stripe")
{
    services.AddOptions<StripeOptions>()
        .BindConfiguration(configSection)
        .ValidateOnStart();

    // Factory for complex initialization
    services.AddSingleton<IPaymentProcessor>(sp =>
    {
        var options = sp.GetRequiredService<IOptions<StripeOptions>>().Value;
        var logger = sp.GetRequiredService<ILogger<StripePaymentProcessor>>();

        return new StripePaymentProcessor(options.ApiKey, options.WebhookSecret, logger);
    });

    return services;
}
```

## Keyed Services (.NET 8+)

```csharp
public static IServiceCollection AddNotificationServices(this IServiceCollection services)
{
    // Register multiple implementations with keys
    services.AddKeyedSingleton<INotificationSender, EmailNotificationSender>("email");
    services.AddKeyedSingleton<INotificationSender, SmsNotificationSender>("sms");
    services.AddKeyedSingleton<INotificationSender, PushNotificationSender>("push");

    // Resolver that picks the right one
    services.AddScoped<INotificationDispatcher, NotificationDispatcher>();

    return services;
}
```
