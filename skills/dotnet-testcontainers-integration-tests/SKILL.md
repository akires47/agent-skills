---
name: dotnet-testcontainers-integration-tests
description: Write integration tests using TestContainers for .NET with xUnit. Covers infrastructure testing with real databases, message queues, and caches in Docker containers instead of mocks.
---

# Integration Testing with TestContainers

## When to Use This Skill

Use this skill when:
- Writing integration tests that need real infrastructure (databases, caches, message queues)
- Testing data access layers against actual databases
- Verifying message queue integrations
- Testing Redis caching behavior
- Avoiding mocks for infrastructure components
- Ensuring tests work against production-like environments
- Testing database migrations and schema changes

## Core Principles

1. **Real Infrastructure Over Mocks** - Use actual databases/services in containers, not mocks
2. **Test Isolation** - Each test gets fresh containers or fresh data
3. **Automatic Cleanup** - TestContainers handles container lifecycle and cleanup
4. **Fast Startup** - Reuse containers across tests in the same class when appropriate
5. **CI/CD Compatible** - Works seamlessly in Docker-enabled CI environments
6. **Port Randomization** - Containers use random ports to avoid conflicts

## Why TestContainers Over Mocks?

### ❌ Problems with Mocking Infrastructure

Mocking infrastructure components doesn't test real behavior:

```csharp
// BAD: Mocking a database
_mockDb.Setup(db => db.QueryAsync<Order>(It.IsAny<string>()))
    .ReturnsAsync(new[] { new Order { Id = 1 } });

// Doesn't test:
// - Real SQL syntax and constraints
// - Performance and indexes
// - Transactions and concurrency
// - Database-specific features
```

### ✅ Benefits of Real Containers

- Test actual database behavior (constraints, triggers, transactions)
- Catch SQL syntax errors before production
- Test migration scripts
- Verify performance characteristics
- Match production environment closely

---

## Quick Start

### Installation

```bash
dotnet add package Testcontainers.PostgreSql
dotnet add package Testcontainers.MsSql
dotnet add package Testcontainers.Redis
dotnet add package Testcontainers.RabbitMq
```

### Basic SQL Server Test

```csharp
public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task CreateOrder_SavesSuccessfully()
    {
        // Arrange
        var connectionString = _sqlContainer.GetConnectionString();
        var repository = new OrderRepository(connectionString);
        var order = new Order { CustomerId = "C123", Total = 99.99m };

        // Act
        await repository.CreateAsync(order);

        // Assert
        var retrieved = await repository.GetAsync(order.Id);
        retrieved.Should().NotBeNull();
        retrieved.Total.Should().Be(99.99m);
    }
}
```

---

## References

See detailed patterns and examples in the `references/` folder:

- [Database Containers](references/database-containers.md) - SQL Server, PostgreSQL patterns and setup
- [Cache Containers](references/cache-containers.md) - Redis integration testing
- [Message Queue Containers](references/message-queue-containers.md) - RabbitMQ patterns
- [Multi-Container Networks](references/multi-container-networks.md) - Testing services that communicate
- [Database Reset with Respawn](references/database-reset-respawn.md) - Fast data cleanup between tests
- [Migration Testing](references/migration-testing.md) - Testing EF Core migrations in containers
- [Troubleshooting](references/troubleshooting.md) - Common issues and solutions

## Resources

- **TestContainers for .NET**: https://dotnet.testcontainers.org/
- **Respawn**: https://github.com/jbogard/Respawn
- **Docker**: https://www.docker.com/
