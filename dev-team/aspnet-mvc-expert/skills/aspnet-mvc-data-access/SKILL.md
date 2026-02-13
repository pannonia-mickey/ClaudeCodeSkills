---
name: aspnet-mvc-data-access
description: This skill should be used when the user asks to "set up Entity Framework", "create a DbContext", "add migrations", "implement repository pattern", "optimize EF queries", "fix N+1 problem", "configure database connection", "add Unit of Work", "implement soft delete", "set up audit logging", "create entity configurations", or needs guidance on data access patterns in ASP.NET Core MVC with Entity Framework Core.
---

# ASP.NET MVC Data Access

Expert guidance for implementing data access in ASP.NET Core MVC using Entity Framework Core, Repository pattern, and query optimization techniques.

## DbContext Setup

### Configuration

```csharp
public class AppDbContext : DbContext, IUnitOfWork
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all IEntityTypeConfiguration<T> in the assembly
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(AppDbContext).Assembly);

        // Global query filter for soft delete
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => !p.IsDeleted);
    }

    // Automatic audit fields
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<AuditableEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTime.UtcNow;
                    break;
            }
        }
        return await base.SaveChangesAsync(ct);
    }
}
```

### Entity Configuration (Fluent API)

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasIndex(p => p.Name);
        builder.HasIndex(p => new { p.CategoryId, p.IsDeleted });
    }
}
```

### Registration

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(10),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        });

    // Development only
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});
```

## Repository Pattern

### Generic Repository

```csharp
public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(
        Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
}

public class Repository<T> : IRepository<T> where T : Entity
{
    protected readonly AppDbContext _db;
    protected readonly DbSet<T> _dbSet;

    public Repository(AppDbContext db)
    {
        _db = db;
        _dbSet = db.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _dbSet.FindAsync(new object[] { id }, ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await _dbSet.ToListAsync(ct);

    public async Task<IReadOnlyList<T>> FindAsync(
        Expression<Func<T, bool>> predicate, CancellationToken ct = default)
        => await _dbSet.Where(predicate).ToListAsync(ct);

    public void Add(T entity) => _dbSet.Add(entity);
    public void Update(T entity) => _dbSet.Update(entity);
    public void Remove(T entity) => _dbSet.Remove(entity);
}
```

### Specialized Repository

```csharp
public interface IProductRepository : IRepository<Product>
{
    Task<IReadOnlyList<Product>> GetByCategoryAsync(int categoryId, CancellationToken ct);
    Task<Product?> GetWithDetailsAsync(int id, CancellationToken ct);
}

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(AppDbContext db) : base(db) { }

    public async Task<IReadOnlyList<Product>> GetByCategoryAsync(
        int categoryId, CancellationToken ct)
        => await _dbSet
            .Where(p => p.CategoryId == categoryId)
            .OrderBy(p => p.Name)
            .ToListAsync(ct);

    public async Task<Product?> GetWithDetailsAsync(int id, CancellationToken ct)
        => await _dbSet
            .Include(p => p.Category)
            .Include(p => p.Reviews)
            .FirstOrDefaultAsync(p => p.Id == id, ct);
}
```

## Unit of Work

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

// AppDbContext implements IUnitOfWork (see DbContext Setup above)

// Usage in a handler
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderHandler(
        IOrderRepository orders,
        IProductRepository products,
        IUnitOfWork unitOfWork)
    {
        _orders = orders;
        _products = products;
        _unitOfWork = unitOfWork;
    }

    public async Task<int> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var product = await _products.GetByIdAsync(request.ProductId, ct)
            ?? throw new NotFoundException(nameof(Product), request.ProductId);

        var order = new Order(product, request.Quantity);
        _orders.Add(order);

        await _unitOfWork.SaveChangesAsync(ct);  // Single transaction
        return order.Id;
    }
}
```

## Query Optimization

### Solving N+1 Problems

```csharp
// BAD — N+1: loads orders, then queries items for each order
var orders = await _db.Orders.ToListAsync();
foreach (var order in orders)
{
    var items = order.Items; // Lazy load = separate query per order
}

// GOOD — Eager loading with Include
var orders = await _db.Orders
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .ToListAsync();

// BETTER — Projection when only specific fields are needed
var orderSummaries = await _db.Orders
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        Total = o.Items.Sum(i => i.Quantity * i.UnitPrice),
        ItemCount = o.Items.Count
    })
    .ToListAsync();
```

### Performance Patterns

```csharp
// Use AsNoTracking for read-only queries
var products = await _db.Products
    .AsNoTracking()
    .Where(p => p.Price > 100)
    .ToListAsync();

// Use Split Queries for wide Include chains
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .AsSplitQuery()
    .ToListAsync();

// Pagination
var page = await _db.Products
    .OrderBy(p => p.Name)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// Compiled queries for hot paths
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Products.FirstOrDefault(p => p.Id == id));
```

## Migrations Workflow

```bash
# Add migration
dotnet ef migrations add InitialCreate --project src/Infrastructure --startup-project src/WebUI

# Apply migrations
dotnet ef database update --project src/Infrastructure --startup-project src/WebUI

# Generate SQL script for production
dotnet ef migrations script --idempotent --project src/Infrastructure --startup-project src/WebUI -o migrate.sql
```

Apply migrations safely in production:
```csharp
// AVOID auto-migration in production Program.cs
// Instead, use idempotent SQL scripts or a migration tool

// Only acceptable for development:
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
```

## Specification Pattern (Advanced)

For complex query composition:

```csharp
public abstract class Specification<T> where T : Entity
{
    public Expression<Func<T, bool>>? Criteria { get; protected set; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
    public int? Take { get; protected set; }
    public int? Skip { get; protected set; }
}

// Usage
public class ActiveProductsByCategorySpec : Specification<Product>
{
    public ActiveProductsByCategorySpec(int categoryId, int page, int pageSize)
    {
        Criteria = p => p.CategoryId == categoryId && p.IsActive;
        Includes.Add(p => p.Category);
        OrderBy = p => p.Name;
        Skip = (page - 1) * pageSize;
        Take = pageSize;
    }
}
```

## Additional Resources

### Reference Files

For advanced data access patterns, consult:

- **`references/ef_core_patterns.md`** — Interceptors, value converters, owned types, table splitting, temporal tables, concurrency handling
- **`references/repository_uow.md`** — Advanced repository abstractions, generic specification evaluator, transaction management, multi-context scenarios
