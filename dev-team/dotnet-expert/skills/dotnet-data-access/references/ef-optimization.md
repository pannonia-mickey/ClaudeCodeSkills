# EF Core Optimization

## Loading Strategies

### Eager Loading (Include)
```csharp
var orders = await db.Orders.Include(o => o.Items).ThenInclude(i => i.Product).ToListAsync(ct);
```

### Split Queries (Avoid Cartesian Explosion)
```csharp
var orders = await db.Orders.Include(o => o.Items).Include(o => o.Shipments).AsSplitQuery().ToListAsync(ct);
```

### Explicit Loading
```csharp
var order = await db.Orders.FindAsync(id);
await db.Entry(order!).Collection(o => o.Items).LoadAsync(ct);
```

### No-Tracking for Read-Only
```csharp
var products = await db.Products.AsNoTracking().Where(p => p.IsActive).ToListAsync(ct);
// Or set globally: db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

## Compiled Queries

```csharp
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => db.Products.FirstOrDefault(p => p.Id == id));

// Usage: var product = await GetProductById(db, 42);
```

## Bulk Operations (EF Core 7+)

```csharp
// ExecuteUpdate — update without loading
await db.Products.Where(p => p.Category == "Seasonal").ExecuteUpdateAsync(
    s => s.SetProperty(p => p.Price, p => p.Price * 0.9m), ct);

// ExecuteDelete — delete without loading
await db.Orders.Where(o => o.Status == OrderStatus.Cancelled && o.CreatedAt < cutoff).ExecuteDeleteAsync(ct);
```

## Connection Resiliency

```csharp
options.UseSqlServer(connectionString, o =>
{
    o.EnableRetryOnFailure(maxRetryCount: 3, maxRetryDelay: TimeSpan.FromSeconds(10), errorNumbersToAdd: null);
    o.CommandTimeout(30);
});
```

## Interceptors

```csharp
// SaveChanges interceptor for audit logging
public class AuditInterceptor(ICurrentUserService currentUser) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct)
    {
        foreach (var entry in eventData.Context!.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added) { entry.Entity.CreatedBy = currentUser.UserId; entry.Entity.CreatedAt = DateTime.UtcNow; }
            if (entry.State == EntityState.Modified) { entry.Entity.ModifiedBy = currentUser.UserId; entry.Entity.ModifiedAt = DateTime.UtcNow; }
        }
        return ValueTask.FromResult(result);
    }
}

// Register: options.AddInterceptors(new AuditInterceptor(currentUser));
```

## Projection (Always Prefer)

```csharp
// GOOD — only loads needed columns
var summaries = await db.Orders.Select(o => new OrderSummaryDto
{
    Id = o.Id, Total = o.Total, ItemCount = o.Items.Count, CustomerName = o.Customer.Name
}).ToListAsync(ct);

// BAD — loads entire entity graph
var orders = await db.Orders.Include(o => o.Items).Include(o => o.Customer).ToListAsync(ct);
```
