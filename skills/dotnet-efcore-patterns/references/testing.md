# Testing EF Core Applications

## In-Memory Database (Quick Unit Tests)

Good for simple tests, but has limitations:

```csharp
public class OrderServiceTests
{
    private readonly AppDbContext _db;

    public OrderServiceTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        _db = new AppDbContext(options);
    }

    [Fact]
    public async Task CreateOrder_SavesSuccessfully()
    {
        // Arrange
        var service = new OrderService(_db);
        var order = new Order { CustomerId = "C1", Total = 100m };

        // Act
        await service.CreateOrderAsync(order);

        // Assert
        var saved = await _db.Orders.FirstOrDefaultAsync(o => o.CustomerId == "C1");
        saved.Should().NotBeNull();
    }
}
```

### Limitations of In-Memory Database

- No relational constraints (foreign keys don't work)
- No SQL-specific features (triggers, stored procedures)
- Different behavior than real databases
- Can give false confidence

## TestContainers (Real Integration Tests)

Use TestContainers for testing against real databases:

```csharp
using Testcontainers.MsSql;

public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private AppDbContext _db;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task ForeignKeyConstraint_EnforcedByDatabase()
    {
        // Arrange
        var order = new Order { CustomerId = "INVALID", Total = 100m };

        // Act & Assert
        _db.Orders.Add(order);
        var act = async () => await _db.SaveChangesAsync();

        await act.Should().ThrowAsync<DbUpdateException>()
            .WithMessage("*FOREIGN KEY constraint*");
    }
}
```

## Testing Migrations

```csharp
public class MigrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private AppDbContext _db;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _db = new AppDbContext(options);
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task AllMigrations_ApplyWithoutError()
    {
        // Act
        await _db.Database.MigrateAsync();

        // Assert
        var pending = await _db.Database.GetPendingMigrationsAsync();
        pending.Should().BeEmpty();
    }
}
```

## When to Use Each Approach

| Test Type | In-Memory | TestContainers |
|-----------|-----------|----------------|
| Quick unit tests | ✅ Good | ❌ Overkill |
| Foreign key constraints | ❌ Not enforced | ✅ Enforced |
| SQL-specific features | ❌ Not supported | ✅ Fully supported |
| Migration testing | ❌ Can't test | ✅ Perfect |
| Performance testing | ❌ Unrealistic | ✅ Realistic |
| Complex queries | ⚠️ May differ | ✅ Accurate |

**Recommendation:** Use In-Memory for simple service tests, TestContainers for repository and integration tests.
