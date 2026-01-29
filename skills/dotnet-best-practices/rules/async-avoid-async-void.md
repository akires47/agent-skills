# async-avoid-async-void

Avoid async void methods except for event handlers; use async Task instead to enable error handling and awaiting.

## Why It Matters

Async void methods can't be awaited, exceptions can't be caught by callers, and they make testing difficult. Async Task provides proper error propagation and composition.

## ❌ Incorrect

```csharp
// BAD: async void for non-event-handler
public async void ProcessOrderAsync(Order order)
{
    try
    {
        await _repository.SaveAsync(order);
        await _eventBus.PublishAsync(new OrderCreated(order.Id));
    }
    catch (Exception ex)
    {
        // Caller can't catch this - crashes the process!
        _logger.LogError(ex, "Failed to process order");
    }
}

// Caller has no way to know when it completes
ProcessOrderAsync(order);  // Fire and forget - dangerous
DoSomethingElse();

// BAD: async void in background work
public async void StartBackgroundWork()
{
    while (true)
    {
        await ProcessBatchAsync();
        await Task.Delay(TimeSpan.FromMinutes(1));
    }
    // If this throws, application crashes
}

// BAD: Can't unit test async void
[Fact]
public void ProcessOrder_SavesSuccessfully()
{
    var order = new Order();
    _service.ProcessOrderAsync(order);  // Returns void - can't await!
    // Test finishes before ProcessOrderAsync completes - race condition
}
```

## ✅ Correct

```csharp
// GOOD: async Task enables proper error handling
public async Task ProcessOrderAsync(
    Order order,
    CancellationToken ct = default)
{
    await _repository.SaveAsync(order, ct);
    await _eventBus.PublishAsync(new OrderCreated(order.Id), ct);
}

// Caller can await and handle errors
try
{
    await ProcessOrderAsync(order, ct);
    Console.WriteLine("Order processed successfully");
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process order");
}

// GOOD: async void only for event handlers
private async void OnButtonClick(object sender, EventArgs e)
{
    try
    {
        await ProcessOrderAsync(_currentOrder);
        MessageBox.Show("Order processed!");
    }
    catch (Exception ex)
    {
        MessageBox.Show($"Error: {ex.Message}");
    }
}

// GOOD: Background service with Task
public sealed class OrderBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await ProcessBatchAsync(ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Batch processing failed");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}

// GOOD: Testable async Task method
public async Task<Result<Order>> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken ct = default)
{
    var validation = Validate(request);
    if (!validation.IsValid)
        return validation.ToResult<Order>();
    
    var order = ToEntity(request);
    await _repository.SaveAsync(order, ct);
    return Result.Success(order);
}

[Fact]
public async Task CreateOrder_WithValidRequest_Succeeds()
{
    var request = new CreateOrderRequest(/* ... */);
    
    var result = await _service.CreateOrderAsync(request);  // Can await!
    
    result.IsSuccess.Should().BeTrue();
}

// GOOD: If you truly need fire-and-forget, be explicit
public void StartProcessing(Order order)
{
    _ = ProcessInBackgroundAsync(order);  // Explicit discard
}

private async Task ProcessInBackgroundAsync(Order order)
{
    try
    {
        await ProcessOrderAsync(order);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Background processing failed");
    }
}
```

## Context

- **Always** use `async Task` except for event handlers
- Event handlers must be `async void` because of delegate signature
- Exceptions in async void crash the app unless caught inside the method
- Can't await, test, or compose async void methods
- Related: `async-all-the-way.md`, `async-cancellation-token.md`, `arch-background-services.md`
- Use `Task.Run` for true fire-and-forget with proper exception handling
