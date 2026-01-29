# async-valuetask

Use ValueTask<T> for hot paths that often complete synchronously, but use Task<T> for typical I/O operations.

## Why It Matters

ValueTask<T> avoids heap allocation when results are cached or immediately available, improving performance in hot paths. However, it has usage restrictions that make Task<T> safer and more appropriate for most scenarios.

## ❌ Incorrect

```csharp
// BAD: ValueTask for always-async operations
public async ValueTask<Order> CreateOrderAsync(CreateOrderRequest request)
{
    // Always hits database - no benefit from ValueTask
    var order = ToEntity(request);
    await _db.SaveAsync(order);
    return order;
}

// BAD: Awaiting ValueTask multiple times
public async Task ProcessAsync()
{
    var task = GetCachedValueAsync(key);
    await task;  // First await
    await task;  // ERROR: Awaiting again is undefined behavior!
}

// BAD: Storing ValueTask for later
private ValueTask<User> _userTask;

public async Task InitializeAsync()
{
    _userTask = LoadUserAsync();  // DON'T store ValueTask
    await Task.Delay(1000);
    var user = await _userTask;   // Unsafe!
}

// BAD: Using .Result on ValueTask
public User GetUser(string id)
{
    return GetUserAsync(id).Result;  // Don't block ValueTask!
}
```

## ✅ Correct

```csharp
// GOOD: ValueTask for frequently cached operations
private readonly Dictionary<string, User> _cache = new();

public ValueTask<User?> GetUserAsync(
    string userId,
    CancellationToken ct = default)
{
    if (_cache.TryGetValue(userId, out var cached))
        return ValueTask.FromResult<User?>(cached);  // No allocation
    
    return new ValueTask<User?>(LoadUserFromDbAsync(userId, ct));
}

private async Task<User?> LoadUserFromDbAsync(string userId, CancellationToken ct)
{
    var user = await _db.Users.FindAsync(userId, ct);
    if (user != null)
        _cache[userId] = user;
    return user;
}

// GOOD: Task for regular I/O operations
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken ct = default)
{
    // Always async - Task is simpler and safer
    return await _repository.GetAsync(orderId, ct);
}

// GOOD: Immediately await ValueTask
public async Task ProcessAsync(string key)
{
    var value = await GetCachedValueAsync(key);  // Await immediately, once
    Console.WriteLine(value);
}

// GOOD: Convert to Task if you need to store or await multiple times
private Task<User> _userTask;

public async Task InitializeAsync()
{
    _userTask = LoadUserAsync().AsTask();  // Convert to Task for storage
    await Task.Delay(1000);
    var user = await _userTask;  // Safe - it's a Task now
}
```

## Context

- Use ValueTask<T> only when: (1) hot path, (2) often completes synchronously, (3) performance matters
- Use Task<T> for: (1) always-async operations, (2) when you need to await multiple times, (3) default choice
- ValueTask rules: await immediately, await once, don't store unless converted to Task
- Never use `.Result` or `.Wait()` on ValueTask
- Related: `async-all-the-way.md`, `perf-defer-enumeration.md`
- Consider `IAsyncEnumerable<T>` for streaming scenarios instead of ValueTask
