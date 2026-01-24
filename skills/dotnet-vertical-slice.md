---
name: dotnet-vertical-slice
description: .NET 10 development using vertical slice (feature) architecture with minimal APIs. Use this skill when building .NET web APIs that prioritize feature cohesion over technical layer separation, keeping all code for a feature together in one place.
---

# .NET 10 Vertical Slice Architecture with Minimal APIs

Build .NET 10 applications organized by feature rather than technical layers. Each feature contains its own endpoint, handler, request/response models, and validation—everything needed for that feature lives together.

## Project Structure

```
src/
├── Features/
│   ├── Products/
│   │   ├── GetProduct.cs
│   │   ├── GetProducts.cs
│   │   ├── CreateProduct.cs
│   │   ├── UpdateProduct.cs
│   │   ├── DeleteProduct.cs
│   │   └── ProductMapper.cs
│   ├── Orders/
│   │   ├── GetOrder.cs
│   │   ├── CreateOrder.cs
│   │   └── OrderMapper.cs
│   └── Users/
│       └── ...
├── Shared/
│   ├── Results/
│   │   ├── Result.cs
│   │   └── Error.cs
│   ├── Validation/
│   │   └── ValidationExtensions.cs
│   └── Extensions/
│       └── EndpointExtensions.cs
├── Entities/
│   ├── Product.cs
│   ├── Order.cs
│   └── User.cs
└── Program.cs
```

## Result Pattern

All operations return a Result type instead of throwing exceptions:

```csharp
// Shared/Results/Error.cs
namespace MyApp.Shared.Results;

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static readonly Error NullValue = new("Error.NullValue", "A null value was provided");
    
    public static Error NotFound(string entity, object id) => 
        new($"{entity}.NotFound", $"{entity} with id '{id}' was not found");
    
    public static Error Validation(string code, string message) => 
        new(code, message);
    
    public static Error Conflict(string message) => 
        new("Error.Conflict", message);
}
```

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

    public T Value => IsSuccess
        ? _value!
        : default!;

    public static implicit operator Result<T>(T value) => Success(value);
}
```

## Validation with Result Pattern

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

    public Result<T> ToResult<T>(Func<T> valueFactory) => IsValid 
        ? Result.Success(valueFactory()) 
        : Result.Failure<T>(_errors.First());
}
```

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

## Result to HTTP Response Mapping

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

    public static IResult ToCreatedResponse<T>(this Result<T> result, string uri) =>
        result.IsSuccess
            ? Results.Created(uri, result.Value)
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
            title: GetTitle(statusCode),
            detail: error.Message,
            statusCode: statusCode,
            extensions: new Dictionary<string, object?> { ["code"] = error.Code });
    }

    private static string GetTitle(int statusCode) => statusCode switch
    {
        StatusCodes.Status404NotFound => "Not Found",
        StatusCodes.Status409Conflict => "Conflict",
        _ => "Bad Request"
    };
}
```

## Feature Slice Pattern

Each feature file is self-contained with endpoint, request, response, validation, and handler:

```csharp
// Features/Products/GetProduct.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Shared.Results;
using MyApp.Shared.Extensions;

namespace MyApp.Features.Products;

public static class GetProduct
{
    public sealed record Response(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        int id,
        AppDbContext db,
        CancellationToken ct)
    {
        var product = await db.Products
            .Where(p => p.Id == id)
            .Select(p => new Response(p.Id, p.Name, p.Price, p.Category))
            .FirstOrDefaultAsync(ct);

        return product is not null
            ? Result.Success(product)
            : Result.Failure<Response>(Error.NotFound("Product", id));
    }

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapGet("/api/products/{id:int}", async (int id, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(id, db, ct)).ToResponse())
        .WithName("GetProduct")
        .WithTags("Products")
        .Produces<Response>()
        .ProducesProblem(StatusCodes.Status404NotFound);
}
```

## Feature with Validation

```csharp
// Features/Products/CreateProduct.cs
using MyApp.Shared.Results;
using MyApp.Shared.Validation;
using MyApp.Shared.Extensions;

