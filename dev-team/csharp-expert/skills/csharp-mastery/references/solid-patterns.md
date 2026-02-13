# SOLID Patterns in C#

## Single Responsibility Principle (SRP)

Each class should have one reason to change. In ASP.NET Core applications, enforce SRP by separating concerns into distinct layers: controllers handle HTTP, services handle business logic, and repositories handle data access.

### Controller / Service / Repository Separation

```csharp
// Controller — HTTP concerns only: routing, model binding, status codes
[ApiController]
[Route("api/[controller]")]
public class OrdersController(IOrderService orderService) : ControllerBase
{
    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var result = await orderService.CreateOrderAsync(request, ct);
        return result.Match<IActionResult>(
            order => CreatedAtAction(nameof(GetById), new { id = order.Id }, order),
            error => BadRequest(error));
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var order = await orderService.GetByIdAsync(id, ct);
        return order is not null ? Ok(order) : NotFound();
    }
}

// Service — business logic, validation, orchestration
public class OrderService(
    IOrderRepository orderRepository,
    IInventoryService inventoryService,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<Result<OrderDto>> CreateOrderAsync(
        CreateOrderRequest request,
        CancellationToken ct)
    {
        // Business rule: verify stock before creating order
        foreach (var item in request.Items)
        {
            var available = await inventoryService.CheckStockAsync(item.ProductId, ct);
            if (available < item.Quantity)
                return Result<OrderDto>.Failure($"Insufficient stock for product {item.ProductId}");
        }

        var order = Order.Create(request.CustomerId, request.Items);
        await orderRepository.AddAsync(order, ct);
        await orderRepository.SaveChangesAsync(ct);

        logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            order.Id, order.CustomerId);

        return Result<OrderDto>.Success(order.ToDto());
    }

    public async Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        var order = await orderRepository.GetByIdAsync(id, ct);
        return order?.ToDto();
    }
}

// Repository — data access only
public class OrderRepository(AppDbContext dbContext) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct) =>
        await dbContext.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await dbContext.Orders.AddAsync(order, ct);

    public async Task SaveChangesAsync(CancellationToken ct) =>
        await dbContext.SaveChangesAsync(ct);
}
```

### Focused Configuration Classes

Split configuration into single-purpose classes rather than piling everything into `Program.cs`.

```csharp
public static class AuthenticationConfig
{
    public static IServiceCollection AddAppAuthentication(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.Authority = configuration["Auth:Authority"];
                options.Audience = configuration["Auth:Audience"];
            });

        services.AddAuthorizationBuilder()
            .AddPolicy("Admin", p => p.RequireRole("admin"))
            .AddPolicy("Manager", p => p.RequireRole("admin", "manager"));

        return services;
    }
}
```

## Open/Closed Principle (OCP)

Classes should be open for extension but closed for modification. Use interfaces, abstract base classes, and the decorator pattern to add behavior without changing existing code.

### Interface-Based Extensibility

```csharp
// Define a notification interface
public interface INotificationSender
{
    Task SendAsync(Notification notification, CancellationToken ct = default);
    bool CanHandle(NotificationType type);
}

// Each channel is a separate implementation — adding SMS requires no changes to existing code
public class EmailNotificationSender(IEmailClient emailClient) : INotificationSender
{
    public bool CanHandle(NotificationType type) => type == NotificationType.Email;

    public async Task SendAsync(Notification notification, CancellationToken ct)
    {
        await emailClient.SendAsync(notification.Recipient, notification.Subject, notification.Body, ct);
    }
}

public class SlackNotificationSender(ISlackClient slackClient) : INotificationSender
{
    public bool CanHandle(NotificationType type) => type == NotificationType.Slack;

    public async Task SendAsync(Notification notification, CancellationToken ct)
    {
        await slackClient.PostMessageAsync(notification.Recipient, notification.Body, ct);
    }
}

// Dispatcher routes to correct sender — closed for modification
public class NotificationDispatcher(IEnumerable<INotificationSender> senders)
{
    public async Task DispatchAsync(Notification notification, CancellationToken ct)
    {
        var sender = senders.FirstOrDefault(s => s.CanHandle(notification.Type))
            ?? throw new NotSupportedException($"No sender for {notification.Type}");
        await sender.SendAsync(notification, ct);
    }
}
```

### Decorator Pattern

```csharp
// Base interface
public interface IOrderService
{
    Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct);
}

// Core implementation
public class OrderService(IOrderRepository repo) : IOrderService
{
    public async Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        var order = await repo.GetByIdAsync(id, ct);
        return order?.ToDto();
    }
}

// Caching decorator — extends behavior without modifying OrderService
public class CachedOrderService(IOrderService inner, IMemoryCache cache) : IOrderService
{
    public async Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        var key = $"order:{id}";
        if (cache.TryGetValue(key, out OrderDto? cached))
            return cached;

        var result = await inner.GetByIdAsync(id, ct);
        if (result is not null)
            cache.Set(key, result, TimeSpan.FromMinutes(5));

        return result;
    }
}

// Registration with decoration
services.AddScoped<OrderService>();
services.AddScoped<IOrderService>(sp =>
    new CachedOrderService(
        sp.GetRequiredService<OrderService>(),
        sp.GetRequiredService<IMemoryCache>()));
```

