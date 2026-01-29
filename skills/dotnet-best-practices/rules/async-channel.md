# async-channel

Use System.Threading.Channels for producer-consumer patterns and async pipelines instead of blocking collections.

## Why It Matters

Channels provide high-performance, async-first producer-consumer communication without blocking threads. They're ideal for decoupling producers from consumers, implementing backpressure, and building async pipelines.

## ❌ Incorrect

```csharp
// BAD: BlockingCollection blocks threads
private readonly BlockingCollection<Order> _queue = new();

public void EnqueueOrder(Order order)
{
    _queue.Add(order);  // Blocks if bounded and full
}

public async Task ProcessOrdersAsync()
{
    foreach (var order in _queue.GetConsumingEnumerable())
    {
        await ProcessOrderAsync(order);  // Blocking enumeration + async processing
    }
}

// BAD: Manual async queue with lock
private readonly Queue<Order> _queue = new();
private readonly SemaphoreSlim _semaphore = new(0);

public async Task EnqueueAsync(Order order)
{
    lock (_queue)
    {
        _queue.Enqueue(order);
    }
    _semaphore.Release();
}

public async Task<Order> DequeueAsync()
{
    await _semaphore.WaitAsync();
    lock (_queue)
    {
        return _queue.Dequeue();
    }
}
```

## ✅ Correct

```csharp
// GOOD: Channel for async producer-consumer
private readonly Channel<Order> _channel = Channel.CreateBounded<Order>(
    new BoundedChannelOptions(100)
    {
        FullMode = BoundedChannelFullMode.Wait  // Backpressure
    });

public async Task EnqueueOrderAsync(
    Order order,
    CancellationToken ct = default)
{
    await _channel.Writer.WriteAsync(order, ct);  // Async wait if full
}

public async Task ProcessOrdersAsync(CancellationToken ct = default)
{
    await foreach (var order in _channel.Reader.ReadAllAsync(ct))
    {
        await ProcessOrderAsync(order, ct);
    }
}

// GOOD: Multiple producers, single consumer
public sealed class OrderProcessor : BackgroundService
{
    private readonly Channel<Order> _channel;
    
    public OrderProcessor()
    {
        _channel = Channel.CreateUnbounded<Order>(
            new UnboundedChannelOptions
            {
                SingleReader = true  // Optimization for single consumer
            });
    }
    
    public async Task SubmitOrderAsync(Order order, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(order, ct);
    }
    
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var order in _channel.Reader.ReadAllAsync(ct))
        {
            try
            {
                await ProcessOrderAsync(order, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
            }
        }
    }
}

// GOOD: Pipeline pattern with channels
public async Task ProcessPipelineAsync(CancellationToken ct)
{
    var inputChannel = Channel.CreateUnbounded<string>();
    var processedChannel = Channel.CreateUnbounded<Order>();
    
    // Stage 1: Parse
    var parseTask = Task.Run(async () =>
    {
        await foreach (var line in inputChannel.Reader.ReadAllAsync(ct))
        {
            var order = ParseOrder(line);
            await processedChannel.Writer.WriteAsync(order, ct);
        }
        processedChannel.Writer.Complete();
    });
    
    // Stage 2: Save
    var saveTask = Task.Run(async () =>
    {
        await foreach (var order in processedChannel.Reader.ReadAllAsync(ct))
        {
            await SaveOrderAsync(order, ct);
        }
    });
    
    // Feed pipeline
    await FeedDataAsync(inputChannel.Writer, ct);
    inputChannel.Writer.Complete();
    
    await Task.WhenAll(parseTask, saveTask);
}
```

## Context

- Use `Channel.CreateBounded()` for backpressure and memory limits
- Use `Channel.CreateUnbounded()` when producer should never wait
- Set `SingleReader = true` or `SingleWriter = true` for optimizations
- Call `Writer.Complete()` when done producing to signal consumers
- Related: `async-iasyncenumerable.md`, `async-semaphore.md`, `arch-background-services.md`
- Channels are thread-safe and allocation-efficient
