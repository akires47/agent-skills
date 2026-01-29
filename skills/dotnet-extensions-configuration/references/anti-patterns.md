# Configuration Anti-Patterns

## 1. Manual Configuration Access

```csharp
// ❌ BAD: Bypasses validation, hard to test
public class MyService
{
    public MyService(IConfiguration configuration)
    {
        var host = configuration["Smtp:Host"]; // No validation!
    }
}

// ✅ GOOD: Strongly-typed, validated
public class MyService
{
    public MyService(IOptions<SmtpSettings> options)
    {
        var host = options.Value.Host; // Validated at startup
    }
}
```

## 2. Validation in Constructor

```csharp
// ❌ BAD: Validation happens at runtime, not startup
public class MyService
{
    public MyService(IOptions<Settings> options)
    {
        if (string.IsNullOrEmpty(options.Value.Required))
            throw new ArgumentException("Required is missing"); // Too late!
    }
}

// ✅ GOOD: Validation at startup
builder.Services.AddOptions<Settings>()
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## 3. Forgetting ValidateOnStart

```csharp
// ❌ BAD: Validation only runs when first accessed
builder.Services.AddOptions<Settings>()
    .ValidateDataAnnotations(); // Missing ValidateOnStart!

// ✅ GOOD: Fails immediately if invalid
builder.Services.AddOptions<Settings>()
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## 4. Throwing in IValidateOptions

```csharp
// ❌ BAD: Throws exception, breaks validation chain
public ValidateOptionsResult Validate(string? name, Settings options)
{
    if (options.Value < 0)
        throw new ArgumentException("Value cannot be negative"); // Wrong!

    return ValidateOptionsResult.Success;
}

// ✅ GOOD: Return failure result
public ValidateOptionsResult Validate(string? name, Settings options)
{
    if (options.Value < 0)
        return ValidateOptionsResult.Fail("Value cannot be negative");

    return ValidateOptionsResult.Success;
}
```

## 5. Not Collecting All Failures

```csharp
// ❌ BAD: Returns on first failure
public ValidateOptionsResult Validate(string? name, Settings options)
{
    if (string.IsNullOrEmpty(options.Host))
        return ValidateOptionsResult.Fail("Host required");

    if (options.Port < 1)
        return ValidateOptionsResult.Fail("Invalid port"); // Never reached if Host empty

    return ValidateOptionsResult.Success;
}

// ✅ GOOD: Collect all failures
public ValidateOptionsResult Validate(string? name, Settings options)
{
    var failures = new List<string>();

    if (string.IsNullOrEmpty(options.Host))
        failures.Add("Host required");

    if (options.Port < 1)
        failures.Add("Invalid port");

    return failures.Count > 0
        ? ValidateOptionsResult.Fail(failures)
        : ValidateOptionsResult.Success;
}
```

## 6. Hardcoding Configuration Values

```csharp
// ❌ BAD: Hardcoded in extension method
public static IServiceCollection AddDatabase(this IServiceCollection services)
{
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer("Server=localhost;..."));  // Hardcoded!
}

// ✅ GOOD: Accept from configuration
public static IServiceCollection AddDatabase(
    this IServiceCollection services,
    IConfiguration configuration)
{
    services.AddOptions<DatabaseSettings>()
        .BindConfiguration("Database")
        .ValidateOnStart();

    services.AddDbContext<AppDbContext>((sp, options) =>
    {
        var settings = sp.GetRequiredService<IOptions<DatabaseSettings>>().Value;
        options.UseSqlServer(settings.ConnectionString);
    });

    return services;
}
```

## 7. Using Wrong Interface

```csharp
// ❌ BAD: IOptionsSnapshot in singleton service
public class SingletonService
{
    public SingletonService(IOptionsSnapshot<Settings> options)
    {
        // IOptionsSnapshot is scoped - can't inject into singleton!
    }
}

// ✅ GOOD: Use IOptions or IOptionsMonitor in singletons
public class SingletonService
{
    public SingletonService(IOptions<Settings> options)
    {
        // IOptions works in singleton
    }
}
```

## 8. Not Using Constants for Section Names

```csharp
// ❌ BAD: Magic strings
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration("Smtp");

var host = configuration["Smtp:Host"];

// ✅ GOOD: Constant in settings class
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    public string Host { get; set; }
}

builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName);

var host = configuration[$"{SmtpSettings.SectionName}:Host"];
```
