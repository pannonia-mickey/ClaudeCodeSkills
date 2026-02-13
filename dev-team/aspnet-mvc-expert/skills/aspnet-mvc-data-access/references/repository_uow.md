# Repository & Unit of Work Advanced Patterns

Advanced abstractions for data access in ASP.NET Core MVC.

## Generic Repository with Specification Support

```csharp
public interface IRepository<T> where T : Entity
{
    // Basic CRUD
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    void Add(T entity);
    void AddRange(IEnumerable<T> entities);
    void Update(T entity);
    void Remove(T entity);

    // Specification-based queries
    Task<T?> FirstOrDefaultAsync(
        Specification<T> spec, CancellationToken ct = default);
    Task<IReadOnlyList<T>> ListAsync(
        Specification<T> spec, CancellationToken ct = default);
    Task<int> CountAsync(
        Specification<T> spec, CancellationToken ct = default);
    Task<bool> AnyAsync(
        Specification<T> spec, CancellationToken ct = default);
}

public class Repository<T> : IRepository<T> where T : Entity
{
    protected readonly AppDbContext Db;
    protected readonly DbSet<T> DbSet;

    public Repository(AppDbContext db)
    {
        Db = db;
        DbSet = db.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await DbSet.FindAsync(new object[] { id }, ct);

    public void Add(T entity) => DbSet.Add(entity);
    public void AddRange(IEnumerable<T> entities) => DbSet.AddRange(entities);
    public void Update(T entity) => DbSet.Update(entity);
    public void Remove(T entity) => DbSet.Remove(entity);

    public async Task<T?> FirstOrDefaultAsync(
        Specification<T> spec, CancellationToken ct = default)
        => await ApplySpecification(spec).FirstOrDefaultAsync(ct);

    public async Task<IReadOnlyList<T>> ListAsync(
        Specification<T> spec, CancellationToken ct = default)
        => await ApplySpecification(spec).ToListAsync(ct);

    public async Task<int> CountAsync(
        Specification<T> spec, CancellationToken ct = default)
        => await ApplySpecification(spec).CountAsync(ct);

    public async Task<bool> AnyAsync(
        Specification<T> spec, CancellationToken ct = default)
        => await ApplySpecification(spec).AnyAsync(ct);

    private IQueryable<T> ApplySpecification(Specification<T> spec)
        => SpecificationEvaluator<T>.GetQuery(DbSet.AsQueryable(), spec);
}
```

## Paginated Result

```csharp
public class PaginatedList<T>
{
    public IReadOnlyList<T> Items { get; }
    public int PageNumber { get; }
    public int TotalPages { get; }
    public int TotalCount { get; }
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    private PaginatedList(
        IReadOnlyList<T> items, int count, int pageNumber, int pageSize)
    {
        Items = items;
        TotalCount = count;
        PageNumber = pageNumber;
        TotalPages = (int)Math.Ceiling(count / (double)pageSize);
    }

    public static async Task<PaginatedList<T>> CreateAsync(
        IQueryable<T> source, int pageNumber, int pageSize,
        CancellationToken ct = default)
    {
        var count = await source.CountAsync(ct);
        var items = await source
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PaginatedList<T>(items, count, pageNumber, pageSize);
    }
}

// Usage in handler
public async Task<PaginatedList<ProductDto>> Handle(
    GetProductsListQuery query, CancellationToken ct)
{
    return await _db.Products
        .AsNoTracking()
        .Where(p => string.IsNullOrEmpty(query.SearchTerm) ||
                     p.Name.Contains(query.SearchTerm))
        .OrderBy(p => p.Name)
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        })
        .ToPaginatedListAsync(query.PageNumber, query.PageSize, ct);
}
```

## Transaction Management

### Explicit Transactions

```csharp
public class TransferFundsHandler : IRequestHandler<TransferFundsCommand>
{
    private readonly AppDbContext _db;

    public TransferFundsHandler(AppDbContext db) => _db = db;

    public async Task Handle(TransferFundsCommand request, CancellationToken ct)
    {
        var strategy = _db.Database.CreateExecutionStrategy();

        await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _db.Database.BeginTransactionAsync(ct);

            try
            {
                var source = await _db.Accounts.FindAsync(request.SourceId)
                    ?? throw new NotFoundException("Source account not found");
                var target = await _db.Accounts.FindAsync(request.TargetId)
                    ?? throw new NotFoundException("Target account not found");

                source.Withdraw(request.Amount);
                target.Deposit(request.Amount);

                await _db.SaveChangesAsync(ct);
                await transaction.CommitAsync(ct);
            }
            catch
            {
                await transaction.RollbackAsync(ct);
                throw;
            }
        });
    }
}
```

