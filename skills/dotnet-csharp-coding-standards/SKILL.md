---
name: dotnet-csharp-coding-standards
description: Write modern, high-performance C# code using records, pattern matching, value objects, async/await, Span<T>/Memory<T>, and best-practice API design patterns. Emphasizes functional-style programming with C# 12+ features.
---

# Modern C# Coding Standards

## When to Use This Skill

Use this skill when:
- Writing new C# code or refactoring existing code
- Designing public APIs for libraries or services
- Optimizing performance-critical code paths
- Implementing domain models with strong typing
- Building async/await-heavy applications
- Working with binary data, buffers, or high-throughput scenarios

## Core Principles

1. **Immutability by Default** - Use `record` types and `init`-only properties
2. **Type Safety** - Leverage nullable reference types and value objects
3. **Modern Pattern Matching** - Use `switch` expressions and patterns extensively
4. **Async Everywhere** - Prefer async APIs with proper cancellation support
5. **Zero-Allocation Patterns** - Use `Span<T>` and `Memory<T>` for performance-critical code
6. **API Design** - Accept abstractions, return appropriately specific types
7. **Composition Over Inheritance** - Avoid abstract base classes, prefer composition
8. **Value Objects as Structs** - Use `readonly record struct` for value objects
9. **Explicit Over Magic** - Use explicit mapping methods instead of AutoMapper
10. **Fail Fast** - Validate inputs at boundaries and use Result types for expected errors

---

## Quick Reference

### Records for Immutable Data

```csharp
// Simple immutable DTO
public record CustomerDto(string Id, string Name, string Email);

// Record with validation
public record EmailAddress
{
    public string Value { get; init; }
    
    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email address");
        Value = value;
    }
}
```

### Value Objects as readonly record struct

```csharp
// Single-value object with validation
public readonly record struct OrderId(string Value)
{
    public OrderId(string value) : this(
        !string.IsNullOrWhiteSpace(value)
            ? value
            : throw new ArgumentException("OrderId cannot be empty"))
    {
    }
}

// NO implicit conversions - defeats type safety!
```

### Pattern Matching

```csharp
// Switch expressions
public string GetPaymentDescription(PaymentMethod payment) => payment switch
{
    { Type: PaymentType.CreditCard, Last4: var last4 } => $"Card ending in {last4}",
    { Type: PaymentType.BankTransfer } => "Bank transfer",
    _ => "Unknown"
};
```

### Async/Await

```csharp
// Always use async for I/O operations with CancellationToken
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken cancellationToken = default)
{
    return await _repository.GetAsync(orderId, cancellationToken);
}
```

### Composition Over Inheritance

```csharp
// ✅ GOOD: Interfaces + composition
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(Money amount, CancellationToken ct);
}

public sealed class CreditCardProcessor : IPaymentProcessor
{
    private readonly IPaymentValidator _validator;
    private readonly ICreditCardGateway _gateway;
    
    // Constructor injection for dependencies
}

// ❌ BAD: Abstract base classes
public abstract class PaymentProcessor { }
```

---

## References

See detailed implementations and patterns in the `references/` folder:

- [Records and Value Types](references/records-and-value-types.md) - Immutable DTOs, value objects, validation
- [Pattern Matching](references/pattern-matching.md) - Switch expressions, property patterns, list patterns
- [Nullable Reference Types](references/nullable-reference-types.md) - Enable nullability, handle nulls explicitly
- [Composition Over Inheritance](references/composition-over-inheritance.md) - Avoid abstract classes, use interfaces
- [Async Patterns](references/async-patterns.md) - async/await, ValueTask, cancellation, streaming
- [Span and Memory](references/span-and-memory.md) - Zero-allocation code with Span<T> and Memory<T>
- [API Design](references/api-design.md) - Accept abstractions, return appropriate types
- [Error Handling](references/error-handling.md) - Result type pattern for expected errors
- [Mapping Patterns](references/mapping-patterns.md) - Why not AutoMapper, explicit mapping methods
- [Testing Patterns](references/testing-patterns.md) - Test builders, value object testing
- [Anti-Patterns](references/anti-patterns.md) - Common mistakes to avoid

## Resources

- **C# Language**: https://learn.microsoft.com/en-us/dotnet/csharp/
- **Pattern Matching**: https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching
- **Span<T> and Memory<T>**: https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/
- **Async Best Practices**: https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming
