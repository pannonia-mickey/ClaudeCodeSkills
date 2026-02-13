# C# and .NET Best Practices

## .NET 8+ Features

### Top-Level Statements and Minimal Hosting

Use the minimal hosting model for new applications. Keep `Program.cs` as an orchestrator that delegates to extension methods.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Delegate registration to focused extension methods
builder.Services
    .AddApplicationServices()
    .AddInfrastructureServices(builder.Configuration)
    .AddPresentation();

builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration));

var app = builder.Build();

app.UseExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

### File-Scoped Namespaces

Always use file-scoped namespaces to reduce indentation.

```csharp
namespace MyApp.Domain.Entities;

public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public decimal Price { get; set; }
}
```

### Required Members

Use the `required` modifier to enforce initialization without constructor parameters.

```csharp
public class CreateProductRequest
{
    public required string Name { get; init; }
    public required decimal Price { get; init; }
    public string? Description { get; init; }
    public required string Category { get; init; }
}
```

### Raw String Literals

Use raw string literals for multi-line strings, SQL, JSON, and regex.

```csharp
var query = """
    SELECT p.Id, p.Name, p.Price
    FROM Products p
    WHERE p.Category = @Category
      AND p.IsActive = 1
    ORDER BY p.Name
    """;

var json = """
    {
        "name": "Test Product",
        "price": 29.99,
        "category": "Electronics"
    }
    """;
```

### Global Using Directives

Place global usings in a `GlobalUsings.cs` file at the project root.

```csharp
// GlobalUsings.cs
global using System.ComponentModel.DataAnnotations;
global using Microsoft.EntityFrameworkCore;
global using Microsoft.Extensions.Logging;
global using MyApp.Domain.Entities;
global using MyApp.Domain.Interfaces;
```

## Nullable Reference Types

### Project-Wide Enablement

Enable nullable reference types in the project file. Treat every warning as an error in CI.

```xml
<PropertyGroup>
    <Nullable>enable</Nullable>
    <WarningsAsErrors>nullable</WarningsAsErrors>
</PropertyGroup>
```

### Defensive Patterns

```csharp
// Use ArgumentNullException.ThrowIfNull for guard clauses
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    ArgumentException.ThrowIfNullOrWhiteSpace(order.CustomerName);
    // ...
}

// Use null-conditional and null-coalescing operators
public string GetDisplayName(User? user) =>
    user?.DisplayName ?? user?.Email ?? "Anonymous";

// Pattern match for null checks
if (await repository.GetByIdAsync(id, ct) is { } entity)
{
    // entity is guaranteed non-null here
    return entity.ToDto();
}

// Use [NotNullWhen] attribute for try-pattern methods
public bool TryGetProduct(int id, [NotNullWhen(true)] out Product? product)
{
    product = _cache.GetValueOrDefault(id);
    return product is not null;
}
```

### Nullable in Generics

```csharp
// Use the default constraint to allow nullable value types
public T? FirstOrDefault<T>(IEnumerable<T> source) where T : struct
{
    foreach (var item in source)
        return item;
    return null;
}

// Use notnull constraint when null is not acceptable
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _store = new();

    public TValue? Get(TKey key) =>
        _store.GetValueOrDefault(key);

    public void Set(TKey key, TValue value) =>
        _store[key] = value;
}
```

## Performance Best Practices

### Span<T> and Memory<T>

Use `Span<T>` and `ReadOnlySpan<T>` for zero-allocation slicing of contiguous memory.

```csharp
// Parse without allocating substrings
public static (string Key, string Value) ParseHeader(ReadOnlySpan<char> header)
{
    var separatorIndex = header.IndexOf(':');
    if (separatorIndex < 0)
        throw new FormatException("Invalid header format");

    var key = header[..separatorIndex].Trim().ToString();
    var value = header[(separatorIndex + 1)..].Trim().ToString();
    return (key, value);
}

// Stack-allocated buffers for small data
public static string FormatId(int id)
{
    Span<char> buffer = stackalloc char[16];
    if (id.TryFormat(buffer, out var written))
        return new string(buffer[..written]);
    return id.ToString();
}
```

### ArrayPool and Object Pooling

