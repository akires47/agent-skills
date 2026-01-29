# Cache Containers - Redis

## Basic Redis Integration Test

```csharp
using Testcontainers.Redis;

public class RedisCacheTests : IAsyncLifetime
{
    private readonly RedisContainer _redisContainer = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();

    private IConnectionMultiplexer _redis;
    private IDatabase _db;

    public async Task InitializeAsync()
    {
        await _redisContainer.StartAsync();

        _redis = await ConnectionMultiplexer.ConnectAsync(_redisContainer.GetConnectionString());
        _db = _redis.GetDatabase();
    }

    public async Task DisposeAsync()
    {
        await _redis.DisposeAsync();
        await _redisContainer.DisposeAsync();
    }

    [Fact]
    public async Task CanSetAndGetValue()
    {
        // Arrange
        const string key = "test:order:123";
        const string value = "Order Data";

        // Act
        await _db.StringSetAsync(key, value);
        var retrieved = await _db.StringGetAsync(key);

        // Assert
        retrieved.ToString().Should().Be(value);
    }

    [Fact]
    public async Task CanSetWithExpiration()
    {
        // Arrange
        const string key = "test:session:abc";
        var expiry = TimeSpan.FromSeconds(1);

        // Act
        await _db.StringSetAsync(key, "data", expiry);
        await Task.Delay(TimeSpan.FromSeconds(2));
        var retrieved = await _db.StringGetAsync(key);

        // Assert
        retrieved.IsNullOrEmpty.Should().BeTrue();
    }
}
```

## Testing Cache Service

```csharp
public class CacheServiceTests : IAsyncLifetime
{
    private readonly RedisContainer _redisContainer = new RedisBuilder().Build();
    private CacheService _cacheService;

    public async Task InitializeAsync()
    {
        await _redisContainer.StartAsync();

        var redis = await ConnectionMultiplexer.ConnectAsync(_redisContainer.GetConnectionString());
        _cacheService = new CacheService(redis);
    }

    public async Task DisposeAsync()
    {
        await _redisContainer.DisposeAsync();
    }

    [Fact]
    public async Task GetOrCreateAsync_CachesResult()
    {
        // Arrange
        const string key = "expensive-computation";
        var callCount = 0;

        // Act - First call
        var result1 = await _cacheService.GetOrCreateAsync(
            key,
            async () =>
            {
                callCount++;
                return await Task.FromResult("computed value");
            },
            TimeSpan.FromMinutes(5));

        // Act - Second call
        var result2 = await _cacheService.GetOrCreateAsync(
            key,
            async () =>
            {
                callCount++;
                return await Task.FromResult("computed value");
            },
            TimeSpan.FromMinutes(5));

        // Assert
        result1.Should().Be("computed value");
        result2.Should().Be("computed value");
        callCount.Should().Be(1); // Factory called only once
    }
}
```

## Testing Distributed Cache (IDistributedCache)

```csharp
public class DistributedCacheTests : IAsyncLifetime
{
    private readonly RedisContainer _redisContainer = new RedisBuilder().Build();
    private IDistributedCache _cache;

    public async Task InitializeAsync()
    {
        await _redisContainer.StartAsync();

        var options = Options.Create(new RedisCacheOptions
        {
            Configuration = _redisContainer.GetConnectionString()
        });

        _cache = new RedisCache(options);
    }

    public async Task DisposeAsync()
    {
        await _redisContainer.DisposeAsync();
    }

    [Fact]
    public async Task CanCacheAndRetrieveObject()
    {
        // Arrange
        var order = new Order { Id = 123, Total = 99.99m };
        var json = JsonSerializer.Serialize(order);
        var key = $"order:{order.Id}";

        // Act
        await _cache.SetStringAsync(key, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });

        var cached = await _cache.GetStringAsync(key);
        var deserialized = JsonSerializer.Deserialize<Order>(cached);

        // Assert
        deserialized.Should().NotBeNull();
        deserialized.Id.Should().Be(123);
        deserialized.Total.Should().Be(99.99m);
    }
}
```
