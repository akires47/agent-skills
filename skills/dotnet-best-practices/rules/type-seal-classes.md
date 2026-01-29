# type-seal-classes

Seal classes by default unless designed for inheritance

## Why It Matters

Sealing classes enables JIT devirtualization optimizations and clearly communicates design intent. Classes should be explicitly designed for inheritance, not left unsealed by default.

## ❌ Incorrect

```csharp
// BAD: Unsealed without reason
public class OrderProcessor
{
    public virtual void Process(Order order) { }
}
```

## ✅ Correct

```csharp
// GOOD: Sealed class
public sealed class OrderProcessor
{
    public void Process(Order order) { }  // Non-virtual
}

// GOOD: Sealed records
public sealed record OrderCreated(OrderId Id, CustomerId CustomerId);
```

## Context

- JIT can devirtualize method calls\n- Prevents accidental inheritance\n- Makes intent clear\n- See also: `type-composition-over-inheritance`, `perf-static-pure-functions`
