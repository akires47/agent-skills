# Named Options

When you have multiple instances of the same settings type (e.g., multiple database connections).

## Configuration

```json
{
  "Databases": {
    "Primary": {
      "ConnectionString": "Server=primary;..."
    },
    "Replica": {
      "ConnectionString": "Server=replica;..."
    }
  }
}
```

## Registration

```csharp
builder.Services.AddOptions<DatabaseSettings>("Primary")
    .BindConfiguration("Databases:Primary")
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddOptions<DatabaseSettings>("Replica")
    .BindConfiguration("Databases:Replica")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## Consumption

```csharp
public class DataService
{
    private readonly DatabaseSettings _primary;
    private readonly DatabaseSettings _replica;

    public DataService(IOptionsSnapshot<DatabaseSettings> options)
    {
        _primary = options.Get("Primary");
        _replica = options.Get("Replica");
    }

    public async Task<User> GetUserAsync(string id)
    {
        // Read from replica
        return await QueryReplicaAsync(id);
    }

    public async Task SaveUserAsync(User user)
    {
        // Write to primary
        await WriteToPrimaryAsync(user);
    }
}
```

## Named Options Validator

```csharp
public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var failures = new List<string>();
        var prefix = string.IsNullOrEmpty(name) ? "" : $"[{name}] ";

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
        {
            failures.Add($"{prefix}ConnectionString is required");
        }

        // Name-specific validation
        if (name == "Primary" && options.ReadOnly)
        {
            failures.Add("Primary database cannot be read-only");
        }

        if (name == "Replica" && !options.ReadOnly)
        {
            failures.Add("Replica database should be read-only");
        }

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
```

## Using Default Name

```csharp
// Get default (unnamed) instance
var settings = options.Value;  // or options.Get(Options.DefaultName)

// Get named instance
var primary = options.Get("Primary");
```

## Multiple Connections Example

```csharp
public class MultiDbContext
{
    private readonly string _primaryConnection;
    private readonly string _analyticsConnection;

    public MultiDbContext(IOptionsSnapshot<ConnectionSettings> options)
    {
        _primaryConnection = options.Get("Primary").ConnectionString;
        _analyticsConnection = options.Get("Analytics").ConnectionString;
    }

    public DbContext GetPrimaryContext()
    {
        var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_primaryConnection);
        return new AppDbContext(optionsBuilder.Options);
    }

    public DbContext GetAnalyticsContext()
    {
        var optionsBuilder = new DbContextOptionsBuilder<AnalyticsDbContext>()
            .UseSqlServer(_analyticsConnection);
        return new AnalyticsDbContext(optionsBuilder.Options);
    }
}
```
