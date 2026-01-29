# Span<T> and Memory<T> for Zero-Allocation Code

Use `Span<T>` and `Memory<T>` instead of `byte[]` or `string` for performance-critical code.

## Span<T> for Synchronous Operations

```csharp
// ✅ GOOD: Span<T> for synchronous, zero-allocation operations
public int ParseOrderId(ReadOnlySpan<char> input)
{
    // Work with data without allocations
    if (!input.StartsWith("ORD-"))
        throw new FormatException("Invalid order ID format");

    var numberPart = input.Slice(4);
    return int.Parse(numberPart);
}

// stackalloc with Span<T>
public void FormatMessage()
{
    Span<char> buffer = stackalloc char[256];
    var written = FormatInto(buffer);
    var message = new string(buffer.Slice(0, written));
}
```

## Memory<T> for Async Operations

```csharp
// ✅ GOOD: Memory<T> for async operations (Span can't cross await)
public async Task<int> ReadDataAsync(
    Memory<byte> buffer,
    CancellationToken cancellationToken)
{
    return await _stream.ReadAsync(buffer, cancellationToken);
}
```

## String Manipulation with Span

```csharp
// ✅ GOOD: String manipulation with Span to avoid allocations
public bool TryParseKeyValue(ReadOnlySpan<char> line, out string key, out string value)
{
    key = string.Empty;
    value = string.Empty;

    int colonIndex = line.IndexOf(':');
    if (colonIndex == -1)
        return false;

    // Only allocate strings once we know the format is valid
    key = new string(line.Slice(0, colonIndex).Trim());
    value = new string(line.Slice(colonIndex + 1).Trim());
    return true;
}
```

## ArrayPool for Large Buffers

```csharp
// ✅ GOOD: ArrayPool for temporary large buffers
public async Task ProcessLargeFileAsync(
    Stream stream,
    CancellationToken cancellationToken)
{
    var buffer = ArrayPool<byte>.Shared.Rent(8192);
    try
    {
        int bytesRead;
        while ((bytesRead = await stream.ReadAsync(buffer.AsMemory(), cancellationToken)) > 0)
        {
            ProcessChunk(buffer.AsSpan(0, bytesRead));
        }
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## Span-Based Parsing

```csharp
// ✅ GOOD: Span-based parsing without substring allocations
public static (string Protocol, string Host, int Port) ParseUrl(ReadOnlySpan<char> url)
{
    var protocolEnd = url.IndexOf("://");
    var protocol = new string(url.Slice(0, protocolEnd));

    var afterProtocol = url.Slice(protocolEnd + 3);
    var portStart = afterProtocol.IndexOf(':');

    var host = new string(afterProtocol.Slice(0, portStart));
    var portSpan = afterProtocol.Slice(portStart + 1);
    var port = int.Parse(portSpan);

    return (protocol, host, port);
}
```

## When to Use What

| Type | Use Case |
|------|----------|
| `Span<T>` | Synchronous operations, stack-allocated buffers, slicing without allocation |
| `ReadOnlySpan<T>` | Read-only views, method parameters for data you won't modify |
| `Memory<T>` | Async operations (Span can't cross await boundaries) |
| `ReadOnlyMemory<T>` | Read-only async operations |
| `byte[]` | When you need to store data long-term or pass to APIs requiring arrays |
| `ArrayPool<T>` | Large temporary buffers (>1KB) to avoid GC pressure |
