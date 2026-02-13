# ASP.NET Core MVC Project Structure

Detailed reference for organizing ASP.NET Core MVC solutions following Clean Architecture principles.

## Solution Structure

### Clean Architecture Solution

```
MySolution/
├── MySolution.sln
├── src/
│   ├── MySolution.Domain/
│   │   ├── MySolution.Domain.csproj        # No project references
│   │   ├── Common/
│   │   │   ├── Entity.cs                   # Base entity with Id
│   │   │   ├── AuditableEntity.cs          # Created/Updated timestamps
│   │   │   ├── ValueObject.cs              # Base value object
│   │   │   └── DomainException.cs
│   │   ├── Entities/
│   │   │   ├── Product.cs
│   │   │   ├── Category.cs
│   │   │   └── Order.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   └── Address.cs
│   │   ├── Enums/
│   │   │   └── OrderStatus.cs
│   │   ├── Events/
│   │   │   ├── OrderPlacedEvent.cs
│   │   │   └── ProductCreatedEvent.cs
│   │   ├── Exceptions/
│   │   │   └── DomainException.cs
│   │   └── Interfaces/
│   │       ├── IProductRepository.cs
│   │       ├── IOrderRepository.cs
│   │       └── IUnitOfWork.cs
│   │
│   ├── MySolution.Application/
│   │   ├── MySolution.Application.csproj    # References: Domain
│   │   ├── Common/
│   │   │   ├── Behaviors/
│   │   │   │   ├── ValidationBehavior.cs
│   │   │   │   ├── LoggingBehavior.cs
│   │   │   │   └── PerformanceBehavior.cs
│   │   │   ├── Exceptions/
│   │   │   │   ├── NotFoundException.cs
│   │   │   │   └── ForbiddenException.cs
│   │   │   ├── Interfaces/
│   │   │   │   ├── IEmailService.cs
│   │   │   │   ├── ICurrentUserService.cs
│   │   │   │   └── IDateTime.cs
│   │   │   ├── Mappings/
│   │   │   │   └── MappingProfile.cs
│   │   │   └── Models/
│   │   │       ├── Result.cs
│   │   │       └── PaginatedList.cs
│   │   ├── Features/
│   │   │   ├── Products/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateProduct/
│   │   │   │   │   │   ├── CreateProductCommand.cs
│   │   │   │   │   │   ├── CreateProductHandler.cs
│   │   │   │   │   │   └── CreateProductValidator.cs
│   │   │   │   │   ├── UpdateProduct/
│   │   │   │   │   └── DeleteProduct/
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetProduct/
│   │   │   │   │   │   ├── GetProductQuery.cs
│   │   │   │   │   │   ├── GetProductHandler.cs
│   │   │   │   │   │   └── ProductDto.cs
│   │   │   │   │   └── GetProductsList/
│   │   │   │   └── EventHandlers/
│   │   │   │       └── ProductCreatedEventHandler.cs
│   │   │   └── Orders/
│   │   │       ├── Commands/
│   │   │       ├── Queries/
│   │   │       └── EventHandlers/
│   │   └── DependencyInjection.cs
│   │
│   ├── MySolution.Infrastructure/
│   │   ├── MySolution.Infrastructure.csproj # References: Application
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   │   ├── ProductConfiguration.cs
│   │   │   │   ├── CategoryConfiguration.cs
│   │   │   │   └── OrderConfiguration.cs
│   │   │   ├── Migrations/
│   │   │   ├── Repositories/
│   │   │   │   ├── ProductRepository.cs
│   │   │   │   └── OrderRepository.cs
│   │   │   ├── Interceptors/
│   │   │   │   ├── AuditableEntityInterceptor.cs
│   │   │   │   └── DomainEventDispatcherInterceptor.cs
│   │   │   └── Seed/
│   │   │       └── AppDbContextSeed.cs
│   │   ├── Services/
│   │   │   ├── EmailService.cs
│   │   │   ├── DateTimeService.cs
│   │   │   └── CurrentUserService.cs
│   │   ├── Identity/
│   │   │   ├── IdentityService.cs
│   │   │   └── TokenService.cs
│   │   └── DependencyInjection.cs
│   │
│   └── MySolution.WebUI/
│       ├── MySolution.WebUI.csproj          # References: Application, Infrastructure
│       ├── Program.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       ├── Controllers/
│       │   ├── ApiControllerBase.cs
│       │   ├── ProductsController.cs
│       │   └── OrdersController.cs
│       ├── Filters/
│       │   └── ApiExceptionFilterAttribute.cs
│       ├── Middleware/
│       │   ├── GlobalExceptionMiddleware.cs
│       │   └── RequestLoggingMiddleware.cs
│       ├── Models/
│       │   └── ApiResponse.cs
│       ├── Views/
│       │   ├── Shared/
│       │   │   ├── _Layout.cshtml
│       │   │   ├── _ValidationScriptsPartial.cshtml
│       │   │   └── Error.cshtml
│       │   ├── _ViewImports.cshtml
│       │   └── _ViewStart.cshtml
│       └── wwwroot/
│           ├── css/
│           ├── js/
│           └── lib/
│
└── tests/
    ├── MySolution.Domain.UnitTests/
    │   └── MySolution.Domain.UnitTests.csproj
    ├── MySolution.Application.UnitTests/
    │   └── MySolution.Application.UnitTests.csproj
    ├── MySolution.Infrastructure.IntegrationTests/
    │   └── MySolution.Infrastructure.IntegrationTests.csproj
    └── MySolution.WebUI.IntegrationTests/
        └── MySolution.WebUI.IntegrationTests.csproj
```

