---
name: dotnet-dependency-injection-patterns
description: Organize DI registrations using IServiceCollection extension methods. Group related services into composable Add* methods for clean Program.cs and reusable configuration in tests.
---

# Dependency Injection Patterns

## When to Use This Skill

Use this skill when:
- Organizing service registrations in ASP.NET Core applications
- Avoiding massive Program.cs/Startup.cs files with hundreds of registrations
- Making service configuration reusable between production and tests
- Designing libraries that integrate with Microsoft.Extensions.DependencyInjection

---

## The Problem

Without organization, Program.cs becomes unmanageable:

```csharp
// BAD: 200+ lines of unorganized registrations
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IUserService, UserService>();
// ... 150 more lines ...
```

Problems:
- Hard to find related registrations
- No clear boundaries between subsystems
- Can't reuse configuration in tests
- Merge conflicts in team settings

---

## The Solution: Extension Method Composition

Group related registrations into extension methods:

```csharp
// GOOD: Clean, composable Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddUserServices()
    .AddOrderServices()
    .AddEmailServices()
    .AddPaymentServices()
    .AddValidators();

var app = builder.Build();
```

Each `Add*` method encapsulates a cohesive set of registrations.

---

## Quick Reference

### Basic Extension Method

```csharp
namespace MyApp.Users;

public static class UserServiceCollectionExtensions
{
    public static IServiceCollection AddUserServices(this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserService, UserService>();
        
        return services; // Return for chaining
    }
}
```

### With Configuration

```csharp
public static IServiceCollection AddEmailServices(
    this IServiceCollection services,
    string configSectionName = "EmailSettings")
{
    services.AddOptions<EmailOptions>()
        .BindConfiguration(configSectionName)
        .ValidateOnStart();

    services.AddScoped<IEmailSender, SmtpEmailSender>();
    
    return services;
}
```

### Testing Benefits

```csharp
public class UserServiceTests
{
    private readonly ServiceProvider _provider;

    public UserServiceTests()
    {
        var services = new ServiceCollection();
        
        // Reuse production registrations
        services.AddUserServices();
        
        // Add test infrastructure
        services.AddSingleton<IUserRepository, InMemoryUserRepository>();
        
        _provider = services.BuildServiceProvider();
    }
}
```

---

## References

See detailed patterns in the `references/` folder:

- [Extension Method Patterns](references/extension-method-patterns.md) - Basic structure, configuration, dependencies
- [Lifetime Management](references/lifetime-management.md) - Singleton, Scoped, Transient, scope creation
- [Testing Patterns](references/testing-patterns.md) - WebApplicationFactory, standalone tests, reusing production config
- [Common Mistakes](references/common-mistakes.md) - Scoped in singleton, missing scopes in background services
- [Advanced Patterns](references/advanced-patterns.md) - Layered extensions, keyed services, conditional registration

## Resources

- **Microsoft.Extensions.DependencyInjection**: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection
