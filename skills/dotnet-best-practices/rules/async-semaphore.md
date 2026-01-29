# async-semaphore

Use SemaphoreSlim for async throttling and resource limiting, not lock or Monitor.

## Why It Matters

SemaphoreSlim supports async waiting without blocking threads, unlike lock/Monitor which block. This prevents thread pool starvation in async code while providing concurrency control.

## ❌ Incorrect

```csharp
// BAD: lock blocks threads in async code
private readonly object _lock = new();
private int _activeConnections = 0;

public async Task<Result> ProcessAsync(Request request)
{
    lock (_lock)  // Blocks thread while waiting!
    {
        if (_activeConnections >= 10)
            return Result.Failure(Error.Unavailable("Too many connections"));
        
        _activeConnections++;
    }
    
    try
    {
        return await ProcessRequestAsync(request);
    }
    finally
    {
        lock (_lock)
        {
            _activeConnections--;
        }
    }
}

// BAD: Monitor.Enter in async method
public async Task UpdateAsync(Order order)
{
    Monitor.Enter(_lock);  // Don't use Monitor with async
    try
    {
        await _repository.UpdateAsync(order);
    }
    finally
    {
        Monitor.Exit(_lock);
    }
}

// BAD: Unbounded concurrent HTTP requests
public async Task<List<Result>> FetchAllAsync(List<string> urls)
{
    var tasks = urls.Select(url => _httpClient.GetAsync(url));
    var responses = await Task.WhenAll(tasks);  // Could be thousands!
    return responses.Select(r => Parse(r)).ToList();
}
```

## ✅ Correct

```csharp
// GOOD: SemaphoreSlim for async concurrency control
private readonly SemaphoreSlim _semaphore = new(10, 10);  // Max 10 concurrent

public async Task<Result> ProcessAsync(
    Request request,
    CancellationToken ct = default)
{
    if (!await _semaphore.WaitAsync(0, ct))  // Try immediate
        return Result.Failure(Error.Unavailable("Server busy"));
    
    try
    {
        return await ProcessRequestAsync(request, ct);
    }
    finally
    {
        _semaphore.Release();
    }
}

// GOOD: Throttle concurrent operations
private readonly SemaphoreSlim _throttle = new(5);  // Max 5 concurrent requests

public async Task<List<Order>> FetchOrdersAsync(
    List<string> orderIds,
    CancellationToken ct = default)
{
    var tasks = orderIds.Select(async id =>
    {
        await _throttle.WaitAsync(ct);
        try
        {
            return await GetOrderAsync(id, ct);
        }
        finally
        {
            _throttle.Release();
        }
    });
    
    return (await Task.WhenAll(tasks)).ToList();
}

// GOOD: Rate limiting pattern
private readonly SemaphoreSlim _rateLimiter;
private readonly TimeSpan _rateLimitPeriod = TimeSpan.FromSeconds(1);

public OrderService(int requestsPerSecond)
{
    _rateLimiter = new SemaphoreSlim(requestsPerSecond, requestsPerSecond);
    
    // Refill semaphore periodically
    _ = Task.Run(async () =>
    {
        while (true)
        {
            await Task.Delay(_rateLimitPeriod);
            
            // Refill tokens
            var currentCount = _rateLimiter.CurrentCount;
            var toRelease = requestsPerSecond - currentCount;
            if (toRelease > 0)
                _rateLimiter.Release(toRelease);
        }
    });
}

public async Task<Result> MakeRequestAsync(
    Request request,
    CancellationToken ct = default)
{
    await _rateLimiter.WaitAsync(ct);
    return await ProcessRequestAsync(request, ct);
}

// GOOD: Dispose SemaphoreSlim properly
public sealed class ResourcePool : IDisposable
{
    private readonly SemaphoreSlim _semaphore = new(10);
    
    public async Task<T> UseResourceAsync<T>(
        Func<Task<T>> operation,
        CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    public void Dispose() => _semaphore.Dispose();
}
```

## Context

- Never use `lock` or `Monitor` across `await` boundaries
- SemaphoreSlim is for async, Semaphore is for sync (older, slower)
- Always pair `WaitAsync()` with `Release()` in finally block
- Use `WaitAsync(0)` to try immediate acquisition without waiting
- Related: `async-cancellation-token.md`, `async-parallel-foreach.md`, `async-channel.md`
- Dispose SemaphoreSlim when done (implements IDisposable)
