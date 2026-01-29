# Troubleshooting TestContainers

## Common Issues and Solutions

### Issue: Docker Not Running

**Error:** `Cannot connect to Docker daemon`

**Solution:**
```bash
# Ensure Docker is running
docker ps

# On Windows, start Docker Desktop
# On Linux, start Docker service
sudo systemctl start docker
```

### Issue: Port Already in Use

**Error:** `Bind for 0.0.0.0:5432 failed: port is already allocated`

**Solution:**
TestContainers automatically assigns random ports. Don't specify fixed ports:

```csharp
// BAD
new MsSqlBuilder().WithPortBinding(1433, 1433)

// GOOD - Let TestContainers assign random port
new MsSqlBuilder().Build()

// Get the assigned port
var port = _container.GetMappedPublicPort(1433);
```

### Issue: Container Startup Timeout

**Error:** `Container did not start within timeout`

**Solution:**
Increase wait timeout or use proper wait strategies:

```csharp
new MsSqlBuilder()
    .WithWaitStrategy(Wait.ForUnixContainer()
        .UntilCommandIsCompleted("/opt/mssql-tools/bin/sqlcmd", "-Q", "SELECT 1")
        .WithTimeout(TimeSpan.FromMinutes(2)))
    .Build();
```

### Issue: Slow Tests

**Problem:** Tests take too long to run

**Solution 1:** Reuse containers across tests in a class

```csharp
public class Tests : IClassFixture<DatabaseFixture>
{
    // Container reused for all tests in this class
}
```

**Solution 2:** Use Respawn instead of recreating containers

```csharp
// Fast data cleanup
await _respawner.ResetAsync(_connection);
```

**Solution 3:** Run migrations only once

```csharp
public class DatabaseFixture
{
    public async Task InitializeAsync()
    {
        // Migrations run once for all tests
        await _dbContext.Database.MigrateAsync();
    }
}
```

### Issue: Tests Pass Locally, Fail in CI

**Common causes:**
1. Docker not available in CI environment
2. Insufficient memory/resources
3. Firewall blocking container ports

**Solution:**
```yaml
# GitHub Actions example
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      # Use service containers instead of TestContainers in CI
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
```

Or ensure Docker-in-Docker is available:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Start Docker
        run: sudo systemctl start docker
      - name: Run tests
        run: dotnet test
```

### Issue: Container Logs Not Visible

**Solution:**
Enable container logging:

```csharp
new MsSqlBuilder()
    .WithOutputConsumer(Consume.RedirectStdoutAndStderrToConsole())
    .Build();

// Or capture to variable
var logs = new StringBuilder();
new MsSqlBuilder()
    .WithOutputConsumer(Consume.RedirectStdoutAndStderrToStream(
        new StringWriter(logs)))
    .Build();
```

### Issue: Permission Denied Errors

**Error:** `Got permission denied while trying to connect to the Docker daemon socket`

**Solution (Linux):**
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Restart session
newgrp docker
```

### Issue: Image Pull Failures

**Error:** `Failed to pull image`

**Solution:**
```csharp
// Pre-pull images in CI setup
docker pull mcr.microsoft.com/mssql/server:2022-latest
docker pull postgres:16-alpine
docker pull redis:7-alpine

// Or use specific image versions
new MsSqlBuilder()
    .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
    .Build();
```

## Performance Tips

1. **Reuse containers** with `IClassFixture`
2. **Use Respawn** for data cleanup
3. **Pull images** before running tests
4. **Use Alpine images** (smaller, faster startup)
5. **Parallel test execution** with separate container instances
6. **Connection pooling** when possible

## Debugging Tests

```csharp
[Fact]
public async Task DebugContainerTest()
{
    var container = new MsSqlBuilder()
        .WithOutputConsumer(Consume.RedirectStdoutAndStderrToConsole())
        .Build();

    await container.StartAsync();

    // Get connection details
    var connectionString = container.GetConnectionString();
    Console.WriteLine($"Connection: {connectionString}");

    // Keep container running for manual inspection
    await Task.Delay(TimeSpan.FromMinutes(5));

    await container.DisposeAsync();
}
```

Then connect manually with SQL Server Management Studio or psql to inspect state.
