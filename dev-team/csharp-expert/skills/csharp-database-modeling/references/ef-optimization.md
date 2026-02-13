# EF Core Optimization

## Loading Strategies

### Eager Loading

Load related data in the initial query using `Include` and `ThenInclude`. Use for data that is always needed together.

```csharp
// Single Include
var order = await dbContext.Orders
    .Include(o => o.Items)
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);

// Nested Include
var order = await dbContext.Orders
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Category)
    .Include(o => o.Customer)
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);

// Filtered Include (EF Core 5+) — load only active items
var order = await dbContext.Orders
    .Include(o => o.Items.Where(i => !i.IsCancelled))
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);
```

### Split Queries

When a query includes multiple collection navigations, the default single-query approach can produce a cartesian product. Split queries execute separate SQL statements instead.

```csharp
// Without split query: generates cartesian product (rows = Items x Shipments)
// With split query: generates 3 separate SQL statements
var orders = await dbContext.Orders
    .Include(o => o.Items)
    .Include(o => o.Shipments)
    .Include(o => o.StatusHistory)
    .AsSplitQuery()
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync(ct);

// Configure split queries as default for the DbContext
options.UseSqlServer(connectionString, o => o.UseQuerySplittingBehavior(
    QuerySplittingBehavior.SplitQuery));
```

### Explicit Loading

Load related data on demand after the entity is already tracked.

```csharp
var order = await dbContext.Orders.FindAsync([orderId], ct);

if (order is not null)
{
    // Load collection navigation
    await dbContext.Entry(order)
        .Collection(o => o.Items)
        .LoadAsync(ct);

    // Load reference navigation
    await dbContext.Entry(order)
        .Reference(o => o.Customer)
        .LoadAsync(ct);

    // Filtered explicit loading
    await dbContext.Entry(order)
        .Collection(o => o.Items)
        .Query()
        .Where(i => i.Quantity > 5)
        .LoadAsync(ct);
}
```

### Lazy Loading

Lazy loading loads related data automatically when a navigation property is accessed. Avoid in web applications — it causes N+1 query problems and is difficult to control.

```csharp
// If you must use lazy loading, configure it explicitly
services.AddDbContext<AppDbContext>(options =>
    options.UseLazyLoadingProxies()
           .UseSqlServer(connectionString));

// Prefer projection over lazy loading
var orderSummaries = await dbContext.Orders
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,    // Translated to JOIN, not lazy load
        ItemCount = o.Items.Count,          // Translated to subquery
        Total = o.Total
    })
    .ToListAsync(ct);
```

## Compiled Queries

Pre-compile LINQ queries to avoid the overhead of translating them to SQL on every execution. Use for hot-path queries that run frequently.

```csharp
public class ProductRepository(AppDbContext dbContext) : IProductRepository
{
    // Compiled query — translation happens once, reused on every call
    private static readonly Func<AppDbContext, int, CancellationToken, Task<Product?>>
        GetByIdQuery = EF.CompileAsyncQuery(
            (AppDbContext ctx, int id, CancellationToken ct) =>
                ctx.Products
                    .Include(p => p.Category)
                    .FirstOrDefault(p => p.Id == id));

    private static readonly Func<AppDbContext, string, IAsyncEnumerable<Product>>
        GetByCategoryQuery = EF.CompileAsyncQuery(
            (AppDbContext ctx, string category) =>
                ctx.Products
                    .Where(p => p.Category.Name == category)
                    .OrderBy(p => p.Name));

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct) =>
        GetByIdQuery(dbContext, id, ct);

    public async Task<List<Product>> GetByCategoryAsync(string category, CancellationToken ct)
    {
        var results = new List<Product>();
        await foreach (var product in GetByCategoryQuery(dbContext, category)
            .WithCancellation(ct))
        {
            results.Add(product);
        }
        return results;
    }
}
```

## Raw SQL and Interpolated Queries

Use raw SQL for queries that cannot be expressed in LINQ or for performance-critical operations.

```csharp
// Safe: interpolated SQL is parameterized automatically
var products = await dbContext.Products
    .FromSqlInterpolated($"""
        SELECT p.*
        FROM Products p
        INNER JOIN Categories c ON p.CategoryId = c.Id
        WHERE c.Name = {categoryName}
          AND p.Price BETWEEN {minPrice} AND {maxPrice}
        ORDER BY p.Name
        """)
    .ToListAsync(ct);

// Raw SQL with explicit parameters
var parameters = new[]
{
    new SqlParameter("@status", "Active"),
    new SqlParameter("@minDate", DateTime.UtcNow.AddDays(-30))
};

var recentOrders = await dbContext.Orders
    .FromSqlRaw("""
        SELECT * FROM Orders
        WHERE Status = @status AND CreatedAt >= @minDate
        """, parameters)
    .Include(o => o.Items)
    .OrderByDescending(o => o.CreatedAt)
    .ToListAsync(ct);

// Non-entity SQL with SqlQuery (EF Core 8+)
var topCategories = await dbContext.Database
    .SqlQuery<CategorySummary>($"""
        SELECT c.Name, COUNT(p.Id) AS ProductCount, AVG(p.Price) AS AveragePrice
        FROM Categories c
        INNER JOIN Products p ON p.CategoryId = c.Id
        GROUP BY c.Name
        HAVING COUNT(p.Id) > {minProducts}
        ORDER BY COUNT(p.Id) DESC
        """)
    .ToListAsync(ct);
```

## Bulk Operations

