# Database Containers

## SQL Server Pattern

```csharp
using Testcontainers.MsSql;

public class SqlServerTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    private SqlConnection _connection;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        _connection = new SqlConnection(_sqlContainer.GetConnectionString());
        await _connection.OpenAsync();

        // Create schema
        await _connection.ExecuteAsync(@"
            CREATE TABLE Orders (
                Id INT PRIMARY KEY IDENTITY,
                CustomerId NVARCHAR(50) NOT NULL,
                Total DECIMAL(18,2) NOT NULL,
                CreatedAt DATETIME2 DEFAULT GETUTCDATE()
            )");
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task CanInsertAndRetrieveOrder()
    {
        // Arrange
        await _connection.ExecuteAsync(@"
            INSERT INTO Orders (CustomerId, Total)
            VALUES (@CustomerId, @Total)",
            new { CustomerId = "CUST001", Total = 99.99m });

        // Act
        var order = await _connection.QuerySingleAsync<Order>(
            "SELECT * FROM Orders WHERE CustomerId = @CustomerId",
            new { CustomerId = "CUST001" });

        // Assert
        Assert.Equal("CUST001", order.CustomerId);
        Assert.Equal(99.99m, order.Total);
    }
}
```

## PostgreSQL Pattern

```csharp
using Testcontainers.PostgreSql;

public class PostgreSqlTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgresContainer = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private NpgsqlConnection _connection;

    public async Task InitializeAsync()
    {
        await _postgresContainer.StartAsync();

        _connection = new NpgsqlConnection(_postgresContainer.GetConnectionString());
        await _connection.OpenAsync();

        // Create schema
        await _connection.ExecuteAsync(@"
            CREATE TABLE orders (
                id SERIAL PRIMARY KEY,
                customer_id VARCHAR(50) NOT NULL,
                total DECIMAL(18,2) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )");
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
        await _postgresContainer.DisposeAsync();
    }

    [Fact]
    public async Task CanInsertAndRetrieveOrder()
    {
        // Arrange
        await _connection.ExecuteAsync(@"
            INSERT INTO orders (customer_id, total)
            VALUES (@CustomerId, @Total)",
            new { CustomerId = "CUST001", Total = 99.99m });

        // Act
        var order = await _connection.QuerySingleAsync<Order>(
            "SELECT * FROM orders WHERE customer_id = @CustomerId",
            new { CustomerId = "CUST001" });

        // Assert
        Assert.Equal("CUST001", order.CustomerId);
        Assert.Equal(99.99m, order.Total);
    }
}
```

## With EF Core DbContext

```csharp
public class EfCoreTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    private AppDbContext _dbContext;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _dbContext = new AppDbContext(options);
        await _dbContext.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _dbContext.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task CanSaveAndQueryEntity()
    {
        // Arrange
        var order = new Order
        {
            CustomerId = "CUST001",
            Total = 99.99m
        };

        // Act
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();

        // Assert
        var retrieved = await _dbContext.Orders
            .FirstOrDefaultAsync(o => o.CustomerId == "CUST001");

        retrieved.Should().NotBeNull();
        retrieved.Total.Should().Be(99.99m);
    }
}
```

## Container Reuse Across Tests

For faster test execution, reuse containers within a test class:

```csharp
public class OrderRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task Test1()
    {
        // All tests share the same container
        var connection = _fixture.GetConnection();
        // Test code...
    }
}

public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private SqlConnection _connection;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();
        _connection = new SqlConnection(_sqlContainer.GetConnectionString());
        await _connection.OpenAsync();
        // Run migrations once
    }

    public SqlConnection GetConnection() => _connection;

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }
}
```
