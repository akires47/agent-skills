# Result Pattern

All operations return Result types instead of throwing exceptions.

## Error Type

```csharp
// Shared/Results/Error.cs
namespace MyApp.Shared.Results;

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    
    public static Error NotFound(string entity, object id) => 
        new($"{entity}.NotFound", $"{entity} with id '{id}' was not found");
    
    public static Error Validation(string code, string message) => 
        new(code, message);
    
    public static Error Conflict(string message) => 
        new("Error.Conflict", message);
}
```

## Result Types

```csharp
// Shared/Results/Result.cs
namespace MyApp.Shared.Results;

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, Error.None);
    public static Result<T> Failure<T>(Error error) => new(default, false, error);
}

public class Result<T> : Result
{
    private readonly T? _value;

    internal Result(T? value, bool isSuccess, Error error) : base(isSuccess, error)
    {
        _value = value;
    }

    public T Value => IsSuccess ? _value! : default!;

    public static implicit operator Result<T>(T value) => Success(value);
}
```

## HTTP Response Mapping

```csharp
// Shared/Extensions/ResultExtensions.cs
namespace MyApp.Shared.Extensions;

public static class ResultExtensions
{
    public static IResult ToResponse(this Result result) =>
        result.IsSuccess
            ? Results.NoContent()
            : result.Error.ToProblemResult();

    public static IResult ToResponse<T>(this Result<T> result) =>
        result.IsSuccess
            ? Results.Ok(result.Value)
            : result.Error.ToProblemResult();

    public static IResult ToCreatedResponse<T>(this Result<T> result, Func<T, string> uriFactory) =>
        result.IsSuccess
            ? Results.Created(uriFactory(result.Value), result.Value)
            : result.Error.ToProblemResult();

    private static IResult ToProblemResult(this Error error)
    {
        var statusCode = error.Code switch
        {
            var code when code.EndsWith(".NotFound") => StatusCodes.Status404NotFound,
            var code when code.StartsWith("Error.Conflict") => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status400BadRequest
        };

        return Results.Problem(
            title: statusCode switch
            {
                StatusCodes.Status404NotFound => "Not Found",
                StatusCodes.Status409Conflict => "Conflict",
                _ => "Bad Request"
            },
            detail: error.Message,
            statusCode: statusCode,
            extensions: new Dictionary<string, object?> { ["code"] = error.Code });
    }
}
```

## Usage Pattern

```csharp
// Return success
return Result.Success(new Response(...));
return new Response(...); // implicit conversion

// Return failure
return Result.Failure<Response>(Error.NotFound("Product", id));
return Result.Failure<Response>(Error.Validation("Name.Empty", "Name is required"));
```