### Middleware for Cross-Cutting Concerns

```csharp
public class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            await next(context);
        }
        finally
        {
            stopwatch.Stop();
            logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}
```

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering correctness. In C#, this applies to interface implementations, class inheritance, and generic variance.

### Interface Implementation Contracts

```csharp
// Base interface with clear contract
public interface IPaymentProcessor
{
    /// <summary>
    /// Processes a payment. Returns a receipt on success, throws PaymentException on failure.
    /// The amount must be positive.
    /// </summary>
    Task<PaymentReceipt> ProcessAsync(PaymentRequest request, CancellationToken ct);
}

// Both implementations honor the same contract
public class StripePaymentProcessor(IStripeClient client) : IPaymentProcessor
{
    public async Task<PaymentReceipt> ProcessAsync(PaymentRequest request, CancellationToken ct)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(request.Amount);
        var charge = await client.ChargeAsync(request.Amount, request.Currency, ct);
        return new PaymentReceipt(charge.Id, charge.Amount, DateTime.UtcNow);
    }
}

public class PayPalPaymentProcessor(IPayPalClient client) : IPaymentProcessor
{
    public async Task<PaymentReceipt> ProcessAsync(PaymentRequest request, CancellationToken ct)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(request.Amount);
        var payment = await client.CreatePaymentAsync(request.Amount, request.Currency, ct);
        return new PaymentReceipt(payment.Id, payment.Amount, DateTime.UtcNow);
    }
}
```

### Generic Constraints and Covariance

```csharp
// Covariant interface — IReadRepository<Dog> can be used where IReadRepository<Animal> is expected
public interface IReadRepository<out T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct);
}

// Contravariant interface — IValidator<Animal> can validate a Dog
public interface IValidator<in T>
{
    ValidationResult Validate(T instance);
}

// Constrained generic method
public static T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) >= 0 ? a : b;
```

## Interface Segregation Principle (ISP)

Keep interfaces small and role-specific. Clients should not depend on methods they do not use.

### Role Interfaces

```csharp
// Segregated read and write interfaces
public interface IReadRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> ListAsync(CancellationToken ct = default);
    Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken ct = default);
}

public interface IWriteRepository<T> where T : class
{
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
    Task SaveChangesAsync(CancellationToken ct = default);
}

// Read-only services only depend on IReadRepository
public class ProductQueryService(IReadRepository<Product> repository)
{
    public Task<List<Product>> GetAllAsync(CancellationToken ct) =>
        repository.ListAsync(ct);
}

// Write services depend on IWriteRepository
public class ProductCommandService(
    IWriteRepository<Product> repository,
    IValidator<Product> validator)
{
    public async Task<Result<Product>> CreateAsync(Product product, CancellationToken ct)
    {
        var validation = validator.Validate(product);
        if (!validation.IsValid)
            return Result<Product>.Failure(validation.ToString());

        await repository.AddAsync(product, ct);
        await repository.SaveChangesAsync(ct);
        return Result<Product>.Success(product);
    }
}
```

### Focused Capability Interfaces

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
    DateTime? ModifiedAt { get; set; }
    string? ModifiedBy { get; set; }
}

public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

// Entity implements only what it needs
public class Order : IAuditable, ISoftDeletable
{
    public int Id { get; set; }
    public decimal Total { get; set; }

    // IAuditable
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; } = default!;
    public DateTime? ModifiedAt { get; set; }
    public string? ModifiedBy { get; set; }

    // ISoftDeletable
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}
```

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions. In .NET, the built-in IServiceCollection container makes this straightforward.

### Constructor Injection

```csharp
// Abstraction
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct);
}

// Low-level implementation
public class SmtpEmailService(IOptions<SmtpSettings> options) : IEmailService
{
    private readonly SmtpSettings _settings = options.Value;

    public async Task SendAsync(string to, string subject, string body, CancellationToken ct)
    {
        using var client = new SmtpClient(_settings.Host, _settings.Port);
        var message = new MailMessage(_settings.From, to, subject, body);
        await client.SendMailAsync(message, ct);
    }
}

// Registration — high-level code never references SmtpEmailService directly
services.Configure<SmtpSettings>(configuration.GetSection("Smtp"));
services.AddScoped<IEmailService, SmtpEmailService>();
```

### Options Pattern for Configuration

```csharp
public class JwtSettings
{
    public const string SectionName = "Jwt";

    public required string Secret { get; init; }
    public required string Issuer { get; init; }
    public required string Audience { get; init; }
    public int ExpirationMinutes { get; init; } = 60;
}

// Registration with validation
services.AddOptions<JwtSettings>()
    .BindConfiguration(JwtSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Consumption via IOptions<T>
public class TokenService(IOptions<JwtSettings> options)
{
    private readonly JwtSettings _settings = options.Value;

    public string GenerateToken(User user)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        // ... token generation
    }
}
```

### Clean Registration Extensions

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IInventoryService, InventoryService>();
        return services;
    }

    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IEmailService, SmtpEmailService>();

        return services;
    }
}

// Clean Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddApplicationServices()
    .AddInfrastructureServices(builder.Configuration)
    .AddAppAuthentication(builder.Configuration);
```
