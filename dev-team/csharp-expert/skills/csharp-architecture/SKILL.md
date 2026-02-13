---
name: C# Architecture
description: This skill should be used when the user asks about "Clean Architecture", "CQRS", "MediatR", ".NET architecture", "domain-driven design", or "vertical slice". It covers Clean Architecture layer separation, CQRS with MediatR, DDD building blocks, vertical slice architecture, middleware pipeline design, and ASP.NET Core infrastructure patterns.
---

# C# Architecture

## Clean Architecture Overview

Clean Architecture organizes code into concentric layers with dependencies pointing inward. The domain layer has zero external dependencies. Application logic depends only on the domain. Infrastructure and presentation depend on application and domain but never the reverse.

### Layer Structure

```
src/
  MyApp.Domain/           # Entities, value objects, domain events, interfaces
  MyApp.Application/      # Use cases, DTOs, validators, service interfaces
  MyApp.Infrastructure/   # EF Core, external services, file system, email
  MyApp.WebApi/           # Controllers, middleware, startup, DI composition
```

### Domain Layer

The domain layer contains entities, value objects, domain events, and repository interfaces. It has no dependencies on any other project or NuGet package (except for base .NET types).

```csharp
// Domain entity with encapsulated business logic
namespace MyApp.Domain.Entities;

public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = [];

    public int Id { get; private set; }
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; } = Money.Zero("USD");
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public DateTime CreatedAt { get; private set; }

    private Order() { } // EF Core constructor

    public static Order Create(int customerId)
    {
        var order = new Order
        {
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            CreatedAt = DateTime.UtcNow
        };

        order.AddDomainEvent(new OrderCreatedEvent(order));
        return order;
    }

    public void AddItem(int productId, int quantity, decimal unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Can only add items to draft orders");

        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing is not null)
        {
            existing.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(productId, quantity, unitPrice));
        }

        RecalculateTotal();
    }

    public void Submit()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Only draft orders can be submitted");
        if (_items.Count == 0)
            throw new DomainException("Cannot submit an empty order");

        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(this));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Order is already cancelled");
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot cancel a shipped order");

        Status = OrderStatus.Cancelled;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    private void RecalculateTotal()
    {
        Total = _items.Aggregate(
            Money.Zero("USD"),
            (sum, item) => sum.Add(new Money(item.Quantity * item.UnitPrice, "USD")));
    }
}
```

### Application Layer

The application layer contains use cases (commands and queries), DTOs, validators, and service interfaces.

```csharp
// Command
namespace MyApp.Application.Orders.Commands;

public record CreateOrderCommand(
    int CustomerId,
    List<OrderItemDto> Items) : IRequest<Result<OrderDto>>;

// Command handler
public class CreateOrderCommandHandler(
    IOrderRepository orderRepository,
    IProductRepository productRepository,
    IUnitOfWork unitOfWork) : IRequestHandler<CreateOrderCommand, Result<OrderDto>>
{
    public async Task<Result<OrderDto>> Handle(
        CreateOrderCommand request,
        CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            var product = await productRepository.GetByIdAsync(item.ProductId, ct);
            if (product is null)
                return Result<OrderDto>.Failure($"Product {item.ProductId} not found");

            order.AddItem(product.Id, item.Quantity, product.Price);
        }

        order.Submit();
        await orderRepository.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return Result<OrderDto>.Success(order.ToDto());
    }
}

// Query
public record GetOrderByIdQuery(int OrderId) : IRequest<OrderDto?>;

public class GetOrderByIdQueryHandler(
    IOrderReadRepository repository) : IRequestHandler<GetOrderByIdQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderByIdQuery request, CancellationToken ct)
    {
        return await repository.GetOrderDtoByIdAsync(request.OrderId, ct);
    }
}
```

### Infrastructure Layer

The infrastructure layer implements interfaces defined in the domain and application layers.

```csharp
namespace MyApp.Infrastructure.Persistence;

public class OrderRepository(AppDbContext dbContext) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct) =>
        await dbContext.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await dbContext.Orders.AddAsync(order, ct);
}

public class UnitOfWork(AppDbContext dbContext) : IUnitOfWork
{
    public async Task<int> SaveChangesAsync(CancellationToken ct) =>
        await dbContext.SaveChangesAsync(ct);
}
```

### Presentation Layer (WebApi)

The presentation layer composes all dependencies and exposes the API.

```csharp
// Program.cs — composition root
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddDomain()
    .AddApplication()
    .AddInfrastructure(builder.Configuration)
    .AddPresentation();

var app = builder.Build();

app.UseExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## CQRS with MediatR

Separate read and write models. Commands modify state and return results. Queries read state and return data. MediatR provides the mediator to dispatch them.

### Pipeline Behaviors

Use MediatR pipeline behaviors for cross-cutting concerns.

```csharp
// Validation behavior — runs before every handler
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(
                validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Logging behavior
public class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);

        var stopwatch = Stopwatch.StartNew();
        var response = await next();
        stopwatch.Stop();

        logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms",
            requestName, stopwatch.ElapsedMilliseconds);

        return response;
    }
}

// Registration
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});
```

### Notifications for Domain Events

```csharp
// Domain event
public record OrderSubmittedEvent(Order Order) : INotification;

// Multiple handlers can react to the same event
public class SendOrderConfirmationHandler(
    IEmailService emailService,
    ICustomerRepository customerRepository)
    : INotificationHandler<OrderSubmittedEvent>
{
    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        var customer = await customerRepository.GetByIdAsync(
            notification.Order.CustomerId, ct);
        if (customer is null) return;

        await emailService.SendOrderConfirmationAsync(customer.Email, notification.Order, ct);
    }
}

