# Architecture Patterns for .NET

## Clean Architecture Structure

### Project Dependencies

Dependencies must always point inward. The domain layer is at the center with zero dependencies. The application layer references only the domain. Infrastructure and presentation reference application and domain.

```
MyApp.Domain           →  (no dependencies)
MyApp.Application      →  MyApp.Domain
MyApp.Infrastructure   →  MyApp.Application, MyApp.Domain
MyApp.WebApi           →  MyApp.Application, MyApp.Infrastructure (for DI registration only)
```

### Domain Layer Contents

```
MyApp.Domain/
  Entities/
    Order.cs
    OrderItem.cs
    Product.cs
    Customer.cs
  ValueObjects/
    Money.cs
    Address.cs
    Email.cs
  Enums/
    OrderStatus.cs
  Events/
    OrderCreatedEvent.cs
    OrderSubmittedEvent.cs
    OrderCancelledEvent.cs
  Interfaces/
    IOrderRepository.cs
    IProductRepository.cs
    IUnitOfWork.cs
  Exceptions/
    DomainException.cs
  Common/
    AggregateRoot.cs
    ValueObject.cs
    IHasDomainEvents.cs
```

### Application Layer Contents

```
MyApp.Application/
  Common/
    Behaviors/
      ValidationBehavior.cs
      LoggingBehavior.cs
    Interfaces/
      IEmailService.cs
      ICacheService.cs
    Models/
      Result.cs
      PagedResult.cs
  Orders/
    Commands/
      CreateOrder/
        CreateOrderCommand.cs
        CreateOrderCommandHandler.cs
        CreateOrderCommandValidator.cs
      CancelOrder/
        CancelOrderCommand.cs
        CancelOrderCommandHandler.cs
    Queries/
      GetOrderById/
        GetOrderByIdQuery.cs
        GetOrderByIdQueryHandler.cs
      ListOrders/
        ListOrdersQuery.cs
        ListOrdersQueryHandler.cs
    DTOs/
      OrderDto.cs
      OrderItemDto.cs
    EventHandlers/
      OrderSubmittedEventHandler.cs
  DependencyInjection.cs
```

### Infrastructure Layer Contents

```
MyApp.Infrastructure/
  Persistence/
    AppDbContext.cs
    Configurations/
      OrderConfiguration.cs
      ProductConfiguration.cs
    Repositories/
      OrderRepository.cs
      ProductRepository.cs
    Interceptors/
      AuditableInterceptor.cs
      SoftDeleteInterceptor.cs
    Migrations/
  Services/
    EmailService.cs
    CacheService.cs
    BlobStorageService.cs
  DependencyInjection.cs
```

### Registration Extensions

```csharp
// MyApp.Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly);
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        });

        services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);

        return services;
    }
}

// MyApp.Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("Default"),
                b => b.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IEmailService, EmailService>();
        services.AddSingleton<ICacheService, RedisCacheService>();

        return services;
    }
}
```

## Vertical Slice Architecture

Organize by feature instead of by layer. Each feature contains everything it needs — command/query, handler, validator, endpoint, and even its own DTOs.

### Feature Organization

```csharp
// Each feature is self-contained
namespace MyApp.Features.Products.CreateProduct;

// 1. Request
public record CreateProductCommand(
    string Name,
    decimal Price,
    string Category,
    string? Description) : IRequest<Result<ProductResponse>>;

// 2. Response DTO (local to this feature)
public record ProductResponse(int Id, string Name, decimal Price);

// 3. Validator
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator(AppDbContext db)
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200)
            .MustAsync(async (name, ct) =>
                !await db.Products.AnyAsync(p => p.Name == name, ct))
            .WithMessage("Product name already exists");

        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.Category).NotEmpty();
    }
}

// 4. Handler
public class CreateProductHandler(AppDbContext db) : IRequestHandler<CreateProductCommand, Result<ProductResponse>>
{
    public async Task<Result<ProductResponse>> Handle(
        CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category,
            Description = request.Description
        };

        db.Products.Add(product);
        await db.SaveChangesAsync(ct);

        return Result<ProductResponse>.Success(
            new ProductResponse(product.Id, product.Name, product.Price));
    }
}

// 5. Endpoint
public static class CreateProductEndpoint
{
    public static RouteGroupBuilder MapCreateProduct(this RouteGroupBuilder group)
    {
        group.MapPost("/", async (CreateProductCommand cmd, IMediator mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return result.Match(
                product => Results.Created($"/api/products/{product.Id}", product),
                error => Results.BadRequest(new { error }));
        })
        .WithName("CreateProduct")
        .Produces<ProductResponse>(StatusCodes.Status201Created)
        .ProducesValidationProblem();

        return group;
    }
}
```

