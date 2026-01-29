# Mapping Patterns

**Prefer statically-typed, explicit code over reflection-based "magic" libraries.**

## Banned Libraries

| Library | Problem |
|---------|---------|
| **AutoMapper** | Reflection magic, hidden mappings, runtime failures, hard to debug |
| **Mapster** | Same issues as AutoMapper |
| **ExpressMapper** | Same issues |

## Why Reflection Mapping Fails

```csharp
// With AutoMapper - compiles fine, fails at runtime
public record UserDto(string Id, string Name, string Email);
public record UserEntity(Guid Id, string FullName, string EmailAddress);

// This mapping silently produces garbage:
// - Id: string vs Guid mismatch
// - Name vs FullName: no match, null/default
// - Email vs EmailAddress: no match, null/default
var dto = _mapper.Map<UserDto>(entity);  // Compiles! Breaks at runtime.
```

## Use Explicit Mapping Methods Instead

```csharp
// Extension method - compile-time checked, easy to find, easy to debug
public static class UserMappings
{
    public static UserDto ToDto(this UserEntity entity) => new(
        Id: entity.Id.ToString(),
        Name: entity.FullName,
        Email: entity.EmailAddress);

    public static UserEntity ToEntity(this CreateUserRequest request) => new(
        Id: Guid.NewGuid(),
        FullName: request.Name,
        EmailAddress: request.Email);
}

// Usage - explicit and traceable
var dto = entity.ToDto();
var entity = request.ToEntity();
```

## Benefits of Explicit Mappings

| Aspect | AutoMapper | Explicit Methods |
|--------|------------|------------------|
| **Compile-time safety** | No - runtime errors | Yes - compiler catches mismatches |
| **Discoverability** | Hidden in profiles | "Go to Definition" works |
| **Debugging** | Black box | Step through code |
| **Refactoring** | Rename breaks silently | IDE renames correctly |
| **Performance** | Reflection overhead | Direct property access |
| **Testing** | Need integration tests | Simple unit tests |

## Complex Mappings

For complex transformations, explicit code is even more valuable:

```csharp
public static OrderSummaryDto ToSummary(this Order order) => new(
    OrderId: order.Id.Value.ToString(),
    CustomerName: order.Customer.FullName,
    ItemCount: order.Items.Count,
    Total: order.Items.Sum(i => i.Quantity * i.UnitPrice),
    Status: order.Status switch
    {
        OrderStatus.Pending => "Awaiting Payment",
        OrderStatus.Paid => "Processing",
        OrderStatus.Shipped => "On the Way",
        OrderStatus.Delivered => "Completed",
        _ => "Unknown"
    },
    FormattedDate: order.CreatedAt.ToString("MMMM d, yyyy"));
```

This is:
- **Readable**: Anyone can understand the transformation
- **Debuggable**: Set a breakpoint, inspect values
- **Testable**: Pass an Order, assert on the result
- **Refactorable**: Change a property name, compiler tells you everywhere it's used

## When Reflection is Acceptable

| Use Case | Acceptable? |
|----------|-------------|
| Serialization (System.Text.Json, Newtonsoft) | Yes - well-tested, source generators available |
| Dependency injection container | Yes - framework infrastructure |
| ORM entity mapping (EF Core) | Yes - necessary for database abstraction |
| Test fixtures and builders | Sometimes - for convenience in tests only |
| **DTO/domain object mapping** | **No - use explicit methods** |