namespace MyApp.Features.Products;

public static class CreateProduct
{
    public sealed record Request(string Name, decimal Price, string Category);
    public sealed record Response(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        Request request,
        AppDbContext db,
        CancellationToken ct)
    {
        var validation = Validate(request);
        if (!validation.IsValid)
            return validation.ToResult<Response>(null!);

        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category
        };

        db.Products.Add(product);
        await db.SaveChangesAsync(ct);

        return new Response(product.Id, product.Name, product.Price, product.Category);
    }

    private static ValidationResult Validate(Request request) =>
        ValidationExtensions.Validate()
            .NotEmpty(request.Name, "Name")
            .MaxLength(request.Name, 200, "Name")
            .GreaterThan(request.Price, 0, "Price")
            .NotEmpty(request.Category, "Category");

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapPost("/api/products", async (Request request, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(request, db, ct)).ToCreatedResponse(r => $"/api/products/{r.Id}"))
        .WithName("CreateProduct")
        .WithTags("Products")
        .Produces<Response>(StatusCodes.Status201Created)
        .ProducesProblem(StatusCodes.Status400BadRequest);
}
```

## Update Feature

```csharp
// Features/Products/UpdateProduct.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Shared.Results;
using MyApp.Shared.Validation;
using MyApp.Shared.Extensions;

namespace MyApp.Features.Products;

public static class UpdateProduct
{
    public sealed record Request(string Name, decimal Price, string Category);
    public sealed record Response(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        int id,
        Request request,
        AppDbContext db,
        CancellationToken ct)
    {
        var validation = Validate(request);
        if (!validation.IsValid)
            return validation.ToResult<Response>(null!);

        var product = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
        
        if (product is null)
            return Result.Failure<Response>(Error.NotFound("Product", id));

        product.Name = request.Name;
        product.Price = request.Price;
        product.Category = request.Category;

        await db.SaveChangesAsync(ct);

        return new Response(product.Id, product.Name, product.Price, product.Category);
    }

    private static ValidationResult Validate(Request request) =>
        ValidationExtensions.Validate()
            .NotEmpty(request.Name, "Name")
            .MaxLength(request.Name, 200, "Name")
            .GreaterThan(request.Price, 0, "Price")
            .NotEmpty(request.Category, "Category");

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapPut("/api/products/{id:int}", async (int id, Request request, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(id, request, db, ct)).ToResponse())
        .WithName("UpdateProduct")
        .WithTags("Products")
        .Produces<Response>()
        .ProducesProblem(StatusCodes.Status400BadRequest)
        .ProducesProblem(StatusCodes.Status404NotFound);
}
```

## Delete Feature

```csharp
// Features/Products/DeleteProduct.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Shared.Results;
using MyApp.Shared.Extensions;

namespace MyApp.Features.Products;

public static class DeleteProduct
{
    public static async Task<Result> HandleAsync(
        int id,
        AppDbContext db,
        CancellationToken ct)
    {
        var rowsAffected = await db.Products
            .Where(p => p.Id == id)
            .ExecuteDeleteAsync(ct);

        return rowsAffected > 0
            ? Result.Success()
            : Result.Failure(Error.NotFound("Product", id));
    }

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapDelete("/api/products/{id:int}", async (int id, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(id, db, ct)).ToResponse())
        .WithName("DeleteProduct")
        .WithTags("Products")
        .Produces(StatusCodes.Status204NoContent)
        .ProducesProblem(StatusCodes.Status404NotFound);
}
```

## List with Filtering

```csharp
// Features/Products/GetProducts.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Shared.Results;
using MyApp.Shared.Extensions;

namespace MyApp.Features.Products;

public static class GetProducts
{
    public sealed record Request(string? Category, int Page = 1, int PageSize = 10);
    public sealed record Response(IReadOnlyList<ProductDto> Items, int TotalCount, int Page, int PageSize);
    public sealed record ProductDto(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        Request request,
        AppDbContext db,
        CancellationToken ct)
    {
        var query = db.Products.AsQueryable();

        if (!string.IsNullOrWhiteSpace(request.Category))
            query = query.Where(p => p.Category == request.Category);

        var totalCount = await query.CountAsync(ct);

        var items = await query
            .OrderBy(p => p.Name)
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category))
            .ToListAsync(ct);

        return new Response(items, totalCount, request.Page, request.PageSize);
    }

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapGet("/api/products", async ([AsParameters] Request request, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(request, db, ct)).ToResponse())
        .WithName("GetProducts")
        .WithTags("Products")
        .Produces<Response>();
}
```

## Manual Mapping (No AutoMapper)

```csharp
// Features/Products/ProductMapper.cs
namespace MyApp.Features.Products;

public static class ProductMapper
{
    public static GetProducts.ProductDto ToDto(this Product product) =>
        new(product.Id, product.Name, product.Price, product.Category);

    public static IReadOnlyList<GetProducts.ProductDto> ToDtos(this IEnumerable<Product> products) =>
        products.Select(ToDto).ToList();

    public static Product ToEntity(this CreateProduct.Request request) =>
        new()
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category
        };

    public static void UpdateFrom(this Product product, UpdateProduct.Request request)
    {
        product.Name = request.Name;
        product.Price = request.Price;
        product.Category = request.Category;
    }
}
```

## Endpoint Registration

```csharp
// Shared/Extensions/EndpointExtensions.cs
using System.Reflection;

namespace MyApp.Shared.Extensions;

public static class EndpointExtensions
{
    public static IEndpointRouteBuilder MapFeatureEndpoints(this IEndpointRouteBuilder app)
    {
        var endpointTypes = Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => t.GetMethod("MapEndpoint", BindingFlags.Public | BindingFlags.Static) is not null);

        foreach (var type in endpointTypes)
        {
            var mapMethod = type.GetMethod("MapEndpoint", BindingFlags.Public | BindingFlags.Static);
            mapMethod?.Invoke(null, [app]);
        }

        return app;
    }
}
```

## Program.cs Setup

```csharp
using MyApp.Shared.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add DbContext (see EF Core skill for configuration)
builder.Services.AddDbContext<AppDbContext>();

// OpenAPI
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.UseSwaggerUI(options => options.SwaggerEndpoint("/openapi/v1.json", "API v1"));
}

app.UseHttpsRedirection();

// Map all feature endpoints
app.MapFeatureEndpoints();

app.Run();
```

## Required NuGet Packages

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.*" />
  <PackageReference Include="Swashbuckle.AspNetCore.SwaggerUI" Version="7.*" />
</ItemGroup>
```

## Examples

- **Simple CRUD API**: Create a feature folder for each entity with Get, GetAll, Create, Update, Delete slices
- **Complex business operation**: Single feature file containing all logic for "PlaceOrder" including validation, inventory check, payment processing
- **Query with filtering**: Feature slice with query parameters using `[AsParameters]` attribute
- **Bulk operations**: Feature slice handling batch imports returning `Result<BulkImportResponse>` with success/failure counts

## Guidelines

- One feature = one file containing endpoint, request/response records, validation, and handler
- Name files by the operation: `CreateProduct.cs`, `GetProducts.cs`, `UpdateProductPrice.cs`
- Keep entities in a shared folder; they're the only cross-cutting concern
- Always return `Result<T>` or `Result` from handlers, never throw exceptions
- Validation returns early with `Result.Failure<T>()` containing validation errors
- Use extension methods in `ResultExtensions` to convert results to HTTP responses
- Prefer projections (`.Select()`) over loading full entities when only reading
- Use static classes with static methods for feature slices (no instance state needed)
- Write manual mapping methods in `*Mapper.cs` files per feature folder
- Group related endpoints with `.WithTags()` for OpenAPI organization
- Keep `Program.cs` minimal—push configuration into extension methods
- See EF Core skill for database configuration
- See Authentication skill for securing endpoints
- See Integration Tests skill for testing patterns
