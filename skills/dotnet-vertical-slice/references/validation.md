# Validation

Fluent validation that integrates with the Result pattern.

## ValidationResult Class

```csharp
// Shared/Validation/ValidationResult.cs
namespace MyApp.Shared.Validation;

public sealed class ValidationResult
{
    private readonly List<Error> _errors = [];
    
    public IReadOnlyList<Error> Errors => _errors;
    public bool IsValid => _errors.Count == 0;

    public ValidationResult AddError(string code, string message)
    {
        _errors.Add(Error.Validation(code, message));
        return this;
    }

    public ValidationResult AddErrorIf(bool condition, string code, string message)
    {
        if (condition) _errors.Add(Error.Validation(code, message));
        return this;
    }

    public Result ToResult() => IsValid 
        ? Result.Success() 
        : Result.Failure(_errors.First());

    public Result<T> ToResult<T>(T value) => IsValid 
        ? Result.Success(value) 
        : Result.Failure<T>(_errors.First());
}
```

## Fluent Extensions

```csharp
// Shared/Validation/ValidationExtensions.cs
namespace MyApp.Shared.Validation;

public static class ValidationExtensions
{
    public static ValidationResult Validate() => new();

    public static ValidationResult NotEmpty(this ValidationResult result, string? value, string field) =>
        result.AddErrorIf(string.IsNullOrWhiteSpace(value), $"{field}.Empty", $"{field} is required");

    public static ValidationResult MaxLength(this ValidationResult result, string? value, int max, string field) =>
        result.AddErrorIf(value?.Length > max, $"{field}.TooLong", $"{field} must not exceed {max} characters");

    public static ValidationResult GreaterThan(this ValidationResult result, decimal value, decimal min, string field) =>
        result.AddErrorIf(value <= min, $"{field}.TooSmall", $"{field} must be greater than {min}");

    public static ValidationResult MustBeTrue(this ValidationResult result, bool condition, string code, string message) =>
        result.AddErrorIf(!condition, code, message);
}
```

## Usage in Handlers

```csharp
public static async Task<Result<Response>> HandleAsync(Request request, ...)
{
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Response>(null!);

    // Continue with business logic...
}

private static ValidationResult Validate(Request request) =>
    ValidationExtensions.Validate()
        .NotEmpty(request.Name, "Name")
        .MaxLength(request.Name, 200, "Name")
        .GreaterThan(request.Price, 0, "Price");
```