public class UpdateInventoryHandler(
    IInventoryService inventoryService)
    : INotificationHandler<OrderSubmittedEvent>
{
    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        foreach (var item in notification.Order.Items)
        {
            await inventoryService.ReserveStockAsync(item.ProductId, item.Quantity, ct);
        }
    }
}
```

## Domain-Driven Design Building Blocks

### Aggregate Root Base Class

```csharp
public abstract class AggregateRoot
{
    private readonly List<INotification> _domainEvents = [];
    public IReadOnlyList<INotification> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(INotification domainEvent) =>
        _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### Value Objects

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType())
            return false;

        var other = (ValueObject)obj;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode() =>
        GetEqualityComponents()
            .Aggregate(0, (hash, component) =>
                HashCode.Combine(hash, component?.GetHashCode() ?? 0));

    public static bool operator ==(ValueObject? left, ValueObject? right) =>
        Equals(left, right);

    public static bool operator !=(ValueObject? left, ValueObject? right) =>
        !Equals(left, right);
}

// Concrete value object
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                $"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

### Specification Pattern

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other) =>
        new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other) =>
        new OrSpecification<T>(this, other);

    public Specification<T> Not() => new NotSpecification<T>(this);
}

// Concrete specification
public class ActiveOrdersSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression() =>
        order => order.Status != OrderStatus.Cancelled
              && order.Status != OrderStatus.Completed;
}

public class HighValueOrdersSpec : Specification<Order>
{
    private readonly decimal _threshold;
    public HighValueOrdersSpec(decimal threshold) => _threshold = threshold;

    public override Expression<Func<Order, bool>> ToExpression() =>
        order => order.Total.Amount >= _threshold;
}

// Usage with repository
var spec = new ActiveOrdersSpec().And(new HighValueOrdersSpec(500));
var orders = await orderRepository.ListAsync(spec, ct);
```

### Result Pattern

```csharp
public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T? value, string? error) => (Value, Error) = (value, error);

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);

    public TOut Match<TOut>(Func<T, TOut> success, Func<string, TOut> failure) =>
        IsSuccess ? success(Value!) : failure(Error!);

    public Result<TOut> Map<TOut>(Func<T, TOut> mapper) =>
        IsSuccess ? Result<TOut>.Success(mapper(Value!)) : Result<TOut>.Failure(Error!);

    public async Task<Result<TOut>> Bind<TOut>(Func<T, Task<Result<TOut>>> binder) =>
        IsSuccess ? await binder(Value!) : Result<TOut>.Failure(Error!);
}
```

## Vertical Slice Architecture

Organize code by feature rather than by layer. Each feature is a self-contained slice that crosses all layers.

```
Features/
  Products/
    CreateProduct/
      CreateProductCommand.cs
      CreateProductHandler.cs
      CreateProductValidator.cs
      CreateProductEndpoint.cs
    GetProduct/
      GetProductQuery.cs
      GetProductHandler.cs
      GetProductEndpoint.cs
    ListProducts/
      ListProductsQuery.cs
      ListProductsHandler.cs
      ListProductsEndpoint.cs
  Orders/
    CreateOrder/
      CreateOrderCommand.cs
      CreateOrderHandler.cs
      CreateOrderValidator.cs
      CreateOrderEndpoint.cs
```

```csharp
// Self-contained feature slice
namespace MyApp.Features.Products.CreateProduct;

public record CreateProductCommand(string Name, decimal Price, string Category)
    : IRequest<Result<int>>;

public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.Category).NotEmpty();
    }
}

public class CreateProductHandler(AppDbContext db)
    : IRequestHandler<CreateProductCommand, Result<int>>
{
    public async Task<Result<int>> Handle(
        CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category
        };

        db.Products.Add(product);
        await db.SaveChangesAsync(ct);
        return Result<int>.Success(product.Id);
    }
}

public static class CreateProductEndpoint
{
    public static void Map(RouteGroupBuilder group) =>
        group.MapPost("/", async (
            CreateProductCommand command,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(command, ct);
            return result.Match(
                id => Results.Created($"/api/products/{id}", new { id }),
                error => Results.BadRequest(error));
        });
}
```

## Middleware Pipeline

Design the ASP.NET Core middleware pipeline as an ordered sequence of cross-cutting concerns.

```csharp
var app = builder.Build();

// Order matters — each middleware wraps the next
app.UseExceptionHandler();              // Outermost: catches all unhandled exceptions
app.UseHsts();                          // HTTP Strict Transport Security
app.UseHttpsRedirection();              // Redirect HTTP to HTTPS
app.UseCors("AllowFrontend");           // CORS policy
app.UseAuthentication();                // Identify the user
app.UseAuthorization();                 // Verify user permissions
app.UseRateLimiter();                   // Throttle requests
app.UseOutputCache();                   // Return cached responses
app.UseResponseCompression();           // Compress response body
app.MapControllers();                   // Terminal: route to endpoints
app.MapHealthChecks("/health");
```

## References

- [Architecture Patterns](references/architecture-patterns.md) — Clean Architecture structure, Vertical Slice, CQRS, Event Sourcing, Domain Events, Specification pattern, and Result pattern.
- [ASP.NET Core Patterns](references/aspnet-patterns.md) — Minimal APIs vs Controllers, middleware pipeline, Options pattern, health checks, background services, output caching, and rate limiting.
