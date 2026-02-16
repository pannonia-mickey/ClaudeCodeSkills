---
name: .NET Architecture
description: This skill should be used when the user asks about "Clean Architecture", "CQRS", "MediatR", ".NET architecture", "domain-driven design", "vertical slice", "middleware pipeline", "dependency injection setup", or "project structure in .NET". It covers Clean Architecture layer separation, CQRS with MediatR, DDD building blocks, vertical slice architecture, middleware pipeline design, and ASP.NET Core infrastructure patterns.
---

# .NET Architecture

## Clean Architecture Overview

Clean Architecture organizes code into concentric layers with dependencies pointing inward. The domain layer has zero external dependencies.

### Layer Structure

```
src/
  MyApp.Domain/           # Entities, value objects, domain events, interfaces
  MyApp.Application/      # Use cases, DTOs, validators, service interfaces
  MyApp.Infrastructure/   # EF Core, external services, file system, email
  MyApp.WebApi/           # Controllers, middleware, startup, DI composition
```

### Domain Layer

Contains entities with encapsulated business logic, value objects, domain events, and repository interfaces.

```csharp
public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = [];
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public static Order Create(int customerId)
    {
        var order = new Order { CustomerId = customerId, Status = OrderStatus.Draft };
        order.AddDomainEvent(new OrderCreatedEvent(order));
        return order;
    }

    public void Submit()
    {
        if (Status != OrderStatus.Draft) throw new DomainException("Only draft orders can be submitted");
        if (_items.Count == 0) throw new DomainException("Cannot submit an empty order");
        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(this));
    }
}
```

### Application Layer — CQRS with MediatR

```csharp
// Command
public record CreateOrderCommand(int CustomerId, List<OrderItemDto> Items) : IRequest<Result<OrderDto>>;

// Handler
public class CreateOrderCommandHandler(
    IOrderRepository orderRepository, IUnitOfWork unitOfWork)
    : IRequestHandler<CreateOrderCommand, Result<OrderDto>>
{
    public async Task<Result<OrderDto>> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);
        foreach (var item in request.Items)
            order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
        order.Submit();
        await orderRepository.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);
        return Result<OrderDto>.Success(order.ToDto());
    }
}
```

### Pipeline Behaviors

```csharp
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();
        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors).Where(f => f is not null).ToList();
        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}

// Registration
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
});
```

## DDD Building Blocks

### Value Objects

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();
    public override bool Equals(object? obj) =>
        obj is ValueObject other && GetType() == other.GetType()
        && GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    public override int GetHashCode() =>
        GetEqualityComponents().Aggregate(0, (hash, c) => HashCode.Combine(hash, c?.GetHashCode() ?? 0));
}
```

### Specification Pattern

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);
    public Specification<T> And(Specification<T> other) => new AndSpecification<T>(this, other);
}

var spec = new ActiveOrdersSpec().And(new HighValueOrdersSpec(500));
var orders = await orderRepository.ListAsync(spec, ct);
```

## Vertical Slice Architecture

Organize code by feature rather than by layer:

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
```

## Middleware Pipeline

```csharp
app.UseExceptionHandler();
app.UseHsts();
app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.UseOutputCache();
app.MapControllers();
app.MapHealthChecks("/health");
```

## DI Registration per Layer

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(config.GetConnectionString("Default"),
                b => b.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
        services.AddScoped<IProductRepository, ProductRepository>();
        return services;
    }
}

// Program.cs
builder.Services.AddApplication().AddInfrastructure(builder.Configuration).AddPresentation();
```

## References

- [Architecture Patterns](references/architecture-patterns.md) — Clean Architecture, Vertical Slice, CQRS, Event Sourcing, Domain Events, Specification pattern, Result pattern.
- [ASP.NET Core Patterns](references/aspnet-patterns.md) — Minimal APIs vs Controllers, middleware pipeline, Options pattern, health checks, background services, output caching, rate limiting.
