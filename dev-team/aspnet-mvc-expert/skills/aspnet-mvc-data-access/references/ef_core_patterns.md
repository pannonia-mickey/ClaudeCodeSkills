# Entity Framework Core Advanced Patterns

Advanced EF Core patterns for ASP.NET Core MVC applications.

## Interceptors

### Auditable Entity Interceptor

Automatically set audit fields on save:

```csharp
public class AuditableEntityInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;
    private readonly IDateTime _dateTime;

    public AuditableEntityInterceptor(
        ICurrentUserService currentUser, IDateTime dateTime)
    {
        _currentUser = currentUser;
        _dateTime = dateTime;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var context = eventData.Context;
        if (context is null) return base.SavingChangesAsync(eventData, result, ct);

        foreach (var entry in context.ChangeTracker.Entries<AuditableEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedBy = _currentUser.UserId;
                    entry.Entity.CreatedAt = _dateTime.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.LastModifiedBy = _currentUser.UserId;
                    entry.Entity.LastModifiedAt = _dateTime.UtcNow;
                    break;
            }
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Register
services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString)
        .AddInterceptors(sp.GetRequiredService<AuditableEntityInterceptor>());
});
```

### Domain Event Dispatcher Interceptor

Dispatch domain events after SaveChanges succeeds:

```csharp
public class DomainEventDispatcherInterceptor : SaveChangesInterceptor
{
    private readonly IMediator _mediator;

    public DomainEventDispatcherInterceptor(IMediator mediator)
        => _mediator = mediator;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        var context = eventData.Context;
        if (context is null) return result;

        var entities = context.ChangeTracker.Entries<Entity>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = entities
            .SelectMany(e => e.DomainEvents)
            .ToList();

        entities.ForEach(e => e.ClearDomainEvents());

        foreach (var domainEvent in domainEvents)
            await _mediator.Publish(domainEvent, ct);

        return result;
    }
}
```

## Value Converters

Convert between domain types and database types:

```csharp
// Money value object
public record Money(decimal Amount, string Currency);

// Value converter
public class MoneyConverter : ValueConverter<Money, string>
{
    public MoneyConverter() : base(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
        v => JsonSerializer.Deserialize<Money>(v, (JsonSerializerOptions?)null)!)
    { }
}

// Strongly-typed ID converter
public readonly record struct ProductId(int Value);

public class ProductIdConverter : ValueConverter<ProductId, int>
{
    public ProductIdConverter() : base(
        v => v.Value,
        v => new ProductId(v))
    { }
}

// Apply in configuration
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.Property(p => p.Id)
            .HasConversion<ProductIdConverter>();

        builder.Property(p => p.Price)
            .HasConversion<MoneyConverter>()
            .HasMaxLength(100);
    }
}

// Global convention for all DateTime properties
protected override void ConfigureConventions(ModelConfigurationBuilder builder)
{
    builder.Properties<DateTime>()
        .HaveConversion<DateTimeToUtcConverter>();

    builder.Properties<string>()
        .HaveMaxLength(256);
}
```

## Owned Types

Map value objects as owned types (stored in same table):

```csharp
public class Address
{
    public string Street { get; private set; } = string.Empty;
    public string City { get; private set; } = string.Empty;
    public string State { get; private set; } = string.Empty;
    public string ZipCode { get; private set; } = string.Empty;
    public string Country { get; private set; } = string.Empty;
}

// Configuration
builder.OwnsOne(o => o.ShippingAddress, sa =>
{
    sa.Property(a => a.Street).HasMaxLength(200).HasColumnName("ShippingStreet");
    sa.Property(a => a.City).HasMaxLength(100).HasColumnName("ShippingCity");
    sa.Property(a => a.State).HasMaxLength(50).HasColumnName("ShippingState");
    sa.Property(a => a.ZipCode).HasMaxLength(20).HasColumnName("ShippingZip");
    sa.Property(a => a.Country).HasMaxLength(100).HasColumnName("ShippingCountry");
});
```

