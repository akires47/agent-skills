# Data Annotations Validation

For simple validation rules, use Data Annotations:

## Using Validation Attributes

```csharp
using System.ComponentModel.DataAnnotations;

public class SmtpSettings
{
    public const string SectionName = "Smtp";

    [Required(ErrorMessage = "SMTP host is required")]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535, ErrorMessage = "Port must be between 1 and 65535")]
    public int Port { get; set; } = 587;

    [EmailAddress(ErrorMessage = "Username must be a valid email address")]
    public string? Username { get; set; }

    public string? Password { get; set; }

    public bool UseSsl { get; set; } = true;
}
```

## Enable Validation

```csharp
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName)
    .ValidateDataAnnotations()  // Enable attribute-based validation
    .ValidateOnStart();         // Validate immediately at startup
```

**Key Point:** `.ValidateOnStart()` is critical. Without it, validation only runs when options are first accessed, which could be minutes or hours into application runtime.

## Available Validation Attributes

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `[Required]` | Field must be non-null | `[Required] public string Host { get; set; }` |
| `[Range(min, max)]` | Numeric range | `[Range(1, 65535)] public int Port { get; set; }` |
| `[EmailAddress]` | Valid email format | `[EmailAddress] public string Email { get; set; }` |
| `[Url]` | Valid URL format | `[Url] public string ApiUrl { get; set; }` |
| `[RegularExpression]` | Regex pattern | `[RegularExpression(@"^\d{3}-\d{3}-\d{4}$")]` |
| `[StringLength]` | String length | `[StringLength(100, MinimumLength = 3)]` |
| `[MinLength]` | Min collection size | `[MinLength(1)] public string[] Items { get; set; }` |
| `[MaxLength]` | Max collection size | `[MaxLength(10)] public string[] Items { get; set; }` |

## Custom Validation Attributes

```csharp
public class ValidUrlAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is not string url)
            return ValidationResult.Success;

        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
            return new ValidationResult("Invalid URL format");

        if (uri.Scheme != "https")
            return new ValidationResult("URL must use HTTPS");

        return ValidationResult.Success;
    }
}

// Usage
public class ApiSettings
{
    [ValidUrl]
    public string BaseUrl { get; set; } = string.Empty;
}
```

## When Data Annotations Are Enough

Use Data Annotations for:
- ✅ Required fields
- ✅ Range checks
- ✅ Format validation (email, URL, regex)
- ✅ Length constraints
- ✅ Simple custom validation

Use IValidateOptions for:
- ❌ Cross-property validation
- ❌ Conditional validation
- ❌ Environment-specific validation
- ❌ Validation requiring external services or DI

See [IValidateOptions](ivalidateoptions.md) for complex scenarios.
