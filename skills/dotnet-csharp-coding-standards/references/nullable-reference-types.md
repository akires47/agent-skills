# Nullable Reference Types (C# 8+)

Enable nullable reference types in your project and handle nulls explicitly.

## Enable in Project

```csharp
// In .csproj
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

## Explicit Nullability

```csharp
public class UserService
{
    // Non-nullable by default
    public string GetUserName(User user) => user.Name;

    // Explicitly nullable return
    public string? FindUserName(string userId)
    {
        var user = _repository.Find(userId);
        return user?.Name;  // Returns null if user not found
    }

    // Null-forgiving operator (use sparingly!)
    public string GetRequiredConfigValue(string key)
    {
        var value = Configuration[key];
        return value!;  // Only if you're CERTAIN it's not null
    }

    // Nullable value objects
    public Money? GetAccountBalance(string accountId)
    {
        var account = _repository.Find(accountId);
        return account?.Balance;
    }
}
```

## Pattern Matching with Null Checks

```csharp
public decimal GetDiscount(Customer? customer) => customer switch
{
    null => 0m,
    { IsVip: true } => 0.20m,
    { OrderCount: > 10 } => 0.10m,
    _ => 0.05m
};
```

## Null-Coalescing Patterns

```csharp
public string GetDisplayName(User? user) =>
    user?.PreferredName ?? user?.Email ?? "Guest";
```

## Guard Clauses with ArgumentNullException.ThrowIfNull (C# 11+)

```csharp
public void ProcessOrder(Order? order)
{
    ArgumentNullException.ThrowIfNull(order);

    // order is now non-nullable in this scope
    Console.WriteLine(order.Id);
}
```