## Soft Delete with Global Query Filters

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}

// Apply global filter for all soft-deletable entities
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
        {
            var parameter = Expression.Parameter(entityType.ClrType, "e");
            var property = Expression.Property(parameter, nameof(ISoftDeletable.IsDeleted));
            var condition = Expression.Equal(property, Expression.Constant(false));
            var lambda = Expression.Lambda(condition, parameter);
            modelBuilder.Entity(entityType.ClrType).HasQueryFilter(lambda);
        }
    }
}

// Override SaveChanges to intercept deletes
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    foreach (var entry in ChangeTracker.Entries<ISoftDeletable>())
    {
        if (entry.State == EntityState.Deleted)
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTime.UtcNow;
            entry.Entity.DeletedBy = _currentUser.UserId;
        }
    }
    return await base.SaveChangesAsync(ct);
}

// Query including deleted items when needed
var allProducts = await _db.Products.IgnoreQueryFilters().ToListAsync();
```

## Concurrency Handling

### Optimistic Concurrency with Row Version

```csharp
public class Product : Entity
{
    // ...
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// Configuration
builder.Property(p => p.RowVersion)
    .IsRowVersion();

// Handle concurrency conflicts
try
{
    await _db.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var currentValues = entry.CurrentValues;
    var databaseValues = await entry.GetDatabaseValuesAsync(ct);

    if (databaseValues is null)
        throw new NotFoundException("Entity was deleted");

    // Option 1: Database wins — reload and retry
    entry.OriginalValues.SetValues(databaseValues);

    // Option 2: Client wins — force overwrite
    // entry.OriginalValues.SetValues(databaseValues);
    // await _db.SaveChangesAsync(ct);

    // Option 3: Merge — compare fields and merge
    // Custom merge logic here
}
```

## Temporal Tables (SQL Server)

```csharp
// Configuration
builder.Entity<Product>()
    .ToTable("Products", t => t.IsTemporal(
        temporal =>
        {
            temporal.HasPeriodStart("ValidFrom");
            temporal.HasPeriodEnd("ValidTo");
            temporal.UseHistoryTable("ProductsHistory");
        }));

// Query historical data
var productAsOf = await _db.Products
    .TemporalAsOf(DateTime.UtcNow.AddDays(-30))
    .FirstOrDefaultAsync(p => p.Id == productId);

var productHistory = await _db.Products
    .TemporalAll()
    .Where(p => p.Id == productId)
    .OrderBy(p => EF.Property<DateTime>(p, "ValidFrom"))
    .ToListAsync();
```

## Connection Resilience

```csharp
services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: new[] { 4060, 40197, 40501, 40613, 49918, 49919, 49920 });
    });
});

// For explicit transactions with retry
var strategy = _db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var transaction = await _db.Database.BeginTransactionAsync();
    try
    {
        // Multiple operations
        _db.Products.Add(product);
        await _db.SaveChangesAsync();

        _db.AuditLogs.Add(new AuditLog("Product created"));
        await _db.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
});
```

## Bulk Operations

For high-volume operations, consider EF Core 7+ `ExecuteUpdate`/`ExecuteDelete`:

```csharp
// Bulk update without loading entities
await _db.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.CategoryId, newCategoryId)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

// Bulk delete without loading entities
await _db.Products
    .Where(p => p.IsDeleted && p.DeletedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();
```

## Query Performance Tips

| Technique | When to Use |
|-----------|-------------|
| `AsNoTracking()` | Read-only queries (most GET operations) |
| `Select()` projection | When you need only specific columns |
| `AsSplitQuery()` | Multiple Include chains causing cartesian explosion |
| Compiled queries | Hot paths executed frequently |
| `TagWith("query-name")` | To identify queries in SQL Profiler/logs |
| `ToQueryString()` | Debug generated SQL during development |
