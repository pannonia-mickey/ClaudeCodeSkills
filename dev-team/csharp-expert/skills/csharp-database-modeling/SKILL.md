---
name: C# Database Modeling
description: This skill should be used when the user asks about "Entity Framework", "EF Core", "DbContext", "EF migration", or "entity configuration". It covers DbContext configuration, entity modeling with Fluent API, database migrations, relationship mapping, value objects, owned types, query optimization, and EF Core 8+ features.
---

# Entity Framework Core Database Modeling

## DbContext Configuration

Configure the DbContext as the central unit of work. Keep it focused — one DbContext per bounded context in larger applications.

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all IEntityTypeConfiguration implementations from this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Automatically set audit fields before saving
        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.ModifiedAt = DateTime.UtcNow;
                    break;
            }
        }

        return await base.SaveChangesAsync(ct);
    }
}
```

Register the DbContext with appropriate options.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sqlOptions =>
        {
            sqlOptions.MigrationsAssembly("MyApp.Infrastructure");
            sqlOptions.EnableRetryOnFailure(maxRetryCount: 3);
            sqlOptions.CommandTimeout(30);
        });

    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});
```

## Entity Configuration with Fluent API

Separate entity configurations into dedicated `IEntityTypeConfiguration<T>` classes. Never use data annotations for anything beyond simple validation — Fluent API keeps domain entities clean.

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(o => o.Id);

        builder.Property(o => o.Id)
            .ValueGeneratedOnAdd();

        builder.Property(o => o.OrderNumber)
            .HasMaxLength(50)
            .IsRequired();

        builder.HasIndex(o => o.OrderNumber)
            .IsUnique();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20)
            .IsRequired();

        builder.Property(o => o.Total)
            .HasPrecision(18, 2);

        // Relationship: Order has many OrderItems
        builder.HasMany(o => o.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // Relationship: Order belongs to Customer
        builder.HasOne(o => o.Customer)
            .WithMany(c => c.Orders)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);

        // Global query filter for soft delete
        builder.HasQueryFilter(o => !o.IsDeleted);
    }
}
```

## Relationships

### One-to-Many

```csharp
// Entity
public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public List<Order> Orders { get; set; } = [];
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
}

// Configuration
builder.HasMany(c => c.Orders)
    .WithOne(o => o.Customer)
    .HasForeignKey(o => o.CustomerId)
    .OnDelete(DeleteBehavior.Cascade);
```

### Many-to-Many

```csharp
// Skip navigation (EF Core 5+)
public class Student
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public List<Course> Courses { get; set; } = [];
}

public class Course
{
    public int Id { get; set; }
    public required string Title { get; set; }
    public List<Student> Students { get; set; } = [];
}

// Configuration with join table customization
builder.HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Dictionary<string, object>>(
        "StudentCourse",
        j => j.HasOne<Course>().WithMany().HasForeignKey("CourseId"),
        j => j.HasOne<Student>().WithMany().HasForeignKey("StudentId"),
        j =>
        {
            j.ToTable("StudentCourses");
            j.HasKey("StudentId", "CourseId");
        });
```

### One-to-One with Owned Types

```csharp
// Value object as owned type
public class Address
{
    public required string Street { get; init; }
    public required string City { get; init; }
    public required string State { get; init; }
    public required string ZipCode { get; init; }
    public required string Country { get; init; }
}

public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public Address ShippingAddress { get; set; } = null!;
    public Address? BillingAddress { get; set; }
}

// Configuration
builder.OwnsOne(c => c.ShippingAddress, a =>
{
    a.Property(x => x.Street).HasMaxLength(200).IsRequired();
    a.Property(x => x.City).HasMaxLength(100).IsRequired();
    a.Property(x => x.State).HasMaxLength(50).IsRequired();
    a.Property(x => x.ZipCode).HasMaxLength(20).IsRequired();
    a.Property(x => x.Country).HasMaxLength(100).IsRequired();
});

builder.OwnsOne(c => c.BillingAddress, a =>
{
    a.Property(x => x.Street).HasMaxLength(200);
    a.Property(x => x.City).HasMaxLength(100);
    a.Property(x => x.State).HasMaxLength(50);
    a.Property(x => x.ZipCode).HasMaxLength(20);
    a.Property(x => x.Country).HasMaxLength(100);
});
```

## Value Conversions

Map domain types to database column types with value converters.

```csharp
// Enum to string conversion
builder.Property(o => o.Status)
    .HasConversion<string>()
    .HasMaxLength(20);

