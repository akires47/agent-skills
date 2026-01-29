# type-records-for-dtos

Use records for immutable Data Transfer Objects (DTOs).

## Why It Matters

Records provide value-based equality, immutability, and concise syntax perfect for DTOs. They prevent accidental mutation and make your data contracts explicit.

## ❌ Incorrect

```csharp
// BAD: Mutable class with public setters
public class CustomerDto
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

// Can be mutated after creation - breaks immutability
var customer = new CustomerDto { Id = "123", Name = "John" };
customer.Email = "changed@example.com";  // Unexpected mutation
```

## ✅ Correct

```csharp
// GOOD: Immutable record
public record CustomerDto(string Id, string Name, string Email);

// Cannot be mutated - compiler enforced
var customer = new CustomerDto("123", "John", "john@example.com");
// customer.Email = "...";  // Compile error!

// Use 'with' expression for non-destructive mutation
var updated = customer with { Email = "newemail@example.com" };
```

## Context

- Records provide automatic Equals/GetHashCode based on values
- Perfect for API request/response types
- Use positional syntax for simple DTOs
- Use property syntax when you need additional members
- See also: `type-immutability-default`, `type-no-public-setters`
