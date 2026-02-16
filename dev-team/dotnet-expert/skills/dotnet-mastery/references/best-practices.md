# C# and .NET Best Practices

## .NET 8+ Features

### TimeProvider for Testable Time

```csharp
public class TokenService(TimeProvider timeProvider)
{
    public DateTimeOffset GetExpiry() => timeProvider.GetUtcNow().AddMinutes(15);
}

// In tests — use FakeTimeProvider
var fake = new FakeTimeProvider(new DateTimeOffset(2024, 1, 1, 0, 0, 0, TimeSpan.Zero));
var service = new TokenService(fake);
fake.Advance(TimeSpan.FromMinutes(10));
```

### SearchValues for Fast Lookups

```csharp
private static readonly SearchValues<char> s_vowels = SearchValues.Create("aeiouAEIOU");

public static int CountVowels(ReadOnlySpan<char> text)
{
    int count = 0;
    foreach (var ch in text)
        if (s_vowels.Contains(ch)) count++;
    return count;
}
```

### FrozenDictionary for Read-Heavy Scenarios

```csharp
private static readonly FrozenDictionary<string, int> StatusCodes =
    new Dictionary<string, int> { ["OK"] = 200, ["NotFound"] = 404, ["Error"] = 500 }.ToFrozenDictionary();
```

## Performance with Span and Memory

```csharp
// Parse without allocations
public static bool TryParseDate(ReadOnlySpan<char> input, out DateOnly date)
{
    Span<Range> parts = stackalloc Range[3];
    if (input.Split(parts, '-') != 3) { date = default; return false; }
    var year = int.Parse(input[parts[0]]);
    var month = int.Parse(input[parts[1]]);
    var day = int.Parse(input[parts[2]]);
    date = new DateOnly(year, month, day);
    return true;
}

// ArrayPool for temporary buffers
public static string ProcessLargeData(int size)
{
    var buffer = ArrayPool<byte>.Shared.Rent(size);
    try
    {
        // use buffer
        return Encoding.UTF8.GetString(buffer.AsSpan(0, size));
    }
    finally { ArrayPool<byte>.Shared.Return(buffer); }
}
```

## Async Best Practices

```csharp
// Use SemaphoreSlim for async throttling
private static readonly SemaphoreSlim _semaphore = new(maxCount: 10);

public async Task<T> ThrottledCallAsync<T>(Func<Task<T>> operation, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try { return await operation(); }
    finally { _semaphore.Release(); }
}

// Use Channel<T> for producer-consumer
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// Producer
await channel.Writer.WriteAsync(new WorkItem(), ct);

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item, ct);
```

## Nullable Reference Types Strategy

1. Enable `<Nullable>enable</Nullable>` project-wide
2. Use `required` for constructor-initialized non-nullable properties
3. Use `ArgumentNullException.ThrowIfNull()` at public API boundaries
4. Return `T?` when absence is a valid outcome, `T` when it's an error (throw)
5. Avoid the null-forgiving operator `!` except when you've verified a value is non-null

## Security Guidance

- Always use parameterized queries (EF Core does this by default)
- Encode all user output with `HtmlEncoder.Default.Encode()` when not using Razor
- Validate all inputs at the boundary with FluentValidation
- Use `[FromBody]`, `[FromQuery]`, `[FromRoute]` explicitly to prevent over-posting
- Store secrets in User Secrets (dev) or Key Vault (prod), never in `appsettings.json`
