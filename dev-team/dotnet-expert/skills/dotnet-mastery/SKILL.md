---
name: .NET Mastery
description: This skill should be used when the user asks to "write a C# class", "create a C# interface", "compose a LINQ query", "implement C# async/await", "define a C# record", "use pattern matching", "build a .NET minimal API", "use generics in C#", or "handle errors in .NET". It covers C# 12+ language features, idiomatic patterns, LINQ, async programming, and core .NET framework usage.
---

# C# Language Mastery

## Modern C# Language Features (C# 12+)

### Records and Immutable Types

Define data transfer objects and value objects with records to get built-in equality, deconstruction, and immutability.

```csharp
// Positional record — immutable by default
public record ProductDto(int Id, string Name, decimal Price, string? Category);

// Record with additional computed members
public record OrderSummary(string OrderId, List<LineItem> Items)
{
    public decimal Total => Items.Sum(i => i.Quantity * i.UnitPrice);
    public int ItemCount => Items.Count;
}

// Record struct for stack-allocated value types
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
    public Money Add(Money other)
    {
        if (other.Currency != Currency)
            throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }
}
```

### Primary Constructors

Use primary constructors on classes and structs to reduce boilerplate for dependency injection.

```csharp
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<Order?> GetOrderAsync(int id, CancellationToken ct = default)
    {
        logger.LogInformation("Fetching order {OrderId}", id);
        return await repository.GetByIdAsync(id, ct);
    }
}
```

### Pattern Matching

Apply switch expressions, property patterns, list patterns, and relational patterns for concise conditional logic.

```csharp
// Switch expression with property patterns
public static string Classify(Order order) => order switch
{
    { Status: OrderStatus.Cancelled }                    => "Cancelled",
    { Total: 0 }                                         => "Empty",
    { Total: > 1000, Customer.IsPremium: true }          => "Premium High-Value",
    { Total: > 1000 }                                    => "High-Value",
    { Items.Count: > 20 }                                => "Bulk",
    _                                                    => "Standard"
};

// List patterns for sequence matching
public static string DescribeRoute(string[] segments) => segments switch
{
    ["api", "v1", var resource]          => $"API v1 resource: {resource}",
    ["api", "v2", var resource, var id]  => $"API v2 resource: {resource}/{id}",
    [var single]                         => $"Root path: {single}",
    []                                   => "Home",
    _                                    => "Unknown route"
};

// Relational patterns
public static string GetDiscount(decimal total) => total switch
{
    >= 500 => "20% off",
    >= 200 => "10% off",
    >= 100 => "5% off",
    _      => "No discount"
};
```

### Nullable Reference Types

Enable nullable reference types project-wide and handle nullability explicitly.

```csharp
public class UserService(IUserRepository repository)
{
    public async Task<User?> FindByEmailAsync(string email, CancellationToken ct)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(email);
        return await repository.FindByEmailAsync(email, ct);
    }

    public async Task<User> GetByIdAsync(int id, CancellationToken ct)
    {
        return await repository.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"User {id} not found");
    }
}
```

### Collection Expressions

```csharp
List<string> names = ["Alice", "Bob", "Charlie"];
int[] primes = [2, 3, 5, 7, 11];

// Spread operator
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]
```

## LINQ

Write LINQ with intent. Prefer method syntax for complex queries; use query syntax when joins improve readability.

```csharp
// Method syntax — projection and filtering
var activeHighValue = orders
    .Where(o => o.Status == OrderStatus.Active)
    .Where(o => o.Total > 500)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new OrderSummaryDto(o.Id, o.CustomerName, o.Total))
    .ToList();

// Query syntax — joins are more readable this way
var report =
    from o in orders
    join c in customers on o.CustomerId equals c.Id
    where o.CreatedAt >= cutoffDate
    group o by new { c.Name, c.Region } into g
    select new { Customer = g.Key.Name, g.Key.Region, TotalRevenue = g.Sum(o => o.Total) };
```

Keep queries as `IQueryable` to push work to the database. Materialize with `ToList()` when the result is needed.

## Async / Await

```csharp
// Always flow CancellationToken through the call chain
public async Task<List<Product>> GetProductsAsync(string? category, CancellationToken ct = default)
{
    var query = _dbContext.Products.AsQueryable();
    if (!string.IsNullOrEmpty(category))
        query = query.Where(p => p.Category == category);
    return await query.ToListAsync(ct);
}

// Use ValueTask when the result is often synchronous
public ValueTask<Product?> GetCachedProductAsync(int id)
{
    if (_cache.TryGetValue(id, out var product))
        return ValueTask.FromResult<Product?>(product);
    return new ValueTask<Product?>(LoadProductAsync(id));
}

// IAsyncEnumerable for streaming results
public async IAsyncEnumerable<LogEntry> StreamLogsAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var entry in _logSource.ReadAllAsync(ct))
        yield return entry;
}
```

Avoid: `async void` (except event handlers), `.Result` or `.Wait()` (causes deadlocks), missing `CancellationToken`.

## Generics and Constraints

```csharp
public interface IRepository<T> where T : class, IEntity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
}

public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;
    private Result(T? value, string? error) => (Value, Error) = (value, error);
    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);
    public TOut Match<TOut>(Func<T, TOut> success, Func<string, TOut> failure) =>
        IsSuccess ? success(Value!) : failure(Error!);
}
```

## Minimal APIs

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();
var app = builder.Build();

var products = app.MapGroup("/api/products").WithTags("Products").RequireAuthorization();

products.MapGet("/", async (IProductService svc, CancellationToken ct) =>
    Results.Ok(await svc.GetAllAsync(ct)));

products.MapGet("/{id:int}", async (int id, IProductService svc, CancellationToken ct) =>
    await svc.GetByIdAsync(id, ct) is { } product
        ? Results.Ok(product)
        : Results.NotFound());

products.MapPost("/", async (CreateProductRequest req, IProductService svc, CancellationToken ct) =>
{
    var created = await svc.CreateAsync(req, ct);
    return Results.Created($"/api/products/{created.Id}", created);
}).AddEndpointFilter<ValidationFilter<CreateProductRequest>>();

app.Run();
```

## References

- [SOLID Patterns in C#](references/solid-patterns.md) — Interface-based design, dependency inversion, and SOLID principle implementation with real C# examples.
- [C# and .NET Best Practices](references/best-practices.md) — .NET 8+ features, performance with Span and ArrayPool, nullable types, async best practices, and security guidance.