```csharp
// Rent arrays from the pool to avoid GC pressure
public static double ComputeAverage(IReadOnlyList<double> values)
{
    var buffer = ArrayPool<double>.Shared.Rent(values.Count);
    try
    {
        for (int i = 0; i < values.Count; i++)
            buffer[i] = values[i];

        return buffer.AsSpan(0, values.Count).ToArray().Average();
    }
    finally
    {
        ArrayPool<double>.Shared.Return(buffer);
    }
}

// Use ObjectPool for expensive-to-create objects
services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
services.AddSingleton(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.Create(new StringBuilderPooledObjectPolicy());
});
```

### String Performance

```csharp
// Use string.Create for building strings without intermediate allocations
public static string CreateGreeting(string name, int age) =>
    string.Create(null, $"Hello, {name}! You are {age} years old.");

// Use StringComparison explicitly — avoids culture-dependent bugs
bool isMatch = input.Equals("expected", StringComparison.OrdinalIgnoreCase);
int index = text.IndexOf("needle", StringComparison.Ordinal);

// Use StringBuilder for loops
public static string JoinWithCommas(IEnumerable<string> items)
{
    var sb = new StringBuilder();
    foreach (var item in items)
    {
        if (sb.Length > 0) sb.Append(", ");
        sb.Append(item);
    }
    return sb.ToString();
}

// Prefer SearchValues for repeated character searches (.NET 8+)
private static readonly SearchValues<char> s_vowels =
    SearchValues.Create("aeiouAEIOU");

public static int CountVowels(ReadOnlySpan<char> text)
{
    int count = 0;
    foreach (var ch in text)
    {
        if (s_vowels.Contains(ch))
            count++;
    }
    return count;
}
```

### Collection Performance

```csharp
// Use FrozenDictionary / FrozenSet for read-heavy lookup tables (.NET 8+)
private static readonly FrozenDictionary<string, int> StatusCodes =
    new Dictionary<string, int>
    {
        ["OK"] = 200,
        ["Created"] = 201,
        ["NotFound"] = 404,
        ["ServerError"] = 500
    }.ToFrozenDictionary();

// Pre-size collections when the count is known
var results = new List<ProductDto>(products.Count);
foreach (var p in products)
    results.Add(p.ToDto());

// Use CollectionsMarshal.GetValueRefOrAddDefault for dictionary update patterns
ref var count = ref CollectionsMarshal.GetValueRefOrAddDefault(
    wordCounts, word, out bool existed);
count++;
```

## Async Best Practices

### Cancellation Token Propagation

Pass `CancellationToken` through every async method. Accept it as the last parameter with a default value.

```csharp
public async Task<List<OrderDto>> GetRecentOrdersAsync(
    int customerId,
    int count = 10,
    CancellationToken ct = default)
{
    var orders = await _dbContext.Orders
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt)
        .Take(count)
        .Select(o => new OrderDto(o.Id, o.Total, o.CreatedAt))
        .ToListAsync(ct);

    return orders;
}
```

### Parallel Async Operations

```csharp
// Use Task.WhenAll for independent concurrent operations
public async Task<DashboardData> GetDashboardAsync(int userId, CancellationToken ct)
{
    var ordersTask = _orderService.GetRecentAsync(userId, ct);
    var notificationsTask = _notificationService.GetUnreadAsync(userId, ct);
    var statsTask = _statsService.GetSummaryAsync(userId, ct);

    await Task.WhenAll(ordersTask, notificationsTask, statsTask);

    return new DashboardData(
        Orders: await ordersTask,
        Notifications: await notificationsTask,
        Stats: await statsTask);
}
```

### Avoid Common Async Pitfalls

```csharp
// WRONG: sync-over-async causes deadlocks
public Order GetOrder(int id) =>
    _orderService.GetOrderAsync(id).Result; // DEADLOCK RISK

// CORRECT: async all the way
public async Task<Order> GetOrderAsync(int id, CancellationToken ct) =>
    await _orderService.GetOrderAsync(id, ct);

// WRONG: async void — exceptions are unobservable
async void OnButtonClick() => await DoWorkAsync(); // NEVER DO THIS

// CORRECT: return Task
async Task OnButtonClickAsync() => await DoWorkAsync();

// WRONG: unnecessary async/await (adds state machine overhead)
public async Task<User> GetUserAsync(int id) =>
    await _repository.GetByIdAsync(id); // Unnecessary wrapper

// CORRECT: pass through the Task directly when no additional logic is needed
public Task<User> GetUserAsync(int id) =>
    _repository.GetByIdAsync(id);
```