### Feature Registration

```csharp
// Register all feature endpoints
public static class ProductEndpoints
{
    public static WebApplication MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products").WithTags("Products");

        group.MapCreateProduct();
        group.MapGetProduct();
        group.MapListProducts();
        group.MapUpdateProduct();
        group.MapDeleteProduct();

        return app;
    }
}

// In Program.cs
app.MapProductEndpoints();
app.MapOrderEndpoints();
```

## CQRS (Command Query Responsibility Segregation)

### Separate Read and Write Models

```csharp
// Write model — rich domain entity
public class Order : AggregateRoot
{
    public int Id { get; private set; }
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    // Business methods that enforce invariants
    public void AddItem(int productId, int quantity, decimal unitPrice) { /* ... */ }
    public void Submit() { /* ... */ }
    public void Cancel(string reason) { /* ... */ }
}

// Read model — flat DTO optimized for queries
public record OrderReadModel(
    int Id,
    string OrderNumber,
    string CustomerName,
    string Status,
    decimal Total,
    int ItemCount,
    DateTime CreatedAt);

// Separate read repository that returns projections
public interface IOrderReadRepository
{
    Task<OrderReadModel?> GetByIdAsync(int id, CancellationToken ct);
    Task<PagedResult<OrderReadModel>> ListAsync(OrderQueryParams queryParams, CancellationToken ct);
}

// Read repository implementation uses projections — never loads full entity graph
public class OrderReadRepository(AppDbContext db) : IOrderReadRepository
{
    public async Task<OrderReadModel?> GetByIdAsync(int id, CancellationToken ct) =>
        await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderReadModel(
                o.Id,
                o.OrderNumber,
                o.Customer.Name,
                o.Status.ToString(),
                o.Total.Amount,
                o.Items.Count,
                o.CreatedAt))
            .FirstOrDefaultAsync(ct);
}
```

## Event Sourcing

Store state changes as a sequence of events rather than overwriting current state.

```csharp
// Event base
public abstract record DomainEvent(DateTime OccurredAt)
{
    public Guid EventId { get; init; } = Guid.NewGuid();
}

// Order events
public record OrderPlacedEvent(int CustomerId, List<OrderItemData> Items, DateTime OccurredAt)
    : DomainEvent(OccurredAt);

public record OrderItemAddedEvent(int ProductId, int Quantity, decimal UnitPrice, DateTime OccurredAt)
    : DomainEvent(OccurredAt);

public record OrderCancelledEvent(string Reason, DateTime OccurredAt)
    : DomainEvent(OccurredAt);

// Event-sourced aggregate
public class Order
{
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }
    private readonly List<DomainEvent> _uncommittedEvents = [];

    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents.AsReadOnly();

    // Rebuild state from events
    public static Order FromHistory(IEnumerable<DomainEvent> events)
    {
        var order = new Order();
        foreach (var @event in events)
            order.Apply(@event);
        return order;
    }

    // Apply event to update state
    private void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case OrderPlacedEvent placed:
                Status = OrderStatus.Draft;
                break;
            case OrderItemAddedEvent itemAdded:
                Total += itemAdded.Quantity * itemAdded.UnitPrice;
                break;
            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;
        }
    }

    // Command methods raise events
    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Already cancelled");

        var @event = new OrderCancelledEvent(reason, DateTime.UtcNow);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }
}

// Event store interface
public interface IEventStore
{
    Task<List<DomainEvent>> GetEventsAsync(string streamId, CancellationToken ct);
    Task AppendEventsAsync(string streamId, IEnumerable<DomainEvent> events, CancellationToken ct);
}
```

## Domain Events Pattern

Raise domain events within aggregates and handle them after persistence to maintain consistency.

