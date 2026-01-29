---
name: dotnet-extensions-configuration
description: Microsoft.Extensions.Options patterns including IValidateOptions, strongly-typed settings, validation on startup, and the Options pattern for clean configuration management.
---

# Microsoft.Extensions Configuration Patterns

## When to Use This Skill

Use this skill when:
- Binding configuration from appsettings.json to strongly-typed classes
- Validating configuration at application startup (fail fast)
- Implementing complex validation logic for settings
- Designing configuration classes that are testable and maintainable
- Understanding IOptions<T>, IOptionsSnapshot<T>, and IOptionsMonitor<T>

## Why Configuration Validation Matters

**The Problem:** Applications often fail at runtime due to misconfiguration - missing connection strings, invalid URLs, out-of-range values.

**The Solution:** Validate configuration at startup. If configuration is invalid, the application fails immediately with a clear error message. This is the "fail fast" principle.

```csharp
// BAD: Fails at runtime when someone tries to use the service
public class EmailService
{
    public EmailService(IOptions<SmtpSettings> options)
    {
        var settings = options.Value;
        // Throws NullReferenceException 10 minutes into production
        _client = new SmtpClient(settings.Host, settings.Port);
    }
}

// GOOD: Fails at startup with clear error
// "SmtpSettings validation failed: Host is required"
```

---

## Quick Reference

### Basic Options Binding

```csharp
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    
    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public bool UseSsl { get; set; } = true;
}

// Registration
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### Data Annotations Validation

```csharp
using System.ComponentModel.DataAnnotations;

public class SmtpSettings
{
    [Required(ErrorMessage = "SMTP host is required")]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int Port { get; set; } = 587;

    [EmailAddress]
    public string? Username { get; set; }
}
```

### Complex Validation with IValidateOptions

```csharp
public class SmtpSettingsValidator : IValidateOptions<SmtpSettings>
{
    public ValidateOptionsResult Validate(string? name, SmtpSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.Host))
            failures.Add("Host is required");

        // Cross-property validation
        if (!string.IsNullOrEmpty(options.Username) && string.IsNullOrEmpty(options.Password))
            failures.Add("Password is required when Username is specified");

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

// Register validator
builder.Services.AddSingleton<IValidateOptions<SmtpSettings>, SmtpSettingsValidator>();
```

---

## References

See detailed patterns in the `references/` folder:

- [Basic Options Binding](references/basic-binding.md) - Define settings, bind from config, consume in services
- [Data Annotations Validation](references/data-annotations.md) - Attribute-based validation rules
- [IValidateOptions](references/ivalidateoptions.md) - Complex validation logic
- [Validators with Dependencies](references/validators-with-dependencies.md) - Inject services into validators
- [Named Options](references/named-options.md) - Multiple instances of the same settings type
- [Options Lifetime](references/options-lifetime.md) - IOptions vs IOptionsSnapshot vs IOptionsMonitor
- [Testing Validators](references/testing-validators.md) - Unit testing configuration validators
- [Anti-Patterns](references/anti-patterns.md) - Common mistakes to avoid

## Resources

- **Microsoft.Extensions.Options**: https://learn.microsoft.com/en-us/dotnet/core/extensions/options
- **Configuration in .NET**: https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration
