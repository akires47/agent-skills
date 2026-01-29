# Anti-Patterns to Avoid

## ❌ DON'T: Use mutable DTOs

```csharp
// BAD: Mutable DTO
public class CustomerDto
{
    public string Id { get; set; }
    public string Name { get; set; }
}

// GOOD: Immutable record
public record CustomerDto(string Id, string Name);
```

## ❌ DON'T: Use classes for value objects

```csharp
// BAD: Value object as class
public class OrderId
{
    public string Value { get; }
    public OrderId(string value) => Value = value;
}

// GOOD: Value object as readonly record struct
public readonly record struct OrderId(string Value);
```

## ❌ DON'T: Create deep inheritance hierarchies

```csharp
// BAD: Deep inheritance
public abstract class Entity { }
public abstract class AggregateRoot : Entity { }
public abstract class Order : AggregateRoot { }
public class CustomerOrder : Order { }

// GOOD: Flat structure with composition
public interface IEntity
{
    Guid Id { get; }
}

public record Order(OrderId Id, CustomerId CustomerId, Money Total) : IEntity
{
    Guid IEntity.Id => Id.Value;
}
```

## ❌ DON'T: Return List<T> when you mean IReadOnlyList<T>

```csharp
// BAD: Exposes internal list for modification
public List<Order> GetOrders() => _orders;

// GOOD: Returns read-only view
public IReadOnlyList<Order> GetOrders() => _orders;
```

## ❌ DON'T: Use byte[] when ReadOnlySpan<byte> works

```csharp
// BAD: Allocates array on every call
public byte[] GetHeader()
{
    var header = new byte[64];
    // Fill header
    return header;
}

// GOOD: Zero allocation with Span
public void GetHeader(Span<byte> destination)
{
    if (destination.Length < 64)
        throw new ArgumentException("Buffer too small");

    // Fill header directly into caller's buffer
}
```

## ❌ DON'T: Forget CancellationToken in async methods

```csharp
// BAD: No cancellation support
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await _repository.GetAsync(id);
}

// GOOD: Cancellation support
public async Task<Order> GetOrderAsync(
    OrderId id,
    CancellationToken cancellationToken = default)
{
    return await _repository.GetAsync(id, cancellationToken);
}
```

## ❌ DON'T: Block on async code

```csharp
// BAD: Deadlock risk!
public Order GetOrder(OrderId id)
{
    return GetOrderAsync(id).Result;
}

// BAD: Also deadlock risk!
public Order GetOrder(OrderId id)
{
    return GetOrderAsync(id).GetAwaiter().GetResult();
}

// GOOD: Async all the way
public async Task<Order> GetOrderAsync(
    OrderId id,
    CancellationToken cancellationToken)
{
    return await _repository.GetAsync(id, cancellationToken);
}
```

## Code Organization Example

```csharp
// File: Domain/Orders/Order.cs

namespace MyApp.Domain.Orders;

// 1. Primary domain type
public record Order(
    OrderId Id,
    CustomerId CustomerId,
    Money Total,
    OrderStatus Status,
    IReadOnlyList<OrderItem> Items
)
{
    // Computed properties
    public bool IsCompleted => Status is OrderStatus.Completed;

    // Domain methods returning Result for expected errors
    public Result<Order, OrderError> AddItem(OrderItem item)
    {
        if (Status is not OrderStatus.Draft)
            return Result<Order, OrderError>.Failure(
                new OrderError("ORDER_NOT_DRAFT", "Can only add items to draft orders"));

        var newItems = Items.Append(item).ToList();
        var newTotal = new Money(
            Items.Sum(i => i.Total.Amount) + item.Total.Amount,
            Total.Currency);

        return Result<Order, OrderError>.Success(
            this with { Items = newItems, Total = newTotal });
    }
}

// 2. Enums for state
public enum OrderStatus
{
    Draft,
    Submitted,
    Processing,
    Completed,
    Cancelled
}

// 3. Related types
public record OrderItem(
    ProductId ProductId,
    Quantity Quantity,
    Money UnitPrice
)
{
    public Money Total => new(
        UnitPrice.Amount * Quantity.Value,
        UnitPrice.Currency);
}

// 4. Value objects
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
}

// 5. Errors
public readonly record struct OrderError(string Code, string Message);
```
