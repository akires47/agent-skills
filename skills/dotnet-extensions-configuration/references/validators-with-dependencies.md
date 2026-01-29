# Validators with Dependencies

IValidateOptions validators are resolved from DI, so they can have dependencies.

## Validator with Logger

```csharp
public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    private readonly ILogger<DatabaseSettingsValidator> _logger;

    public DatabaseSettingsValidator(ILogger<DatabaseSettingsValidator> logger)
    {
        _logger = logger;
    }

    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
        {
            failures.Add("ConnectionString is required");
        }

        if (!options.ConnectionString?.Contains("Encrypt=True") == true)
        {
            _logger.LogWarning("Database connection should use encryption");
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Environment-Specific Validation

```csharp
public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    private readonly IHostEnvironment _environment;

    public DatabaseSettingsValidator(IHostEnvironment environment)
    {
        _environment = environment;
    }

    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
        {
            failures.Add("ConnectionString is required");
        }

        // Environment-specific validation
        if (_environment.IsProduction())
        {
            if (options.ConnectionString?.Contains("localhost") == true)
            {
                failures.Add("Production cannot use localhost database");
            }

            if (!options.ConnectionString?.Contains("Encrypt=True") == true)
            {
                failures.Add("Production database must use encryption");
            }
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Validator with External Service

```csharp
public class ApiSettingsValidator : IValidateOptions<ApiSettings>
{
    private readonly IHttpClientFactory _httpClientFactory;

    public ApiSettingsValidator(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public ValidateOptionsResult Validate(string? name, ApiSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.BaseUrl))
        {
            failures.Add("BaseUrl is required");
        }

        if (!Uri.TryCreate(options.BaseUrl, UriKind.Absolute, out _))
        {
            failures.Add($"BaseUrl '{options.BaseUrl}' is not a valid URL");
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Registration

```csharp
// Validators are registered as singletons
builder.Services.AddSingleton<IValidateOptions<DatabaseSettings>, DatabaseSettingsValidator>();
builder.Services.AddSingleton<IValidateOptions<ApiSettings>, ApiSettingsValidator>();
```

## Multiple Validators for Same Options

You can register multiple validators for the same options type:

```csharp
builder.Services.AddSingleton<IValidateOptions<SmtpSettings>, SmtpSettingsValidator>();
builder.Services.AddSingleton<IValidateOptions<SmtpSettings>, SmtpSecurityValidator>();

// Both validators will run
```
