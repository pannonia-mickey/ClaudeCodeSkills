---
name: C# Mastery
description: This skill should be used when the user asks to "write a C# class", "create a C# interface", "compose a LINQ query", "implement C# async/await", "define a C# record", "use pattern matching", or "build a .NET minimal API". It covers C# 12+ language features, idiomatic patterns, and core .NET framework usage.
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

Use primary constructors on classes and structs to reduce boilerplate for dependency injection and initialization.

```csharp
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<Order?> GetOrderAsync(int id, CancellationToken ct = default)
    {
        logger.LogInformation("Fetching order {OrderId}", id);
        return await repository.GetByIdAsync(id, ct);
    }
}

// Primary constructor with validation
public class PositiveAmount(decimal value)
{
    public decimal Value { get; } = value > 0
        ? value
        : throw new ArgumentOutOfRangeException(nameof(value), "Must be positive");
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

Enable nullable reference types project-wide and handle nullability explicitly. Treat every nullable annotation as a contract.

```csharp
// Nullable-aware API design
public class UserService(IUserRepository repository)
{
    // Return type signals the value may be absent
    public async Task<User?> FindByEmailAsync(string email, CancellationToken ct)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(email);
        return await repository.FindByEmailAsync(email, ct);
    }

    // Non-nullable return — throw if not found
    public async Task<User> GetByIdAsync(int id, CancellationToken ct)
    {
        return await repository.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"User {id} not found");
    }
}
```

### Collection Expressions and Inline Arrays

Use collection expressions for concise initialization.

```csharp
// Collection expressions (C# 12)
List<string> names = ["Alice", "Bob", "Charlie"];
int[] primes = [2, 3, 5, 7, 11];
ReadOnlySpan<byte> header = [0x48, 0x54, 0x54, 0x50];

// Spread operator
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]
```

## LINQ

Write LINQ with intent. Prefer method syntax for complex queries; use query syntax when joins or multiple `from` clauses improve readability.

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
    select new
    {
        Customer = g.Key.Name,
        g.Key.Region,
        OrderCount = g.Count(),
        TotalRevenue = g.Sum(o => o.Total)
    };

// IQueryable vs IEnumerable — keep queries as IQueryable to push work to the database
IQueryable<Product> GetExpensiveProducts(AppDbContext db) =>
    db.Products
        .Where(p => p.Price > 100)
        .OrderBy(p => p.Name);
```

Understand deferred execution: LINQ queries are not evaluated until enumerated. Materialize with `ToList()`, `ToArray()`, or `First()` when the result is needed immediately or when the underlying data source may change.

## Async / Await

Follow these rules for correct, efficient async code.

```csharp
// Always flow CancellationToken through the call chain
public async Task<List<Product>> GetProductsAsync(
    string? category,
    CancellationToken ct = default)
{
    var query = _dbContext.Products.AsQueryable();

    if (!string.IsNullOrEmpty(category))
        query = query.Where(p => p.Category == category);

    return await query.ToListAsync(ct);
}

// Use ValueTask when the result is often synchronous (caching, buffered reads)
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
    {
        yield return entry;
    }
}
```

Avoid these common mistakes:
- Never use `async void` except for event handlers. Always return `Task` or `ValueTask`.
- Never call `.Result` or `.Wait()` on a task from synchronous code — it causes deadlocks.
- Always pass `CancellationToken` through every async method signature.
- Use `Task.WhenAll` for concurrent independent operations, not sequential awaits.

## Generics and Constraints

Design generic APIs with meaningful constraints.

```csharp
// Generic repository with constraints
public interface IRepository<T> where T : class, IEntity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
}

// Generic result type
public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T? value, string? error) => (Value, Error) = (value, error);

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<string, TOut> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}
```

## Minimal APIs

Build lightweight HTTP APIs with route groups, filters, and typed results.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

var products = app.MapGroup("/api/products")
    .WithTags("Products")
    .RequireAuthorization();

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

## Interface Design

Design interfaces that are small, focused, and composable.

```csharp
// Role interfaces — segregated by capability
public interface IReadRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> ListAsync(CancellationToken ct = default);
}

public interface IWriteRepository<T> where T : class
{
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
}

// Compose interfaces for full access
public interface IRepository<T> : IReadRepository<T>, IWriteRepository<T>
    where T : class;
```

## Error Handling

Prefer typed results or problem details over naked exceptions for expected failure modes.

```csharp
// Problem Details for API errors (RFC 9457)
app.MapGet("/api/orders/{id}", async (int id, IOrderService svc, CancellationToken ct) =>
{
    var result = await svc.GetOrderAsync(id, ct);
    return result.Match<IResult>(
        success: order => Results.Ok(order),
        failure: error => Results.Problem(
            title: "Order not found",
            detail: error.Message,
            statusCode: StatusCodes.Status404NotFound));
});
```

## References

- [SOLID Patterns in C#](references/solid-patterns.md) — Interface-based design, dependency inversion, and SOLID principle implementation with real C# examples.
- [C# and .NET Best Practices](references/best-practices.md) — .NET 8+ features, performance with Span and ArrayPool, nullable types, async best practices, and security guidance.
