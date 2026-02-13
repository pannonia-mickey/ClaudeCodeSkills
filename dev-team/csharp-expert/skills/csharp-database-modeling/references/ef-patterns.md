# EF Core Patterns

## Fluent API Configuration Patterns

### Convention-Based Configuration Class Discovery

Apply all configurations from an assembly automatically rather than registering each one manually.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // Apply global conventions
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        // Configure all string properties to have a default max length
        foreach (var property in entityType.GetProperties()
            .Where(p => p.ClrType == typeof(string)))
        {
            if (property.GetMaxLength() is null)
                property.SetMaxLength(256);
        }

        // Configure all decimal properties to have precision
        foreach (var property in entityType.GetProperties()
            .Where(p => p.ClrType == typeof(decimal) || p.ClrType == typeof(decimal?)))
        {
            if (property.GetPrecision() is null)
                property.SetPrecision(18);
            if (property.GetScale() is null)
                property.SetScale(2);
        }
    }
}
```

### Strongly-Typed ID Configuration

```csharp
public readonly record struct ProductId(Guid Value)
{
    public static ProductId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}

public class ProductIdConverter : ValueConverter<ProductId, Guid>
{
    public ProductIdConverter()
        : base(id => id.Value, value => new ProductId(value)) { }
}

// Register globally for all ProductId properties
modelBuilder.Entity<Product>()
    .Property(p => p.Id)
    .HasConversion<ProductIdConverter>();

// Or register a convention for all strongly-typed IDs
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<ProductId>()
        .HaveConversion<ProductIdConverter>();

    configurationBuilder.Properties<OrderId>()
        .HaveConversion<OrderIdConverter>();
}
```

## Value Conversions

### Enum Conversions

```csharp
// Store enum as string in database
builder.Property(o => o.Status)
    .HasConversion<string>()
    .HasMaxLength(30);

// Custom enum conversion with mapping
builder.Property(o => o.Priority)
    .HasConversion(
        v => v.ToString().ToLowerInvariant(),
        v => Enum.Parse<Priority>(v, ignoreCase: true))
    .HasMaxLength(20);

// Flags enum stored as comma-separated string
builder.Property(u => u.Permissions)
    .HasConversion(
        v => string.Join(",", Enum.GetValues<Permission>().Where(p => v.HasFlag(p))),
        v => v.Split(",", StringSplitOptions.RemoveEmptyEntries)
            .Select(s => Enum.Parse<Permission>(s))
            .Aggregate((a, b) => a | b))
    .HasMaxLength(500);
```

### Complex Value Object Conversions

```csharp
// Money value object
public record Money(decimal Amount, string Currency);

public class MoneyConverter : ValueConverter<Money, string>
{
    public MoneyConverter()
        : base(
            m => $"{m.Amount}:{m.Currency}",
            s => ParseMoney(s)) { }

    private static Money ParseMoney(string s)
    {
        var parts = s.Split(':');
        return new Money(decimal.Parse(parts[0]), parts[1]);
    }
}

// Email value object with validation
public record Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.Trim().ToLowerInvariant();
    }

    public static implicit operator string(Email email) => email.Value;
}

builder.Property(u => u.Email)
    .HasConversion(e => e.Value, s => new Email(s))
    .HasMaxLength(256);
```

## Shadow Properties

Shadow properties exist in the EF Core model but not on the entity class. Use them for metadata that does not belong in the domain model.

```csharp
// Define shadow properties in configuration
builder.Property<DateTime>("CreatedAt")
    .HasDefaultValueSql("GETUTCDATE()");

builder.Property<DateTime?>("ModifiedAt");

builder.Property<string>("CreatedBy")
    .HasMaxLength(100);

// Set shadow properties in SaveChanges override
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    foreach (var entry in ChangeTracker.Entries())
    {
        if (entry.State == EntityState.Added)
        {
            entry.Property("CreatedAt").CurrentValue = DateTime.UtcNow;
            entry.Property("CreatedBy").CurrentValue = _currentUser.Id;
        }

        if (entry.State == EntityState.Modified)
        {
            entry.Property("ModifiedAt").CurrentValue = DateTime.UtcNow;
        }
    }

    return await base.SaveChangesAsync(ct);
}

// Query using shadow properties
var recentProducts = await dbContext.Products
    .OrderByDescending(p => EF.Property<DateTime>(p, "CreatedAt"))
    .Take(10)
    .ToListAsync(ct);
