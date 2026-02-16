# SOLID Patterns in C#

## Single Responsibility Principle (SRP)

Each class should have one reason to change. Separate concerns into focused services.

```csharp
// BAD — Controller handles validation, business logic, and persistence
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request)
    {
        if (string.IsNullOrEmpty(request.ProductId)) return BadRequest();
        var product = await _db.Products.FindAsync(request.ProductId);
        if (product is null) return NotFound();
        var order = new Order { ProductId = product.Id, Quantity = request.Quantity, Total = product.Price * request.Quantity };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        await _emailService.SendConfirmation(order);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}

// GOOD — Thin controller delegates to focused services
public class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderCommand command, CancellationToken ct)
    {
        var result = await mediator.Send(command, ct);
        return result.Match(
            order => CreatedAtAction(nameof(GetById), new { id = order.Id }, order),
            error => BadRequest(error));
    }
}
```

## Open/Closed Principle (OCP)

Classes should be open for extension but closed for modification. Use strategy pattern or decorators.

```csharp
// Strategy pattern for discount calculation
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class VolumeDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.ItemCount > 10 ? 0.1m : 0m;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Customer.IsLoyal ? 0.05m : 0m;
}

public class DiscountService(IEnumerable<IDiscountStrategy> strategies)
{
    public decimal CalculateTotal(Order order)
    {
        var totalDiscount = strategies.Sum(s => s.Calculate(order));
        return order.Subtotal * (1 - Math.Min(totalDiscount, 0.5m));
    }
}

// Adding a new discount requires only a new class, no modification of existing code
services.AddTransient<IDiscountStrategy, VolumeDiscount>();
services.AddTransient<IDiscountStrategy, LoyaltyDiscount>();
services.AddTransient<IDiscountStrategy, SeasonalDiscount>(); // NEW — no changes elsewhere
```

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering program correctness.

```csharp
// BAD — Square violates LSP by constraining Width/Height coupling
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }
public class Square : Rectangle
{
    public override int Width { set { base.Width = value; base.Height = value; } }
    public override int Height { set { base.Width = value; base.Height = value; } }
}

// GOOD — Use abstractions that make sense for all implementations
public interface IShape { double Area { get; } }
public record Rectangle(double Width, double Height) : IShape { public double Area => Width * Height; }
public record Circle(double Radius) : IShape { public double Area => Math.PI * Radius * Radius; }
```

## Interface Segregation Principle (ISP)

Clients should not depend on interfaces they don't use. Create focused role interfaces.

```csharp
// BAD — one large interface forces implementors to stub unused methods
public interface IRepository<T> { Task<T?> GetByIdAsync(int id); Task<List<T>> GetAllAsync(); void Add(T entity); void Update(T entity); void Delete(T entity); }

// GOOD — segregated by capability
public interface IReadRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<T>> ListAsync(CancellationToken ct = default);
}

public interface IWriteRepository<T> where T : class
{
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

public interface IRepository<T> : IReadRepository<T>, IWriteRepository<T> where T : class;

// Read-only services only depend on IReadRepository<T>
public class ProductQueryService(IReadRepository<Product> repository) { }
```

## Dependency Inversion Principle (DIP)

High-level modules depend on abstractions, not concrete implementations.

```csharp
// Application layer defines the interface
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct);
}

// Infrastructure layer provides the implementation
public class SmtpEmailSender(IOptions<SmtpSettings> options) : IEmailSender
{
    public async Task SendAsync(string to, string subject, string body, CancellationToken ct)
    {
        using var client = new SmtpClient(options.Value.Host, options.Value.Port);
        await client.SendMailAsync(new MailMessage("noreply@app.com", to, subject, body), ct);
    }
}

// DI wiring in composition root — the only place that knows the concrete type
services.AddScoped<IEmailSender, SmtpEmailSender>();
services.Configure<SmtpSettings>(configuration.GetSection("Smtp"));
```