### TransactionScope for Cross-Context Operations

```csharp
public async Task ProcessOrderAsync(Order order, CancellationToken ct)
{
    using var scope = new TransactionScope(
        TransactionScopeOption.Required,
        new TransactionOptions
        {
            IsolationLevel = IsolationLevel.ReadCommitted,
            Timeout = TimeSpan.FromSeconds(30)
        },
        TransactionScopeAsyncFlowOption.Enabled);

    // Operation 1: Save order
    _orderContext.Orders.Add(order);
    await _orderContext.SaveChangesAsync(ct);

    // Operation 2: Update inventory
    await _inventoryService.ReserveStockAsync(order.Items, ct);

    scope.Complete(); // Commit both operations
}
```

## Multi-Context Scenarios

When an application requires multiple databases:

```csharp
// Primary database context
public class AppDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
}

// Reporting/read-only database context
public class ReportingDbContext : DbContext
{
    public ReportingDbContext(DbContextOptions<ReportingDbContext> options)
        : base(options) { }

    // Read-only — disable change tracking globally
    protected override void OnConfiguring(DbContextOptionsBuilder builder)
    {
        builder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }

    public DbSet<SalesReport> SalesReports => Set<SalesReport>();
}

// Registration
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(config.GetConnectionString("Primary")));

services.AddDbContext<ReportingDbContext>(options =>
    options.UseSqlServer(config.GetConnectionString("Reporting")));
```

## Dapper for Performance-Critical Queries

When EF Core overhead is unacceptable, use Dapper alongside EF:

```csharp
public interface IDapperQueryService
{
    Task<IReadOnlyList<T>> QueryAsync<T>(
        string sql, object? parameters = null, CancellationToken ct = default);
    Task<T?> QueryFirstOrDefaultAsync<T>(
        string sql, object? parameters = null, CancellationToken ct = default);
}

public class DapperQueryService : IDapperQueryService
{
    private readonly IDbConnectionFactory _connectionFactory;

    public DapperQueryService(IDbConnectionFactory connectionFactory)
        => _connectionFactory = connectionFactory;

    public async Task<IReadOnlyList<T>> QueryAsync<T>(
        string sql, object? parameters = null, CancellationToken ct = default)
    {
        await using var connection = await _connectionFactory.CreateConnectionAsync(ct);
        var result = await connection.QueryAsync<T>(
            new CommandDefinition(sql, parameters, cancellationToken: ct));
        return result.ToList().AsReadOnly();
    }

    public async Task<T?> QueryFirstOrDefaultAsync<T>(
        string sql, object? parameters = null, CancellationToken ct = default)
    {
        await using var connection = await _connectionFactory.CreateConnectionAsync(ct);
        return await connection.QueryFirstOrDefaultAsync<T>(
            new CommandDefinition(sql, parameters, cancellationToken: ct));
    }
}

// Usage — complex reporting query
var report = await _dapper.QueryAsync<SalesSummaryDto>(
    @"SELECT
        c.Name AS CategoryName,
        COUNT(o.Id) AS OrderCount,
        SUM(oi.Quantity * oi.UnitPrice) AS TotalRevenue
    FROM Orders o
    INNER JOIN OrderItems oi ON o.Id = oi.OrderId
    INNER JOIN Products p ON oi.ProductId = p.Id
    INNER JOIN Categories c ON p.CategoryId = c.Id
    WHERE o.PlacedAt >= @StartDate AND o.PlacedAt <= @EndDate
    GROUP BY c.Name
    ORDER BY TotalRevenue DESC",
    new { StartDate = startDate, EndDate = endDate });
```

## Choosing the Right Pattern

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple CRUD, small app | DbContext directly in handlers |
| Medium app, testability needed | Generic Repository + Unit of Work |
| Complex domain with invariants | Specialized Repositories + Domain Entities |
| Complex query composition | Specification Pattern |
| Performance-critical reads | Dapper or raw SQL |
| Multiple databases | Multiple DbContexts |
| Event-driven side effects | Domain Events + Interceptors |
| Audit trail | Interceptors + Temporal Tables |
