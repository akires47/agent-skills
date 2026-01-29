# IValidateOptions for Complex Validation

Data Annotations work for simple rules, but complex validation requires `IValidateOptions<T>`.

## When to Use IValidateOptions

| Scenario | Data Annotations | IValidateOptions |
|----------|------------------|------------------|
| Required field | ✅ | ✅ |
| Range check | ✅ | ✅ |
| Regex pattern | ✅ | ✅ |
| Cross-property validation | ❌ | ✅ |
| Conditional validation | ❌ | ✅ |
| External service checks | ❌ | ✅ |
| Custom error messages with context | Limited | ✅ |
| Dependency injection in validator | ❌ | ✅ |

## Implementing IValidateOptions

```csharp
using Microsoft.Extensions.Options;

public class SmtpSettingsValidator : IValidateOptions<SmtpSettings>
{
    public ValidateOptionsResult Validate(string? name, SmtpSettings options)
    {
        var failures = new List<string>();

        // Required field validation
        if (string.IsNullOrWhiteSpace(options.Host))
        {
            failures.Add("Host is required");
        }

        // Range validation
        if (options.Port is < 1 or > 65535)
        {
            failures.Add($"Port {options.Port} is invalid. Must be between 1 and 65535");
        }

        // Cross-property validation
        if (!string.IsNullOrEmpty(options.Username) && string.IsNullOrEmpty(options.Password))
        {
            failures.Add("Password is required when Username is specified");
        }

        // Conditional validation
        if (options.UseSsl && options.Port == 25)
        {
            failures.Add("Port 25 is typically not used with SSL. Consider port 465 or 587");
        }

        // Return result
        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Register the Validator

```csharp
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName)
    .ValidateDataAnnotations()  // Run attribute validation first
    .ValidateOnStart();

// Register the custom validator
builder.Services.AddSingleton<IValidateOptions<SmtpSettings>, SmtpSettingsValidator>();
```

**Order matters:** Data Annotations run first, then IValidateOptions validators. All failures are collected and reported together.

## Complex Validation Example

```csharp
public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
        {
            failures.Add("ConnectionString is required");
        }

        // Validate connection string format
        if (!string.IsNullOrEmpty(options.ConnectionString))
        {
            try
            {
                var builder = new SqlConnectionStringBuilder(options.ConnectionString);
                if (string.IsNullOrEmpty(builder.DataSource))
                {
                    failures.Add("ConnectionString must specify a Data Source");
                }
            }
            catch (Exception ex)
            {
                failures.Add($"ConnectionString is malformed: {ex.Message}");
            }
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Best Practices

### ✅ DO

- Collect all failures before returning
- Provide descriptive error messages
- Validate cross-property relationships
- Use try-catch for format validation
- Return `ValidateOptionsResult.Fail(failures)` for errors

### ❌ DON'T

- Throw exceptions (return failures instead)
- Return on first failure (collect all)
- Perform expensive operations (keep validation fast)
- Mutate options in validator
