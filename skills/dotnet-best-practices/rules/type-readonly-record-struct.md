# type-readonly-record-struct

Use readonly record struct for value objects

## Why It Matters

Value objects should be immutable and stack-allocated for performance. readonly record struct prevents defensive copies and provides value semantics.

## ❌ Incorrect

```csharp
// BAD: Mutable struct causes defensive copies
public struct OrderId
{
    public Guid Value { get; set; }  // Mutable!
}
```

## ✅ Correct

```csharp
// GOOD: Immutable value object
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}
```

## Context

- Use for domain primitives (OrderId, CustomerId, Money)\n- Stack-allocated, no heap pressure\n- Value-based equality automatically\n- See also: `type-records-for-dtos`, `type-explicit-conversions`
