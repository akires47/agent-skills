# Feature Examples

Complete examples for common operations.

## Get Single Entity

```csharp
// Features/Products/GetProduct.cs
public static class GetProduct
{
    public sealed record Response(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        int id, AppDbContext db, CancellationToken ct)
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

## List with Filtering and Pagination

```csharp
// Features/Products/GetProducts.cs
public static class GetProducts
{
    public sealed record Request(string? Category, int Page = 1, int PageSize = 10);
    public sealed record Response(IReadOnlyList<ProductDto> Items, int TotalCount, int Page, int PageSize);
    public sealed record ProductDto(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        Request request, AppDbContext db, CancellationToken ct)
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
        .WithTags("Products");
}
```

## Update

```csharp
// Features/Products/UpdateProduct.cs
public static class UpdateProduct
{
    public sealed record Request(string Name, decimal Price, string Category);
    public sealed record Response(int Id, string Name, decimal Price, string Category);

    public static async Task<Result<Response>> HandleAsync(
        int id, Request request, AppDbContext db, CancellationToken ct)
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
            .GreaterThan(request.Price, 0, "Price");

    public static void MapEndpoint(IEndpointRouteBuilder app) => app
        .MapPut("/api/products/{id:int}", async (int id, Request request, AppDbContext db, CancellationToken ct) =>
            (await HandleAsync(id, request, db, ct)).ToResponse())
        .WithName("UpdateProduct")
        .WithTags("Products");
}
```

## Delete

```csharp
// Features/Products/DeleteProduct.cs
public static class DeleteProduct
{
    public static async Task<r> HandleAsync(int id, AppDbContext db, CancellationToken ct)
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
        .WithTags("Products");
}
```

## Manual Mapper

```csharp
// Features/Products/ProductMapper.cs
public static class ProductMapper
{
    public static GetProducts.ProductDto ToDto(this Product product) =>
        new(product.Id, product.Name, product.Price, product.Category);

    public static Product ToEntity(this CreateProduct.Request request) =>
        new() { Name = request.Name, Price = request.Price, Category = request.Category };

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

## Program.cs

```csharp
using MyApp.Shared.Extensions;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDbContext>();
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.UseSwaggerUI(options => options.SwaggerEndpoint("/openapi/v1.json", "API v1"));
}

app.UseHttpsRedirection();
app.MapFeatureEndpoints();
app.Run();
```