EF Core 7+ supports `ExecuteUpdate` and `ExecuteDelete` for efficient bulk operations that bypass change tracking.

```csharp
// Bulk update — single SQL UPDATE statement
var affected = await dbContext.Products
    .Where(p => p.Category.Name == "Discontinued")
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.ModifiedAt, DateTime.UtcNow),
        ct);

// Bulk delete — single SQL DELETE statement
var deleted = await dbContext.Orders
    .Where(o => o.Status == OrderStatus.Cancelled
             && o.CreatedAt < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync(ct);

// Conditional bulk update
await dbContext.Products
    .Where(p => p.Price > 100)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 0.9m), // 10% discount
        ct);
```

For very large bulk inserts (tens of thousands of rows), consider using a third-party library like EFCore.BulkExtensions or direct ADO.NET with SqlBulkCopy.

```csharp
// SqlBulkCopy for massive inserts
public async Task BulkInsertProductsAsync(
    IEnumerable<Product> products,
    CancellationToken ct)
{
    var table = new DataTable();
    table.Columns.Add("Name", typeof(string));
    table.Columns.Add("Price", typeof(decimal));
    table.Columns.Add("CategoryId", typeof(int));

    foreach (var p in products)
        table.Rows.Add(p.Name, p.Price, p.CategoryId);

    var connection = (SqlConnection)dbContext.Database.GetDbConnection();
    await connection.OpenAsync(ct);

    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "Products",
        BatchSize = 5000
    };

    await bulkCopy.WriteToServerAsync(table, ct);
}
```

## Change Tracking Optimization

### No-Tracking Queries

Disable change tracking for read-only queries to reduce memory and CPU overhead.

```csharp
// Per-query no-tracking
var products = await dbContext.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync(ct);

// No-tracking with identity resolution (deduplicates entities in results)
var orders = await dbContext.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .ToListAsync(ct);

// Default no-tracking for the entire context (read-heavy scenarios)
services.AddDbContext<ReadOnlyDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

### Efficient Updates

```csharp
// Attach and mark specific properties as modified (avoids loading the entity)
public async Task UpdatePriceAsync(int productId, decimal newPrice, CancellationToken ct)
{
    var product = new Product { Id = productId };
    dbContext.Attach(product);
    product.Price = newPrice;
    dbContext.Entry(product).Property(p => p.Price).IsModified = true;

    await dbContext.SaveChangesAsync(ct);
}
```

## Interceptors

### Command Interceptor for Query Logging

```csharp
public class SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger) : DbCommandInterceptor
{
    private const int SlowQueryThresholdMs = 500;

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken ct = default)
    {
        if (eventData.Duration.TotalMilliseconds > SlowQueryThresholdMs)
        {
            logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                (int)eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }

        return result;
    }
}
```

### Connection Interceptor for Multi-Tenancy

```csharp
public class TenantConnectionInterceptor(ITenantProvider tenantProvider)
    : DbConnectionInterceptor
{
    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct = default)
    {
        // Set tenant context at the database level
        var tenantId = tenantProvider.TenantId;
        await using var command = connection.CreateCommand();
        command.CommandText = $"EXEC sp_set_session_context @key=N'TenantId', @value=@tenantId";
        var param = command.CreateParameter();
        param.ParameterName = "@tenantId";
        param.Value = tenantId;
        command.Parameters.Add(param);
        await command.ExecuteNonQueryAsync(ct);
    }
}
```

### SaveChanges Interceptor for Domain Events

```csharp
public class DomainEventInterceptor(IMediator mediator) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return result;

        var domainEvents = eventData.Context.ChangeTracker
            .Entries<IHasDomainEvents>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        foreach (var domainEvent in domainEvents)
        {
            await mediator.Publish(domainEvent, ct);
        }

        // Clear events after publishing
        foreach (var entry in eventData.Context.ChangeTracker.Entries<IHasDomainEvents>())
        {
            entry.Entity.ClearDomainEvents();
        }

        return result;
    }
}
```

## Query Performance Tips

### Use Indexes Strategically

```csharp
// Single column index
builder.HasIndex(p => p.Name);

// Composite index
builder.HasIndex(o => new { o.CustomerId, o.CreatedAt });

// Unique index
builder.HasIndex(u => u.Email).IsUnique();

// Filtered index (only index active records)
builder.HasIndex(p => p.Name)
    .HasFilter("[IsActive] = 1");

// Include columns (covering index)
builder.HasIndex(p => p.CategoryId)
    .IncludeProperties(p => new { p.Name, p.Price });
```

### Pagination

```csharp
// Keyset pagination (more efficient than OFFSET for large datasets)
public async Task<List<ProductDto>> GetProductsAfterAsync(
    int? lastId,
    int pageSize = 20,
    CancellationToken ct = default)
{
    var query = dbContext.Products
        .AsNoTracking()
        .OrderBy(p => p.Id);

    if (lastId.HasValue)
        query = (IOrderedQueryable<Product>)query.Where(p => p.Id > lastId.Value);

    return await query
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync(ct);
}

// Traditional offset pagination (simpler but slower for deep pages)
public async Task<PagedResult<ProductDto>> GetProductsPagedAsync(
    int page,
    int pageSize = 20,
    CancellationToken ct = default)
{
    var totalCount = await dbContext.Products.CountAsync(ct);

    var items = await dbContext.Products
        .AsNoTracking()
        .OrderBy(p => p.Id)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync(ct);

    return new PagedResult<ProductDto>(items, totalCount, page, pageSize);
}
```
