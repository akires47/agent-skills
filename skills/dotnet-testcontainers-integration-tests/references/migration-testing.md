# Migration Testing

## Testing EF Core Migrations

```csharp
public class MigrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private AppDbContext _dbContext;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _dbContext = new AppDbContext(options);
    }

    public async Task DisposeAsync()
    {
        await _dbContext.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task AllMigrations_ApplySuccessfully()
    {
        // Act
        await _dbContext.Database.MigrateAsync();

        // Assert
        var pendingMigrations = await _dbContext.Database.GetPendingMigrationsAsync();
        pendingMigrations.Should().BeEmpty();
    }

    [Fact]
    public async Task CanInsertDataAfterMigration()
    {
        // Arrange
        await _dbContext.Database.MigrateAsync();

        // Act
        _dbContext.Orders.Add(new Order { CustomerId = "C1", Total = 100m });
        await _dbContext.SaveChangesAsync();

        // Assert
        var count = await _dbContext.Orders.CountAsync();
        count.Should().Be(1);
    }
}
```

## Testing Migration Rollback

```csharp
[Fact]
public async Task CanRollbackMigration()
{
    // Apply all migrations
    await _dbContext.Database.MigrateAsync();

    var migrations = await _dbContext.Database.GetAppliedMigrationsAsync();
    var secondToLast = migrations.Reverse().Skip(1).First();

    // Rollback to previous migration
    await _dbContext.Database.MigrateAsync(secondToLast);

    var current = await _dbContext.Database.GetAppliedMigrationsAsync();
    current.Should().NotContain(migrations.Last());
}
```

## Testing Data Migration Scripts

```csharp
[Fact]
public async Task DataMigration_MigratesExistingData()
{
    // Arrange - Create old schema
    await _connection.ExecuteAsync(@"
        CREATE TABLE OldOrders (
            OrderId INT PRIMARY KEY,
            CustomerName NVARCHAR(100)
        )");

    await _connection.ExecuteAsync(@"
        INSERT INTO OldOrders (OrderId, CustomerName)
        VALUES (1, 'John Doe')");

    // Act - Run migration that transforms data
    await _dbContext.Database.MigrateAsync();

    // Assert - New schema has migrated data
    var order = await _dbContext.Orders.FindAsync(1);
    order.Should().NotBeNull();
    order.Customer.Name.Should().Be("John Doe");
}
```

## Testing Schema Changes Don't Break Queries

```csharp
[Fact]
public async Task AddingNewColumn_DoesNotBreakExistingQueries()
{
    // Arrange - Start with older migration
    await _dbContext.Database.MigrateAsync("20231001_InitialCreate");

    // Insert data with old schema
    await _connection.ExecuteAsync(@"
        INSERT INTO Orders (CustomerId, Total)
        VALUES ('C1', 100.00)");

    // Act - Apply migration that adds new column
    await _dbContext.Database.MigrateAsync("20231015_AddOrderStatus");

    // Assert - Old data still queryable, new column has default
    var orders = await _dbContext.Orders.ToListAsync();
    orders.Should().HaveCount(1);
    orders[0].Status.Should().Be(OrderStatus.Pending); // Default value
}
```
