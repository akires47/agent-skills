# Agent Guidelines for akires47-agent-skills

This repository contains a collection of reusable agent skills (knowledge modules) for .NET development. Skills are markdown files that agents can load to enhance their capabilities when working on specific types of projects.

## Repository Structure

```
akires47-agent-skills/
├── skills/
│   ├── dotnet-vertical-slice/          # Vertical slice architecture with minimal APIs
│   ├── dotnet-csharp-coding-standards/ # Modern C# patterns and best practices
│   ├── dotnet-testcontainers-integration-tests/ # Integration testing with TestContainers
│   ├── dotnet-efcore-patterns/         # Entity Framework Core patterns
│   ├── dotnet-dependency-injection-patterns/ # DI patterns and lifetime management
│   ├── dotnet-extensions-configuration/ # Options pattern and configuration
│   └── [other skills]/
├── README.md
└── AGENTS.md (this file)
```

## Working with Skills

### Skill File Structure

Each skill directory contains:
- `SKILL.md` - Main skill documentation with patterns and quick reference
- `references/` - Detailed reference documentation for specific topics

### Skill Format

Skills use YAML frontmatter followed by markdown content:

```yaml
---
name: skill-name
description: When and why to use this skill
---
# Skill Title

Content with patterns, code examples, and guidelines...
```

## Build/Test/Lint Commands

This is a **documentation-only repository** with no code to build, test, or lint. There are:
- No `.csproj`, `.sln`, or `package.json` files
- No build scripts or test runners
- No linting configuration

### Editing Skills

When modifying skill files:
1. Maintain the YAML frontmatter structure
2. Keep code examples accurate and up-to-date
3. Follow markdown formatting conventions
4. Update reference files in the `references/` subdirectory as needed

### Validation

To validate changes:
- Ensure markdown is properly formatted
- Verify code examples are syntactically correct
- Check that cross-references between files are accurate
- Confirm YAML frontmatter is valid

## Code Style Guidelines

Since this repository contains markdown documentation with embedded code examples, follow these conventions:

### Markdown Formatting

- Use ATX-style headers (`#`, `##`, etc.)
- Use fenced code blocks with language identifiers (` ```csharp `)
- Keep line length reasonable (wrap at ~120 characters for prose)
- Use reference-style links for repeated URLs

### Code Examples in Skills

All C# code examples should follow these patterns:

#### 1. Immutability by Default
```csharp
// ✅ Use records for DTOs
public record CustomerDto(string Id, string Name, string Email);

// ✅ Use readonly record struct for value objects
public readonly record struct OrderId(string Value);
```

#### 2. Type Safety
```csharp
// ✅ Enable nullable reference types
#nullable enable

// ✅ Use value objects instead of primitives
public readonly record struct EmailAddress(string Value)
{
    public EmailAddress(string value) : this(
        !string.IsNullOrWhiteSpace(value) && value.Contains('@')
            ? value
            : throw new ArgumentException("Invalid email"))
    {
    }
}
```

#### 3. Pattern Matching
```csharp
// ✅ Use switch expressions
public string GetDescription(PaymentMethod payment) => payment switch
{
    { Type: PaymentType.CreditCard, Last4: var last4 } => $"Card ending in {last4}",
    { Type: PaymentType.BankTransfer } => "Bank transfer",
    _ => "Unknown"
};
```

#### 4. Async/Await
```csharp
// ✅ Always include CancellationToken
public async Task<Order> GetOrderAsync(
    string orderId,
    CancellationToken cancellationToken = default)
{
    return await _repository.GetAsync(orderId, cancellationToken);
}
```

#### 5. Result Pattern (No Exceptions)
```csharp
// ✅ Return Result<T> for expected errors
public async Task<Result<Order>> CreateOrderAsync(
    CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    var validation = ValidateRequest(request);
    if (!validation.IsValid)
        return Result.Failure<Order>(Error.Validation("Invalid request"));

    // ... implementation
    return Result.Success(order);
}
```

#### 6. API Design
```csharp
// ✅ Accept abstractions
public decimal CalculateTotal(IEnumerable<OrderItem> items)
{
    return items.Sum(item => item.Price * item.Quantity);
}

// ✅ Return appropriately specific types
public IReadOnlyList<Order> GetOrders(string customerId)
{
    return _repository.Query()
        .Where(o => o.CustomerId == customerId)
        .ToList();
}
```

#### 7. Composition Over Inheritance
```csharp
// ✅ Use interfaces + composition
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(Money amount, CancellationToken ct);
}

public sealed class CreditCardProcessor : IPaymentProcessor
{
    // Implementation
}

// ❌ Avoid abstract base classes
```

### Naming Conventions

- **Files**: Use kebab-case (e.g., `result-pattern.md`, `error-handling.md`)
- **Directories**: Use kebab-case (e.g., `dotnet-vertical-slice`, `references`)
- **C# Code**: Follow standard .NET conventions (PascalCase for types/methods, camelCase for parameters)

### Error Handling in Examples

Always show the Result pattern for expected errors:

```csharp
// ✅ Result type for expected errors
return Result.Failure<Response>(Error.NotFound("Product", id));

// ✅ Exceptions only for unexpected errors (system failures)
throw new InvalidOperationException("Critical system failure");
```

## Contribution Guidelines

When adding or modifying skills:

1. **Create focused skills** - Each skill should address a specific technology or pattern
2. **Include practical examples** - Show real-world code, not contrived demos
3. **Provide references** - Break complex topics into separate reference files
4. **Follow existing structure** - Match the format of existing skills
5. **Update README.md** - Add new skills to the table in README.md
6. **Use frontmatter** - Always include valid YAML frontmatter with name and description

## Resources

- Repository: https://github.com/akires47/akires47-agent-skills
- Inspired by: https://github.com/Aaronontheweb/dotnet-skills
- .NET Documentation: https://learn.microsoft.com/en-us/dotnet/
