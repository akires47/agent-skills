# Database Reset with Respawn

Respawn provides fast database cleanup between tests by intelligently deleting data while preserving schema.

## Installation

```bash
dotnet add package Respawn
```

## Basic Respawn Pattern

```csharp
using Respawn;

public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private SqlConnection _connection;
    private Respawner _respawner;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();
        _connection = new SqlConnection(_sqlContainer.GetConnectionString());
        await _connection.OpenAsync();

        // Run migrations
        await RunMigrationsAsync(_connection);

        // Initialize Respawn
        _respawner = await Respawner.CreateAsync(_connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.SqlServer,
            TablesToIgnore = new Respawn.Graph.Table[]
            {
                "__EFMigrationsHistory"
            }
        });
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task Test1_CreatesOrder()
    {
        // Arrange
        var repo = new OrderRepository(_connection);

        // Act
        await repo.CreateAsync(new Order { CustomerId = "C1", Total = 100m });

        // Assert
        var orders = await repo.GetAllAsync();
        orders.Should().HaveCount(1);

        // Cleanup for next test
        await _respawner.ResetAsync(_connection);
    }

    [Fact]
    public async Task Test2_AlsoCreatesOrder()
    {
        // This test starts with a clean database thanks to Respawn
        var repo = new OrderRepository(_connection);

        await repo.CreateAsync(new Order { CustomerId = "C2", Total = 200m });

        var orders = await repo.GetAllAsync();
        orders.Should().HaveCount(1); // Only this test's data

        await _respawner.ResetAsync(_connection);
    }
}
```

## With xUnit IAsyncLifetime Per Test

```csharp
public class OrderTests : IClassFixture<DatabaseFixture>, IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;

    public OrderTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    public Task InitializeAsync() => Task.CompletedTask;

    public async Task DisposeAsync()
    {
        // Reset database after each test
        await _fixture.ResetDatabaseAsync();
    }

    [Fact]
    public async Task CreateOrder_Test1()
    {
        // Database is clean at start
        var repo = new OrderRepository(_fixture.Connection);
        await repo.CreateAsync(new Order { Id = 1 });
        // Automatic cleanup after test
    }

    [Fact]
    public async Task CreateOrder_Test2()
    {
        // Database is clean again
        var repo = new OrderRepository(_fixture.Connection);
        await repo.CreateAsync(new Order { Id = 2 });
        // Automatic cleanup after test
    }
}

public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private Respawner _respawner;

    public SqlConnection Connection { get; private set; }

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();
        Connection = new SqlConnection(_sqlContainer.GetConnectionString());
        await Connection.OpenAsync();

        // Run migrations
        await RunMigrationsAsync();

        _respawner = await Respawner.CreateAsync(Connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.SqlServer
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await _respawner.ResetAsync(Connection);
    }

    public async Task DisposeAsync()
    {
        await Connection.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }
}
```

## Respawn Options

```csharp
var respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
{
    DbAdapter = DbAdapter.SqlServer, // or PostgreSql, MySql
    TablesToIgnore = new[]
    {
        new Respawn.Graph.Table("__EFMigrationsHistory"),
        new Respawn.Graph.Table("dbo", "VersionInfo")
    },
    SchemasToInclude = new[] { "dbo" },
    TablesToInclude = new[] { new Respawn.Graph.Table("Orders") },
    CheckTemporalTables = true // For SQL Server temporal tables
});
```

## Why Respawn vs Transactions?

| Approach | Speed | Isolation | Complex Scenarios |
|----------|-------|-----------|-------------------|
| **Respawn** | ⚡ Fast | ✅ Perfect | ✅ Works with everything |
| **Transactions + Rollback** | 🐌 Slower | ⚠️ Can leak | ❌ Breaks distributed scenarios |
| **Recreate Container** | 🐌🐌 Very Slow | ✅ Perfect | ✅ Works, but slow |

Respawn is the best balance of speed and reliability.
