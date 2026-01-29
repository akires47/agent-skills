# Multi-Container Networks

## Testing Microservices Communication

When testing services that communicate with each other, use Docker networks:

```csharp
public class MicroserviceTests : IAsyncLifetime
{
    private readonly INetwork _network = new NetworkBuilder()
        .WithName(Guid.NewGuid().ToString("D"))
        .Build();

    private readonly PostgreSqlContainer _dbContainer;
    private readonly RabbitMqContainer _rabbitContainer;
    private readonly RedisContainer _cacheContainer;

    public MicroserviceTests()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithNetwork(_network)
            .WithNetworkAliases("postgres")
            .Build();

        _rabbitContainer = new RabbitMqBuilder()
            .WithNetwork(_network)
            .WithNetworkAliases("rabbitmq")
            .Build();

        _cacheContainer = new RedisBuilder()
            .WithNetwork(_network)
            .WithNetworkAliases("redis")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _network.CreateAsync();
        await Task.WhenAll(
            _dbContainer.StartAsync(),
            _rabbitContainer.StartAsync(),
            _cacheContainer.StartAsync()
        );
    }

    public async Task DisposeAsync()
    {
        await Task.WhenAll(
            _dbContainer.DisposeAsync().AsTask(),
            _rabbitContainer.DisposeAsync().AsTask(),
            _cacheContainer.DisposeAsync().AsTask()
        );
        await _network.DeleteAsync();
    }

    [Fact]
    public async Task ServicesCanCommunicate()
    {
        // All containers can reach each other by network alias
        // postgres://postgres:5432
        // rabbitmq://rabbitmq:5672
        // redis://redis:6379
    }
}
```

## Testing API with Database

```csharp
public class ApiWithDatabaseTests : IAsyncLifetime
{
    private readonly INetwork _network = new NetworkBuilder().Build();
    private readonly PostgreSqlContainer _dbContainer;
    private readonly IContainer _apiContainer;

    public ApiWithDatabaseTests()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithNetwork(_network)
            .WithNetworkAliases("database")
            .Build();

        _apiContainer = new ContainerBuilder()
            .WithImage("myapi:latest")
            .WithNetwork(_network)
            .WithEnvironment("DatabaseConnection", "Host=database;Port=5432;...")
            .WithPortBinding(8080, true)
            .WithWaitStrategy(Wait.ForUnixContainer().UntilHttpRequestIsSucceeded(r => r.ForPath("/health")))
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _network.CreateAsync();
        await _dbContainer.StartAsync();
        await _apiContainer.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _apiContainer.DisposeAsync();
        await _dbContainer.DisposeAsync();
        await _network.DeleteAsync();
    }

    [Fact]
    public async Task ApiCanAccessDatabase()
    {
        var apiPort = _apiContainer.GetMappedPublicPort(8080);
        var client = new HttpClient { BaseAddress = new Uri($"http://localhost:{apiPort}") };

        var response = await client.GetAsync("/api/orders");

        response.IsSuccessStatusCode.Should().BeTrue();
    }
}
```