## Layer Responsibilities

### Domain Layer

**Purpose**: Core business logic, entities, value objects, domain events

**Rules**:
- Zero external dependencies (no NuGet packages except MediatR.Contracts for INotification)
- No knowledge of other layers
- Contains only: entities, value objects, enums, exceptions, interfaces, domain events
- Entities enforce their own invariants

**Key files**:
```csharp
// Entity base class
public abstract class Entity
{
    public int Id { get; protected set; }

    private readonly List<INotification> _domainEvents = new();
    public IReadOnlyList<INotification> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(INotification @event) => _domainEvents.Add(@event);
    public void ClearDomainEvents() => _domainEvents.Clear();

    public override bool Equals(object? obj)
        => obj is Entity other && Id == other.Id;

    public override int GetHashCode() => Id.GetHashCode();
}

// Rich domain entity
public class Order : AuditableEntity
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public decimal Total => _items.Sum(i => i.LineTotal);

    public void AddItem(Product product, int quantity)
    {
        if (quantity <= 0)
            throw new DomainException("Quantity must be positive");

        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a non-draft order");

        _items.Add(new OrderItem(product.Id, product.Price, quantity));
    }

    public void Place()
    {
        if (!_items.Any())
            throw new DomainException("Cannot place an empty order");

        Status = OrderStatus.Placed;
        AddDomainEvent(new OrderPlacedEvent(Id));
    }
}
```

### Application Layer

**Purpose**: Use cases, orchestration, DTOs, validation, mappings

**Rules**:
- References Domain only
- Contains CQRS commands/queries, handlers, validators, DTOs
- Defines interfaces for infrastructure services (e.g., `IEmailService`)
- No direct database or external service access

**Key conventions**:
- One folder per feature: `Features/Products/Commands/CreateProduct/`
- Each command/query has its own handler and validator
- DTOs live next to their query handlers
- MediatR pipeline behaviors handle cross-cutting concerns

### Infrastructure Layer

**Purpose**: External concerns — database, email, file system, third-party APIs

**Rules**:
- References Application (for interface implementations)
- Contains EF Core DbContext, configurations, repositories
- Implements all interfaces defined in Application
- Handles external service integration

**DI Registration pattern**:
```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, IConfiguration configuration)
    {
        // Database
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        // Repositories
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());

        // Services
        services.AddTransient<IEmailService, EmailService>();
        services.AddSingleton<IDateTime, DateTimeService>();

        // Identity
        services.AddTransient<ITokenService, TokenService>();

        return services;
    }
}
```

### WebUI Layer (Presentation)

**Purpose**: HTTP endpoints, middleware, view rendering, API surface