### IAsyncEnumerable for Streaming

```csharp
// Stream large result sets without loading everything into memory
public async IAsyncEnumerable<ProductDto> StreamProductsAsync(
    string category,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var product in _dbContext.Products
        .Where(p => p.Category == category)
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return product.ToDto();
    }
}

// Consume in a minimal API endpoint
app.MapGet("/api/products/stream", (string category, AppDbContext db, CancellationToken ct) =>
    Results.Ok(StreamProducts(db, category, ct)));
```

## Security Best Practices

### Input Validation

```csharp
// Use FluentValidation or DataAnnotations for input validation
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(256);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(12)
            .Matches("[A-Z]").WithMessage("Must contain uppercase letter")
            .Matches("[0-9]").WithMessage("Must contain digit");

        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100)
            .Matches(@"^[\w\s\-']+$").WithMessage("Contains invalid characters");
    }
}
```

### Secrets and Configuration

```csharp
// Never hardcode secrets — use user-secrets in development, Key Vault in production
// appsettings.json — no secrets here
{
    "ConnectionStrings": {
        "Default": "" // Set via environment variable or Key Vault
    }
}

// Program.cs — add Key Vault in production
if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVault:Uri"]!;
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        new DefaultAzureCredential());
}
```

### SQL Injection Prevention

```csharp
// ALWAYS use parameterized queries
// CORRECT: EF Core parameterizes automatically
var products = await _dbContext.Products
    .Where(p => p.Name.Contains(searchTerm))
    .ToListAsync(ct);

// CORRECT: parameterized raw SQL
var products = await _dbContext.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name LIKE {'%' + searchTerm + '%'}")
    .ToListAsync(ct);

// WRONG: string concatenation in SQL — SQL injection vulnerability
var products = await _dbContext.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Name LIKE '%{searchTerm}%'") // NEVER
    .ToListAsync(ct);
```

### Authentication and Authorization

```csharp
// Use policy-based authorization
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CanManageProducts", policy =>
        policy.RequireRole("Admin", "ProductManager"))
    .AddPolicy("CanViewReports", policy =>
        policy.RequireClaim("permission", "reports:read"));

// Apply on endpoints
app.MapDelete("/api/products/{id}", async (int id, IProductService svc, CancellationToken ct) =>
{
    await svc.DeleteAsync(id, ct);
    return Results.NoContent();
}).RequireAuthorization("CanManageProducts");

// Resource-based authorization
public class OrderAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.CustomerId.ToString() == userId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

## Logging Best Practices

```csharp
// Use structured logging with message templates — never string interpolation
logger.LogInformation(
    "Processing order {OrderId} for customer {CustomerId} with {ItemCount} items",
    order.Id, order.CustomerId, order.Items.Count);

// Use LoggerMessage.Define for high-performance logging
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} processed in {ElapsedMs}ms")]
    public static partial void OrderProcessed(ILogger logger, int orderId, long elapsedMs);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Payment failed for order {OrderId}: {Reason}")]
    public static partial void PaymentFailed(ILogger logger, int orderId, string reason);
}

// Usage
LogMessages.OrderProcessed(logger, order.Id, stopwatch.ElapsedMilliseconds);
```

## Exception Handling

```csharp
// Use global exception handling middleware
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();

        logger.LogError(exception, "Unhandled exception");

        context.Response.StatusCode = exception switch
        {
            NotFoundException => StatusCodes.Status404NotFound,
            ValidationException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError
        };

        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = context.Response.StatusCode,
            Title = "An error occurred",
            Detail = context.Environment.IsDevelopment() ? exception?.Message : null
        });
    });
});

// Define domain-specific exceptions
public class NotFoundException(string message) : Exception(message);
public class ConflictException(string message) : Exception(message);
public class BusinessRuleException(string message) : Exception(message);
```
