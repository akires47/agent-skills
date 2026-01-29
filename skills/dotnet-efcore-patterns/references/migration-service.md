# Migration Service Pattern

## .NET Aspire Integration

Create a dedicated migration service to run migrations on startup:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.AddSqlServerDbContext<AppDbContext>("database");

var app = builder.Build();

// Run migrations in dedicated service
await app.RunDatabaseMigrationsAsync<AppDbContext>();

app.Run();
```

## Custom Migration Service

```csharp
public static class MigrationExtensions
{
    public static async Task RunDatabaseMigrationsAsync<TContext>(
        this WebApplication app)
        where TContext : DbContext
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<TContext>();
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<TContext>>();

        try
        {
            logger.LogInformation("Starting database migration...");

            var pendingMigrations = await context.Database.GetPendingMigrationsAsync();

            if (pendingMigrations.Any())
            {
                logger.LogInformation(
                    "Applying {Count} pending migrations: {Migrations}",
                    pendingMigrations.Count(),
                    string.Join(", ", pendingMigrations));

                await context.Database.MigrateAsync();

                logger.LogInformation("Database migration completed successfully");
            }
            else
            {
                logger.LogInformation("Database is up to date");
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "An error occurred while migrating the database");
            throw;
        }
    }
}
```

## With Retry Policy

```csharp
public static async Task RunDatabaseMigrationsWithRetryAsync<TContext>(
    this WebApplication app,
    int maxRetries = 5)
    where TContext : DbContext
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<TContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<TContext>>();

    var retry = 0;
    while (retry < maxRetries)
    {
        try
        {
            await context.Database.MigrateAsync();
            logger.LogInformation("Database migration completed");
            return;
        }
        catch (Exception ex) when (retry < maxRetries - 1)
        {
            retry++;
            logger.LogWarning(ex,
                "Migration attempt {Retry}/{MaxRetries} failed. Retrying...",
                retry, maxRetries);
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retry)));
        }
    }
}
```

## Separate Migration Project

For larger applications, create a separate migration runner:

```csharp
// MyApp.Migrator/Program.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Infrastructure;

var connectionString = args.Length > 0
    ? args[0]
    : throw new ArgumentException("Connection string required");

var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
optionsBuilder.UseSqlServer(connectionString);

await using var context = new AppDbContext(optionsBuilder.Options);

Console.WriteLine("Running migrations...");
await context.Database.MigrateAsync();
Console.WriteLine("Migrations completed successfully");
```

Run before app startup:

```bash
dotnet run --project MyApp.Migrator -- "Server=...;Database=MyApp;..."
```

## Docker Entrypoint Pattern

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet build -c Release -o /app/build
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
COPY entrypoint.sh .
RUN chmod +x entrypoint.sh

ENTRYPOINT ["./entrypoint.sh"]
```

```bash
#!/bin/bash
# entrypoint.sh

echo "Running database migrations..."
dotnet MyApp.Migrator.dll "$ConnectionString"

echo "Starting application..."
dotnet MyApp.dll
```
