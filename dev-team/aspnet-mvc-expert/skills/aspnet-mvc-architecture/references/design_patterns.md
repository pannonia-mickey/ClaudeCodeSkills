# Design Patterns for ASP.NET Core MVC

Detailed reference for design patterns commonly applied in ASP.NET Core MVC enterprise applications.

## Repository Pattern

### Why Use Repository

- Abstracts data access logic from business logic
- Enables unit testing with mock repositories
- Centralizes query logic — avoids scattered DbContext usage
- Makes it easy to swap data providers

### Implementation Guidelines

- Define repository interfaces in the **Domain** or **Application** layer
- Implement repositories in the **Infrastructure** layer
- Repositories should return domain entities, not DTOs
- Expose only the operations each aggregate needs (Interface Segregation)

```csharp
// Domain/Interfaces/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(int customerId, CancellationToken ct = default);
    void Add(Order order);
    void Remove(Order order);
}
```

### When NOT to Use Repository

- For simple CRUD with no business logic — use DbContext directly in handlers
- When vertical slice architecture keeps queries close to handlers
- If the project is small and unlikely to change data providers

## Unit of Work Pattern

Coordinates the work of multiple repositories within a single database transaction.

```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _db;
    private IProductRepository? _products;
    private IOrderRepository? _orders;

    public UnitOfWork(AppDbContext db) => _db = db;

    public IProductRepository Products =>
        _products ??= new ProductRepository(_db);

    public IOrderRepository Orders =>
        _orders ??= new OrderRepository(_db);

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _db.SaveChangesAsync(ct);

    public void Dispose() => _db.Dispose();
}
```

**Simpler approach**: Use `DbContext` as the Unit of Work directly (it already is one):
```csharp
services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
```

## Result Pattern

Avoid exceptions for expected failures. Return result objects instead:

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }
    public ResultErrorType? ErrorType { get; }

    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(string error, ResultErrorType errorType)
    {
        IsSuccess = false;
        Error = error;
        ErrorType = errorType;
    }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> NotFound(string error) => new(error, ResultErrorType.NotFound);
    public static Result<T> ValidationError(string error) => new(error, ResultErrorType.Validation);
    public static Result<T> Forbidden(string error) => new(error, ResultErrorType.Forbidden);
}

public enum ResultErrorType { NotFound, Validation, Forbidden, Conflict }

// Usage in handler
public async Task<Result<ProductDto>> Handle(GetProductQuery query, CancellationToken ct)
{
    var product = await _repository.GetByIdAsync(query.Id, ct);
    if (product is null)
        return Result<ProductDto>.NotFound($"Product {query.Id} not found");

    return Result<ProductDto>.Success(_mapper.Map<ProductDto>(product));
}

// Map to HTTP response in controller
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var result = await _mediator.Send(new GetProductQuery(id));
    return result.IsSuccess
        ? Ok(result.Value)
        : result.ErrorType switch
        {
            ResultErrorType.NotFound => NotFound(result.Error),
            ResultErrorType.Validation => BadRequest(result.Error),
            ResultErrorType.Forbidden => Forbid(),
            _ => StatusCode(500)
        };
}
```

## Specification Pattern

Encapsulate query criteria into reusable, composable specifications:

```csharp
public abstract class Specification<T> where T : class
{
    public Expression<Func<T, bool>>? Criteria { get; protected init; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public List<string> IncludeStrings { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; protected init; }
    public Expression<Func<T, object>>? OrderByDescending { get; protected init; }
    public int? Take { get; protected init; }
    public int? Skip { get; protected init; }
    public bool IsNoTracking { get; protected init; } = true;
}

// Concrete specification
public class ActiveProductsByCategorySpec : Specification<Product>
{
    public ActiveProductsByCategorySpec(int categoryId, int page, int pageSize)
    {
        Criteria = p => p.CategoryId == categoryId && p.IsActive;
        Includes.Add(p => p.Category);
        OrderBy = p => p.Name;
        Skip = (page - 1) * pageSize;
        Take = pageSize;
        IsNoTracking = true;
    }
}

// Evaluator in repository
public class SpecificationEvaluator<T> where T : class
{
    public static IQueryable<T> GetQuery(IQueryable<T> source, Specification<T> spec)
    {
        var query = source;

        if (spec.Criteria is not null)
            query = query.Where(spec.Criteria);

        query = spec.Includes.Aggregate(query,
            (current, include) => current.Include(include));

        query = spec.IncludeStrings.Aggregate(query,
            (current, include) => current.Include(include));

        if (spec.OrderBy is not null)
            query = query.OrderBy(spec.OrderBy);
        else if (spec.OrderByDescending is not null)
            query = query.OrderByDescending(spec.OrderByDescending);

        if (spec.IsNoTracking)
            query = query.AsNoTracking();

        if (spec.Skip.HasValue)
            query = query.Skip(spec.Skip.Value);

        if (spec.Take.HasValue)
            query = query.Take(spec.Take.Value);

        return query;
    }
}
```

## Domain Events

Decouple side effects from the core business operation:

```csharp
// Domain/Events/OrderPlacedEvent.cs
public record OrderPlacedEvent(int OrderId, int CustomerId, decimal Total) : INotification;

// Raise events from entities
public class Order : Entity
{
    public void Place()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Only draft orders can be placed");