```

## Inheritance Mapping Strategies

### Table Per Hierarchy (TPH) — Default

All types in the hierarchy share one table with a discriminator column. Best for simple hierarchies with similar columns.

```csharp
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime ProcessedAt { get; set; }
}

public class CreditCardPayment : Payment
{
    public required string Last4Digits { get; set; }
    public required string CardBrand { get; set; }
}

public class BankTransferPayment : Payment
{
    public required string BankName { get; set; }
    public required string AccountLast4 { get; set; }
}

// Configuration (TPH is the default)
builder.HasDiscriminator<string>("PaymentType")
    .HasValue<CreditCardPayment>("CreditCard")
    .HasValue<BankTransferPayment>("BankTransfer");
```

### Table Per Type (TPT)

Each type gets its own table. Better normalization but requires joins.

```csharp
modelBuilder.Entity<Payment>().ToTable("Payments");
modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
```

### Table Per Concrete Type (TPC) — EF Core 7+

Each concrete type gets its own table with all columns. No joins needed, but no shared table for the base type.

```csharp
modelBuilder.Entity<Payment>().UseTpcMappingStrategy();

modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");

// Configure sequence for ID generation across tables
modelBuilder.HasSequence<int>("PaymentIds");
modelBuilder.Entity<Payment>()
    .Property(p => p.Id)
    .UseSequence("PaymentIds");
```

## Global Query Filters

Apply filters automatically to all queries against an entity. Commonly used for soft delete and multi-tenancy.

```csharp
// Soft delete filter
builder.HasQueryFilter(e => !e.IsDeleted);

// Multi-tenant filter
public class AppDbContext(
    DbContextOptions<AppDbContext> options,
    ITenantProvider tenantProvider) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == tenantProvider.TenantId);

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == tenantProvider.TenantId);
    }
}

// Bypass filters when needed (e.g., admin queries)
var allOrders = await dbContext.Orders
    .IgnoreQueryFilters()
    .ToListAsync(ct);
```

## JSON Columns (EF Core 7+)

Store complex, semi-structured data as JSON inside a relational column. Query into the JSON structure with LINQ.

```csharp
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public decimal Price { get; set; }
    public ProductDetails Details { get; set; } = new();
}

public class ProductDetails
{
    public string? Description { get; set; }
    public List<string> Tags { get; set; } = [];
    public List<ProductImage> Images { get; set; } = [];
    public SpecificationSheet? Specifications { get; set; }
}

public class ProductImage
{
    public required string Url { get; set; }
    public string? AltText { get; set; }
    public int SortOrder { get; set; }
}

public class SpecificationSheet
{
    public double? Weight { get; set; }
    public string? WeightUnit { get; set; }
    public Dictionary<string, string> CustomFields { get; set; } = new();
}

// Configuration
builder.OwnsOne(p => p.Details, d =>
{
    d.ToJson();
    d.OwnsMany(dd => dd.Images);
    d.OwnsOne(dd => dd.Specifications);
});

// Query into JSON
var taggedProducts = await dbContext.Products
    .Where(p => p.Details.Tags.Contains("electronics"))
    .OrderBy(p => p.Name)
    .ToListAsync(ct);

var heavyProducts = await dbContext.Products
    .Where(p => p.Details.Specifications != null
             && p.Details.Specifications.Weight > 10)
    .ToListAsync(ct);
```

## Sequence and Concurrency Configuration

```csharp
// Optimistic concurrency with row version
builder.Property(o => o.RowVersion)
    .IsRowVersion();

// Or use a concurrency token on a specific property
builder.Property(o => o.Status)
    .IsConcurrencyToken();

// Handle concurrency conflicts
try
{
    await dbContext.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync(ct);

    if (databaseValues is null)
        throw new NotFoundException("Entity was deleted");

    // Client wins: overwrite database values
    entry.OriginalValues.SetValues(databaseValues);
    await dbContext.SaveChangesAsync(ct);
}
```

## Seed Data

```csharp
// Static seed data in configuration
builder.HasData(
    new Product { Id = 1, Name = "Widget", Price = 9.99m, CategoryId = 1 },
    new Product { Id = 2, Name = "Gadget", Price = 19.99m, CategoryId = 1 },
    new Product { Id = 3, Name = "Doohickey", Price = 4.99m, CategoryId = 2 }
);

// Seed owned types separately
builder.OwnsOne(c => c.Address).HasData(new
{
    CustomerId = 1,
    Street = "123 Main St",
    City = "Springfield",
    State = "IL",
    ZipCode = "62701",
    Country = "US"
});
```
