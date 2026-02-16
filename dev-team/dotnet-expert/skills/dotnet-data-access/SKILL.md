---
name: .NET Data Access
description: This skill should be used when the user asks about "Entity Framework", "EF Core", "DbContext", "EF migration", "entity configuration", "repository pattern in .NET", "Unit of Work", "query optimization EF", "N+1 problem .NET", "soft delete EF", or "database access in .NET". It covers DbContext configuration, Fluent API entity modeling, migrations, query optimization, Repository and Unit of Work patterns, and EF Core 8+ features.
---

# Entity Framework Core Data Access

## DbContext Configuration

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added: entry.Entity.CreatedAt = DateTime.UtcNow; break;
                case EntityState.Modified: entry.Entity.ModifiedAt = DateTime.UtcNow; break;
            }
        }
        return await base.SaveChangesAsync(ct);
    }
}
```

## Entity Configuration with Fluent API

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.OrderNumber).HasMaxLength(50).IsRequired();
        builder.HasIndex(o => o.OrderNumber).IsUnique();
        builder.Property(o => o.Status).HasConversion<string>().HasMaxLength(20);
        builder.Property(o => o.Total).HasPrecision(18, 2);
        builder.HasMany(o => o.Items).WithOne(i => i.Order).HasForeignKey(i => i.OrderId).OnDelete(DeleteBehavior.Cascade);
        builder.HasOne(o => o.Customer).WithMany(c => c.Orders).HasForeignKey(o => o.CustomerId).OnDelete(DeleteBehavior.Restrict);
        builder.HasQueryFilter(o => !o.IsDeleted);
    }
}
```

## Value Conversions

```csharp
// Strongly-typed ID
public readonly record struct OrderId(Guid Value) { public static OrderId New() => new(Guid.NewGuid()); }

builder.Property(o => o.Id).HasConversion(id => id.Value, value => new OrderId(value));

// Owned types for value objects
builder.OwnsOne(c => c.ShippingAddress, a =>
{
    a.Property(x => x.Street).HasMaxLength(200).IsRequired();
    a.Property(x => x.City).HasMaxLength(100).IsRequired();
});
```

## JSON Columns (EF Core 7+)

```csharp
public class Product { public int Id { get; set; } public ProductMetadata Metadata { get; set; } = new(); }
public class ProductMetadata { public List<string> Tags { get; set; } = []; public DimensionsInfo? Dimensions { get; set; } }

builder.OwnsOne(p => p.Metadata, m => { m.ToJson(); m.OwnsOne(md => md.Dimensions); });
```

## Query Optimization

```csharp
// Projection — only select needed columns
var orders = await dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderListDto { Id = o.Id, Total = o.Total, ItemCount = o.Items.Count })
    .ToListAsync(ct);

// Split queries — avoid cartesian explosion
var orders = await dbContext.Orders.Include(o => o.Items).Include(o => o.Shipments)
    .AsSplitQuery().Where(o => o.Status == OrderStatus.Active).ToListAsync(ct);

// No-tracking for read-only
var products = await dbContext.Products.AsNoTracking().Where(p => p.IsActive).ToListAsync(ct);

// Compiled queries for hot paths
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => db.Products.FirstOrDefault(p => p.Id == id));
```

## Repository Pattern

```csharp
public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    void Add(T entity);
    void Remove(T entity);
}

public class Repository<T>(AppDbContext db) : IRepository<T> where T : Entity
{
    protected readonly DbSet<T> DbSet = db.Set<T>();
    public async Task<T?> GetByIdAsync(int id, CancellationToken ct) => await DbSet.FindAsync([id], ct);
    public async Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct)
        => await DbSet.Where(predicate).ToListAsync(ct);
    public void Add(T entity) => DbSet.Add(entity);
    public void Remove(T entity) => DbSet.Remove(entity);
}
```

## Interceptors

```csharp
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    {
        if (eventData.Context is null) return ValueTask.FromResult(result);
        foreach (var entry in eventData.Context.ChangeTracker.Entries<ISoftDeletable>())
        {
            if (entry.State == EntityState.Deleted)
            {
                entry.State = EntityState.Modified;
                entry.Entity.IsDeleted = true;
                entry.Entity.DeletedAt = DateTime.UtcNow;
            }
        }
        return ValueTask.FromResult(result);
    }
}
```

## Migrations

```bash
dotnet ef migrations add AddOrderTable --project src/Infrastructure --startup-project src/WebApi
dotnet ef database update --project src/Infrastructure --startup-project src/WebApi
dotnet ef migrations script --idempotent --output migrations.sql
```

## References

- [EF Core Patterns](references/ef-patterns.md) — Fluent API, value conversions, inheritance mapping, global query filters, JSON columns, temporal tables.
- [EF Core Optimization](references/ef-optimization.md) — Loading strategies, compiled queries, bulk operations, change tracking, interceptors.