        Status = OrderStatus.Placed;
        PlacedAt = DateTime.UtcNow;
        AddDomainEvent(new OrderPlacedEvent(Id, CustomerId, Total));
    }
}

// Base entity with event support
public abstract class Entity
{
    private readonly List<INotification> _domainEvents = new();
    public IReadOnlyList<INotification> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(INotification @event) => _domainEvents.Add(@event);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Dispatch events in SaveChangesAsync
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var entities = ChangeTracker.Entries<Entity>()
        .Where(e => e.Entity.DomainEvents.Any())
        .Select(e => e.Entity)
        .ToList();

    var domainEvents = entities.SelectMany(e => e.DomainEvents).ToList();
    entities.ForEach(e => e.ClearDomainEvents());

    var result = await base.SaveChangesAsync(ct);

    foreach (var domainEvent in domainEvents)
        await _mediator.Publish(domainEvent, ct);

    return result;
}

// Handle events
public class SendOrderConfirmationHandler : INotificationHandler<OrderPlacedEvent>
{
    private readonly IEmailService _email;

    public SendOrderConfirmationHandler(IEmailService email) => _email = email;

    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
    {
        await _email.SendOrderConfirmationAsync(notification.OrderId, ct);
    }
}
```

## Decorator Pattern for Cross-Cutting Concerns

Use MediatR pipeline behaviors as decorators:

```csharp
// Logging behavior
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}: {@Request}",
            requestName, request);

        var response = await next();

        _logger.LogInformation("Handled {RequestName}", requestName);
        return response;
    }
}

// Performance behavior
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    private readonly Stopwatch _timer = new();

    public PerformanceBehavior(
        ILogger<PerformanceBehavior<TRequest, TResponse>> logger) => _logger = logger;

    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        _timer.Start();
        var response = await next();
        _timer.Stop();

        if (_timer.ElapsedMilliseconds > 500)
        {
            _logger.LogWarning("Long running request: {RequestName} ({ElapsedMs}ms)",
                typeof(TRequest).Name, _timer.ElapsedMilliseconds);
        }

        return response;
    }
}
```

## Strategy Pattern for Flexible Business Rules

```csharp
// Define strategy interface
public interface IDiscountStrategy
{
    decimal CalculateDiscount(Order order);
}

// Concrete strategies
public class VolumeDiscountStrategy : IDiscountStrategy
{
    public decimal CalculateDiscount(Order order)
        => order.Items.Sum(i => i.Quantity) > 100 ? 0.10m : 0m;
}

public class LoyaltyDiscountStrategy : IDiscountStrategy
{
    public decimal CalculateDiscount(Order order)
        => order.Customer.OrderCount > 10 ? 0.05m : 0m;
}

// Register all strategies
services.AddScoped<IDiscountStrategy, VolumeDiscountStrategy>();
services.AddScoped<IDiscountStrategy, LoyaltyDiscountStrategy>();

// Composite usage — apply all applicable discounts
public class DiscountService
{
    private readonly IEnumerable<IDiscountStrategy> _strategies;

    public DiscountService(IEnumerable<IDiscountStrategy> strategies)
        => _strategies = strategies;

    public decimal CalculateTotalDiscount(Order order)
        => _strategies.Sum(s => s.CalculateDiscount(order));
}
```

## Factory Pattern for Object Creation

```csharp
public interface INotificationFactory
{
    INotificationSender Create(NotificationType type);
}

public class NotificationFactory : INotificationFactory
{
    private readonly IServiceProvider _serviceProvider;

    public NotificationFactory(IServiceProvider serviceProvider)
        => _serviceProvider = serviceProvider;

    public INotificationSender Create(NotificationType type) => type switch
    {
        NotificationType.Email => _serviceProvider.GetRequiredService<EmailSender>(),
        NotificationType.Sms => _serviceProvider.GetRequiredService<SmsSender>(),
        NotificationType.Push => _serviceProvider.GetRequiredService<PushSender>(),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}
```

## Pattern Selection Guide

| Scenario | Recommended Pattern |
|----------|-------------------|
| Data access abstraction | Repository + Unit of Work |
| Complex query composition | Specification |
| Side effects from business operations | Domain Events |
| Expected failure handling (not exceptions) | Result Pattern |
| Request pipeline cross-cutting concerns | MediatR Behaviors (Decorator) |
| Varying business rules | Strategy |
| Complex object construction | Builder / Factory |
| Feature organization | Vertical Slices or Clean Architecture |
