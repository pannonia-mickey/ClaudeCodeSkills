# Architecture Patterns in .NET

## Clean Architecture — Detailed Structure

```
src/
  MyApp.Domain/              # Zero external dependencies
    Entities/                # Aggregate roots, entities
    ValueObjects/            # Money, Address, Email
    Events/                  # Domain events (INotification)
    Exceptions/              # DomainException, NotFoundException
    Interfaces/              # IOrderRepository, IUnitOfWork
    Enums/                   # OrderStatus, PaymentType
  MyApp.Application/         # Depends only on Domain
    Common/
      Behaviors/             # ValidationBehavior, LoggingBehavior
      Interfaces/            # IApplicationDbContext, IEmailSender
      Models/                # Result<T>, PaginatedList<T>
    Features/
      Orders/
        Commands/            # CreateOrder, CancelOrder
        Queries/             # GetOrderById, ListOrders
        DTOs/                # OrderDto, OrderListDto
  MyApp.Infrastructure/      # Depends on Application + Domain
    Persistence/
      Configurations/        # IEntityTypeConfiguration<T>
      Migrations/
      Repositories/
    Services/                # SmtpEmailSender, BlobStorageService
    DependencyInjection.cs
  MyApp.WebApi/              # Composition root
    Controllers/
    Middleware/
    Program.cs
```

**Project references:** WebApi → Infrastructure → Application → Domain

## Event Sourcing Basics

```csharp
public abstract class EventSourcedAggregate
{
    private readonly List<IDomainEvent> _uncommittedEvents = [];
    public int Version { get; protected set; }

    protected void Apply(IDomainEvent @event)
    {
        When(@event);
        Version++;
        _uncommittedEvents.Add(@event);
    }

    protected abstract void When(IDomainEvent @event);

    public IReadOnlyList<IDomainEvent> GetUncommittedEvents() => _uncommittedEvents.AsReadOnly();
    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();
}
```

## Domain Events with MediatR

```csharp
public record OrderSubmittedEvent(Order Order) : INotification;

public class SendOrderConfirmationHandler(IEmailService email, ICustomerRepository customers)
    : INotificationHandler<OrderSubmittedEvent>
{
    public async Task Handle(OrderSubmittedEvent notification, CancellationToken ct)
    {
        var customer = await customers.GetByIdAsync(notification.Order.CustomerId, ct);
        if (customer is not null)
            await email.SendOrderConfirmationAsync(customer.Email, notification.Order, ct);
    }
}

// Dispatch domain events after SaveChanges
public class DomainEventDispatcher(IMediator mediator) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(SaveChangesCompletedEventData eventData, int result, CancellationToken ct)
    {
        var aggregates = eventData.Context!.ChangeTracker.Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Any()).Select(e => e.Entity).ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearDomainEvents());

        foreach (var domainEvent in events)
            await mediator.Publish(domainEvent, ct);

        return result;
    }
}
```

## Result Pattern with Map/Bind

```csharp
public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T? value, string? error) => (Value, Error) = (value, error);

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);

    public Result<TOut> Map<TOut>(Func<T, TOut> mapper) =>
        IsSuccess ? Result<TOut>.Success(mapper(Value!)) : Result<TOut>.Failure(Error!);

    public async Task<Result<TOut>> Bind<TOut>(Func<T, Task<Result<TOut>>> binder) =>
        IsSuccess ? await binder(Value!) : Result<TOut>.Failure(Error!);

    public TOut Match<TOut>(Func<T, TOut> success, Func<string, TOut> failure) =>
        IsSuccess ? success(Value!) : failure(Error!);
}
```

## Specification Pattern (Expression-Based)

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public Specification<T> And(Specification<T> other) => new AndSpecification<T>(this, other);
    public Specification<T> Or(Specification<T> other) => new OrSpecification<T>(this, other);
    public Specification<T> Not() => new NotSpecification<T>(this);
}

internal class AndSpecification<T>(Specification<T> left, Specification<T> right) : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}
```