```csharp
// Dispatch domain events after SaveChanges
public class DomainEventDispatcher(IMediator mediator) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return result;

        var aggregates = eventData.Context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Count > 0)
            .Select(e => e.Entity)
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();

        // Clear events before dispatching to prevent re-dispatch
        foreach (var aggregate in aggregates)
            aggregate.ClearDomainEvents();

        foreach (var domainEvent in events)
            await mediator.Publish(domainEvent, ct);

        return result;
    }
}
```

## Specification Pattern

Encapsulate query criteria in reusable, composable specifications.

```csharp
// Base specification with Include support
public abstract class Specification<T> where T : class
{
    public abstract Expression<Func<T, bool>> Criteria { get; }
    public List<Expression<Func<T, object>>> Includes { get; } = [];
    public List<string> IncludeStrings { get; } = [];
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
    public Expression<Func<T, object>>? OrderByDescending { get; protected set; }
    public int? Take { get; protected set; }
    public int? Skip { get; protected set; }

    protected void AddInclude(Expression<Func<T, object>> includeExpression) =>
        Includes.Add(includeExpression);
}

// Concrete specification
public class ActiveOrdersByCustomerSpec : Specification<Order>
{
    public ActiveOrdersByCustomerSpec(int customerId, int pageSize = 20, int page = 1)
    {
        AddInclude(o => o.Items);
        OrderByDescending = o => o.CreatedAt;
        Skip = (page - 1) * pageSize;
        Take = pageSize;
    }

    public override Expression<Func<Order, bool>> Criteria =>
        o => o.Status != OrderStatus.Cancelled;
}

// Generic repository that evaluates specifications
public class Repository<T>(AppDbContext db) : IRepository<T> where T : class
{
    public async Task<List<T>> ListAsync(Specification<T> spec, CancellationToken ct)
    {
        var query = db.Set<T>().AsQueryable();

        query = query.Where(spec.Criteria);

        foreach (var include in spec.Includes)
            query = query.Include(include);

        if (spec.OrderBy is not null)
            query = query.OrderBy(spec.OrderBy);
        else if (spec.OrderByDescending is not null)
            query = query.OrderByDescending(spec.OrderByDescending);

        if (spec.Skip.HasValue)
            query = query.Skip(spec.Skip.Value);

        if (spec.Take.HasValue)
            query = query.Take(spec.Take.Value);

        return await query.ToListAsync(ct);
    }
}
```

## Result Pattern

Replace exceptions for expected failure modes with a typed Result that forces callers to handle both success and failure.

```csharp
// Result with error details
public class Result
{
    public bool IsSuccess { get; }
    public string? Error { get; }
    public ResultErrorType? ErrorType { get; }

    protected Result(bool isSuccess, string? error, ResultErrorType? errorType)
    {
        IsSuccess = isSuccess;
        Error = error;
        ErrorType = errorType;
    }

    public static Result Success() => new(true, null, null);
    public static Result Failure(string error, ResultErrorType type = ResultErrorType.Validation) =>
        new(false, error, type);
}

public class Result<T> : Result
{
    public T? Value { get; }

    private Result(T? value, bool isSuccess, string? error, ResultErrorType? errorType)
        : base(isSuccess, error, errorType)
    {
        Value = value;
    }

    public static Result<T> Success(T value) => new(value, true, null, null);
    public static new Result<T> Failure(string error, ResultErrorType type = ResultErrorType.Validation) =>
        new(default, false, error, type);
    public static Result<T> NotFound(string error) =>
        new(default, false, error, ResultErrorType.NotFound);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<string, TOut> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

public enum ResultErrorType
{
    Validation,
    NotFound,
    Conflict,
    Unauthorized,
    Forbidden
}

// Map Result to HTTP responses
public static class ResultExtensions
{
    public static IResult ToHttpResult<T>(this Result<T> result) =>
        result.Match<IResult>(
            value => Results.Ok(value),
            error => result.ErrorType switch
            {
                ResultErrorType.NotFound => Results.NotFound(new { error }),
                ResultErrorType.Conflict => Results.Conflict(new { error }),
                ResultErrorType.Unauthorized => Results.Unauthorized(),
                ResultErrorType.Forbidden => Results.Forbid(),
                _ => Results.BadRequest(new { error })
            });
}
```
