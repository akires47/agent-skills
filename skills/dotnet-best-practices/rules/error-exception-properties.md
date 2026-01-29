# error-exception-properties

Add context properties to custom exceptions to aid debugging and diagnostics.

## Why It Matters

Exception messages alone often lack crucial context for debugging. Adding strongly-typed properties captures state at the point of failure, enabling better logging, monitoring, and troubleshooting without parsing error messages.

## ❌ Incorrect

```csharp
// BAD: No context properties
public class DataValidationException : Exception
{
    public DataValidationException(string message) : base(message) { }
}

throw new DataValidationException("Field validation failed");
// What field? What value? What rule?

// BAD: Context only in message string
throw new InvalidOperationException(
    $"Cannot process order {orderId} in status {status} for customer {customerId}");
// Have to parse the message to extract orderId, status, customerId

// BAD: Generic exception with no context
public async Task ProcessAsync(string id)
{
    var entity = await _repository.GetAsync(id);
    if (entity == null)
        throw new Exception("Not found");  // Which entity type? Which ID?
}
```

## ✅ Correct

```csharp
// GOOD: Properties for structured context
public sealed class EntityNotFoundException : Exception
{
    public string EntityType { get; }
    public string EntityId { get; }
    
    public EntityNotFoundException(string entityType, string entityId)
        : base($"{entityType} with ID '{entityId}' was not found")
    {
        EntityType = entityType;
        EntityId = entityId;
    }
    
    // Override Data property for logging frameworks
    public override IDictionary Data => new Dictionary<string, object>
    {
        ["EntityType"] = EntityType,
        ["EntityId"] = EntityId
    };
}

// GOOD: Rich context for complex validation
public sealed class DataValidationException : Exception
{
    public string FieldName { get; }
    public object? AttemptedValue { get; }
    public string ValidationRule { get; }
    
    public DataValidationException(
        string fieldName,
        object? attemptedValue,
        string validationRule)
        : base($"Field '{fieldName}' failed validation: {validationRule}")
    {
        FieldName = fieldName;
        AttemptedValue = attemptedValue;
        ValidationRule = validationRule;
    }
}

throw new DataValidationException("Age", -5, "Must be non-negative");

// GOOD: Capture relevant state
public sealed class InvalidOrderStateException : InvalidOperationException
{
    public OrderId OrderId { get; }
    public OrderStatus CurrentStatus { get; }
    public OrderStatus RequiredStatus { get; }
    
    public InvalidOrderStateException(
        OrderId orderId,
        OrderStatus current,
        OrderStatus required)
        : base($"Order {orderId} is in {current} status, but {required} is required")
    {
        OrderId = orderId;
        CurrentStatus = current;
        RequiredStatus = required;
    }
}

// Usage in logging
catch (InvalidOrderStateException ex)
{
    _logger.LogError(ex,
        "Cannot process order {OrderId} in status {CurrentStatus}, requires {RequiredStatus}",
        ex.OrderId, ex.CurrentStatus, ex.RequiredStatus);
}
```

## Context

- Properties enable structured logging and monitoring
- Include IDs, entity types, and relevant state values
- Don't include sensitive data (passwords, tokens, PII)
- Override `Data` property for legacy logging frameworks
- Related: `error-custom-exceptions.md`, `log-structured-logging.md`
- Properties should be read-only and set via constructor