// Custom value converter for strongly-typed IDs
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
}

builder.Property(o => o.Id)
    .HasConversion(
        id => id.Value,
        value => new OrderId(value));

// DateOnly / TimeOnly
builder.Property(e => e.BirthDate)
    .HasConversion<DateOnlyConverter>();

// Money value object stored as decimal column
builder.Property(o => o.Price)
    .HasConversion(
        money => money.Amount,
        amount => new Money(amount, "USD"))
    .HasPrecision(18, 2);
```

## JSON Columns (EF Core 7+)

Store complex objects as JSON inside a relational column.

```csharp
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public ProductMetadata Metadata { get; set; } = new();
}

public class ProductMetadata
{
    public List<string> Tags { get; set; } = [];
    public Dictionary<string, string> Attributes { get; set; } = new();
    public DimensionsInfo? Dimensions { get; set; }
}

public class DimensionsInfo
{
    public double Width { get; set; }
    public double Height { get; set; }
    public double Depth { get; set; }
    public string Unit { get; set; } = "cm";
}

// Configuration
builder.OwnsOne(p => p.Metadata, m =>
{
    m.ToJson();
    m.OwnsOne(md => md.Dimensions);
});
```

## Migrations

### Creating and Applying Migrations

```bash
# Create a migration
dotnet ef migrations add AddOrderTable --project src/Infrastructure --startup-project src/WebApi

# Apply migrations
dotnet ef database update --project src/Infrastructure --startup-project src/WebApi

# Generate SQL script for production deployment
dotnet ef migrations script --idempotent --output migrations.sql
```

### Migration Best Practices

Apply data migrations carefully. Use raw SQL for large data operations to avoid loading entities into memory.

```csharp
public partial class SplitFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "FirstName",
            table: "Customers",
            type: "nvarchar(100)",
            nullable: true);

        migrationBuilder.AddColumn<string>(
            name: "LastName",
            table: "Customers",
            type: "nvarchar(100)",
            nullable: true);

        // Migrate data with raw SQL
        migrationBuilder.Sql("""
            UPDATE Customers
            SET FirstName = LEFT(FullName, CHARINDEX(' ', FullName + ' ') - 1),
                LastName = LTRIM(SUBSTRING(FullName, CHARINDEX(' ', FullName + ' '), LEN(FullName)))
            WHERE FullName IS NOT NULL
            """);

        migrationBuilder.AlterColumn<string>(
            name: "FirstName",
            table: "Customers",
            type: "nvarchar(100)",
            nullable: false,
            defaultValue: "");

        migrationBuilder.DropColumn(name: "FullName", table: "Customers");
    }
}
```

## Query Optimization

### Projection

Always project to DTOs when the full entity is not needed. This generates efficient SQL that only selects required columns.

```csharp
// Good: project to DTO
var orders = await dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderListDto
    {
        Id = o.Id,
        OrderNumber = o.OrderNumber,
        Total = o.Total,
        ItemCount = o.Items.Count,
        CustomerName = o.Customer.Name
    })
    .ToListAsync(ct);

// Avoid: loading full entity graph when only a few fields are needed
var orders = await dbContext.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct); // Loads everything into memory
```

### Split Queries

Use split queries when loading entities with multiple collection navigations to avoid cartesian explosion.

```csharp
var orders = await dbContext.Orders
    .Include(o => o.Items)
    .Include(o => o.Shipments)
    .AsSplitQuery() // Generates separate SQL queries per Include
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync(ct);
```

### No-Tracking Queries

Use no-tracking for read-only operations to avoid change tracker overhead.

```csharp
var products = await dbContext.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToListAsync(ct);
```

## Interceptors

Use interceptors for cross-cutting concerns like soft delete and audit logging.

```csharp
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
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

// Register interceptor
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString)
        .AddInterceptors(new SoftDeleteInterceptor());
});
```

## References

- [EF Core Patterns](references/ef-patterns.md) — Fluent API patterns, value conversions, shadow properties, inheritance mapping (TPH/TPT/TPC), global query filters, and JSON columns.
- [EF Core Optimization](references/ef-optimization.md) — Loading strategies, compiled queries, raw SQL, bulk operations, change tracking optimization, and interceptors.