**Rules**:
- References Application and Infrastructure
- Controllers are thin — delegate to MediatR
- Contains middleware, filters, view models
- Maps between HTTP and application concerns

**Thin controller pattern**:
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ISender _mediator;

    public ProductsController(ISender mediator) => _mediator = mediator;

    [HttpGet]
    public async Task<ActionResult<PaginatedList<ProductDto>>> GetAll(
        [FromQuery] GetProductsListQuery query)
        => Ok(await _mediator.Send(query));

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> Get(int id)
        => Ok(await _mediator.Send(new GetProductQuery(id)));

    [HttpPost]
    [Authorize(Policy = "CanManageProducts")]
    public async Task<ActionResult<int>> Create(CreateProductCommand command)
    {
        var id = await _mediator.Send(command);
        return CreatedAtAction(nameof(Get), new { id }, id);
    }

    [HttpPut("{id}")]
    [Authorize(Policy = "CanManageProducts")]
    public async Task<IActionResult> Update(int id, UpdateProductCommand command)
    {
        if (id != command.Id) return BadRequest();
        await _mediator.Send(command);
        return NoContent();
    }

    [HttpDelete("{id}")]
    [Authorize(Policy = "CanManageProducts")]
    public async Task<IActionResult> Delete(int id)
    {
        await _mediator.Send(new DeleteProductCommand(id));
        return NoContent();
    }
}
```

## Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Solution | PascalCase | `MyCompany.ProductCatalog` |
| Project | Solution.Layer | `MyCompany.ProductCatalog.Domain` |
| Entity | Singular noun | `Product`, `Order`, `Category` |
| Repository interface | `I` + Entity + `Repository` | `IProductRepository` |
| Repository class | Entity + `Repository` | `ProductRepository` |
| Command | Verb + Entity + `Command` | `CreateProductCommand` |
| Query | `Get` + Entity + `Query` | `GetProductQuery` |
| Handler | Command/Query + `Handler` | `CreateProductHandler` |
| Validator | Command + `Validator` | `CreateProductValidator` |
| DTO | Entity + `Dto` | `ProductDto` |
| Configuration | Entity + `Configuration` | `ProductConfiguration` |
| Controller | Entity(plural) + `Controller` | `ProductsController` |
| Service interface | `I` + Name + `Service` | `IEmailService` |
| Service class | Name + `Service` | `EmailService` |

## Project References

```
Domain          → (none)
Application     → Domain
Infrastructure  → Application
WebUI           → Application, Infrastructure
Tests           → Layer being tested + required dependencies
```

**csproj reference examples**:
```xml
<!-- Application.csproj -->
<ItemGroup>
    <ProjectReference Include="..\MySolution.Domain\MySolution.Domain.csproj" />
</ItemGroup>

<!-- Infrastructure.csproj -->
<ItemGroup>
    <ProjectReference Include="..\MySolution.Application\MySolution.Application.csproj" />
</ItemGroup>

<!-- WebUI.csproj -->
<ItemGroup>
    <ProjectReference Include="..\MySolution.Application\MySolution.Application.csproj" />
    <ProjectReference Include="..\MySolution.Infrastructure\MySolution.Infrastructure.csproj" />
</ItemGroup>
```

## Program.cs Composition

```csharp
var builder = WebApplication.CreateBuilder(args);

// Layer registrations
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

// Web layer
builder.Services.AddControllersWithViews(options =>
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute()));

builder.Services.AddAuthentication(/* ... */);
builder.Services.AddAuthorization(/* ... */);

var app = builder.Build();

// Middleware pipeline (order matters)
app.UseExceptionHandler("/error");
app.UseHsts();
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();

// Expose Program class for integration tests
public partial class Program { }
```

## Configuration Management

```
appsettings.json                    # Base configuration (non-secret defaults)
appsettings.Development.json        # Development overrides
appsettings.Production.json         # Production overrides (non-secret)
User Secrets                        # Local development secrets
Environment Variables               # Production secrets
Azure Key Vault / AWS Secrets       # Cloud secrets management
```

**Never store secrets in appsettings.json.** Use User Secrets for development and environment variables or a vault for production.
