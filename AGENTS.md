# Agent Guidelines for akires47-agent-skills

This repository contains a collection of reusable agent skills (knowledge modules) for .NET development. Skills are markdown files that agents can load to enhance their capabilities when working on specific types of projects.

## Repository Structure

```
akires47-agent-skills/
├── skills/
│   └── dotnet-best-practices/          # Comprehensive .NET development guidelines
│       ├── SKILL.md                    # Categorized index with quick reference
│       ├── AGENTS.md                   # Full compiled guide for LLM context
│       └── rules/                      # 94 individual rule files
│           ├── error-result-pattern.md
│           ├── async-cancellation-token.md
│           ├── db-notracking-default.md
│           └── ...
├── README.md
└── AGENTS.md (this file)
```

## Working with Skills

### Skill File Structure

The `dotnet-best-practices` skill contains:
- `SKILL.md` - Categorized index with quick reference for all 94 rules
- `AGENTS.md` - Full compiled guide with all rules expanded inline
- `rules/` - Individual rule files (one per rule)

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
1. Maintain the YAML frontmatter structure in SKILL.md
2. Keep code examples accurate and up-to-date
3. Follow markdown formatting conventions
4. Update individual rule files in the `rules/` directory
5. Each rule file should follow the format: title, "Why It Matters", ❌ Incorrect, ✅ Correct, Context

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

- **Rule Files**: Use kebab-case with category prefix (e.g., `error-result-pattern.md`, `async-cancellation-token.md`)
- **Directories**: Use kebab-case (e.g., `dotnet-best-practices`, `rules`)
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

When adding or modifying rules:

1. **One rule per file** - Each rule gets its own markdown file in `rules/`
2. **Include practical examples** - Show real-world code, not contrived demos
3. **Follow the template** - Use ❌ Incorrect and ✅ Correct sections
4. **Cross-reference** - Link related rules in the Context section
5. **Update SKILL.md** - Add new rules to the categorized index
6. **Maintain priority** - Place rules in appropriate priority category (CRITICAL, HIGH, MEDIUM, LOW)

## Rule Categories

The `dotnet-best-practices` skill organizes 94 rules into 11 priority-ranked categories:

| Priority | Category | Impact | Prefix | Rules |
|----------|----------|--------|--------|-------|
| 1 | Error Handling | CRITICAL | `error-` | 8 |
| 2 | Async Patterns | CRITICAL | `async-` | 9 |
| 3 | Type Design | HIGH | `type-` | 10 |
| 4 | Database Performance | HIGH | `db-` | 12 |
| 5 | API Design | MEDIUM-HIGH | `api-` | 8 |
| 6 | Dependency Injection | MEDIUM | `di-` | 7 |
| 7 | Architecture | MEDIUM | `arch-` | 8 |
| 8 | Serialization | MEDIUM | `serial-` | 6 |
| 9 | Performance | LOW-MEDIUM | `perf-` | 12 |
| 10 | Logging | LOW-MEDIUM | `log-` | 6 |
| 11 | Testing | LOW | `test-` | 8 |

## Resources

- Repository: https://github.com/akires47/akires47-agent-skills
- Inspired by: https://github.com/Aaronontheweb/dotnet-skills
- Format based on: https://github.com/vercel-labs/agent-skills
- .NET Documentation: https://learn.microsoft.com/en-us/dotnet/
