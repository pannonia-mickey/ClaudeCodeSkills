# EF Core Patterns

## Fluent API Configuration

Always use `IEntityTypeConfiguration<T>` classes in dedicated files. Never use data annotations for anything beyond simple validation.

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Status).HasConversion<string>().HasMaxLength(20);
        builder.Property(o => o.Total).HasPrecision(18, 2);
        builder.HasMany(o => o.Items).WithOne(i => i.Order).HasForeignKey(i => i.OrderId).OnDelete(DeleteBehavior.Cascade);
        builder.HasQueryFilter(o => !o.IsDeleted);
    }
}
```

## Value Conversions

```csharp
// Enum to string
builder.Property(o => o.Status).HasConversion<string>().HasMaxLength(20);

// Strongly-typed ID
public readonly record struct OrderId(Guid Value) { public static OrderId New() => new(Guid.NewGuid()); }
builder.Property(o => o.Id).HasConversion(id => id.Value, value => new OrderId(value));

// DateOnly
builder.Property(e => e.BirthDate).HasConversion<DateOnlyConverter>();
```

## Inheritance Mapping

| Strategy | Tables | Queries | Best For |
|----------|--------|---------|----------|
| TPH | 1 table, discriminator | Fast | Few derived types, similar columns |
| TPT | 1 per type, JOINs | Slower | Normalized schema |
| TPC | 1 per concrete, no JOINs | Fast reads | No shared queries needed |

```csharp
// TPH (default)
modelBuilder.Entity<Payment>().HasDiscriminator<string>("PaymentType")
    .HasValue<CreditCardPayment>("CreditCard").HasValue<BankTransferPayment>("BankTransfer");

// TPC (EF Core 7+)
modelBuilder.Entity<Payment>().UseTpcMappingStrategy();
```

## Global Query Filters

```csharp
builder.HasQueryFilter(o => !o.IsDeleted);
// Bypass when needed:
var allOrders = await db.Orders.IgnoreQueryFilters().ToListAsync();
```

## JSON Columns (EF Core 7+)

```csharp
builder.OwnsOne(p => p.Metadata, m => { m.ToJson(); m.OwnsOne(md => md.Dimensions); });
```

## Temporal Tables (EF Core 6+)

```csharp
builder.ToTable("Products", b => b.IsTemporal());
// Query history:
var history = await db.Products.TemporalAll().Where(p => p.Id == id).OrderBy(p => EF.Property<DateTime>(p, "PeriodStart")).ToListAsync();
```

## Concurrency Tokens

```csharp
builder.Property(o => o.RowVersion).IsRowVersion(); // SQL Server timestamp
// Or:
builder.Property(o => o.Version).IsConcurrencyToken();
```

## Shadow Properties

```csharp
builder.Property<DateTime>("LastModified");
// Set in SaveChangesAsync:
entry.Property("LastModified").CurrentValue = DateTime.UtcNow;
```
