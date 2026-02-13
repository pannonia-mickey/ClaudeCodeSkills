---
name: aspnet-mvc-architecture
description: This skill should be used when the user asks to "set up project structure", "implement clean architecture", "add dependency injection", "configure middleware", "create a service layer", "implement repository pattern", "add MediatR", "set up CQRS", "organize ASP.NET project", or needs guidance on ASP.NET Core MVC architecture and design patterns.
---

# ASP.NET MVC Architecture & Design Patterns

Expert guidance for structuring ASP.NET Core MVC applications using proven architectural patterns and SOLID design principles.

## Core Architectural Patterns

### Clean Architecture (Recommended Default)

Organize solutions into concentric layers with dependencies pointing inward:

```
Solution/
├── src/
│   ├── Domain/                  # Entities, value objects, domain events, interfaces
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Enums/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── Interfaces/          # Repository & service contracts
│   ├── Application/             # Use cases, DTOs, validators, mappings
│   │   ├── Common/
│   │   │   ├── Behaviors/       # MediatR pipeline behaviors
│   │   │   ├── Interfaces/
│   │   │   ├── Mappings/
│   │   │   └── Models/
│   │   └── Features/
│   │       └── Products/
│   │           ├── Commands/
│   │           ├── Queries/
│   │           └── DTOs/
│   ├── Infrastructure/          # EF Core, external services, file system
│   │   ├── Persistence/
│   │   │   ├── Configurations/  # EF entity configs
│   │   │   ├── Migrations/
│   │   │   └── Repositories/
│   │   ├── Services/
│   │   └── DependencyInjection.cs
│   └── WebUI/                   # Controllers, views, filters, middleware
│       ├── Controllers/
│       ├── Filters/
│       ├── Middleware/
│       ├── Views/
│       └── Program.cs
└── tests/
    ├── Domain.UnitTests/
    ├── Application.UnitTests/
    ├── Infrastructure.IntegrationTests/
    └── WebUI.IntegrationTests/
```

**Dependency rule**: Domain depends on nothing. Application depends on Domain. Infrastructure and WebUI depend on Application.

### Vertical Slice Architecture (Alternative)

Organize by feature instead of layer when the project has many independent features:

```csharp
// Features/Products/CreateProduct.cs — everything for one use case in one file
public static class CreateProduct
{
    public record Command(string Name, decimal Price) : IRequest<int>;

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
            RuleFor(x => x.Price).GreaterThan(0);
        }
    }

    public class Handler : IRequestHandler<Command, int>
    {
        private readonly AppDbContext _db;
        public Handler(AppDbContext db) => _db = db;

        public async Task<int> Handle(Command request, CancellationToken ct)
        {
            var product = new Product(request.Name, request.Price);
            _db.Products.Add(product);
            await _db.SaveChangesAsync(ct);
            return product.Id;
        }
    }
}
```

## Dependency Injection Registration

Structure DI registration per layer using extension methods:

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));

        services.AddScoped<IUnitOfWork>(provider =>
            provider.GetRequiredService<AppDbContext>());

        services.AddScoped<IProductRepository, ProductRepository>();

        return services;
    }
}

// Program.cs — compose layers
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
```

**Service lifetime guidelines:**
| Lifetime | Use For | Examples |
|-----------|---------|----------|
| Scoped | Per-request state, DbContext | Repositories, UnitOfWork, DbContext |
| Singleton | Stateless shared services | Caching, HttpClientFactory, IOptions |
| Transient | Lightweight, stateless | Validators, Mappers |

## Middleware Pipeline

Configure the middleware pipeline in the correct order:

```csharp
// Program.cs — order matters
app.UseExceptionHandler("/error");
app.UseHsts();
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors("AllowedOrigins");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();
```

## MediatR + CQRS Pipeline

Set up a behavioral pipeline with cross-cutting concerns:

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

Register in DI:
```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(typeof(ApplicationAssemblyMarker).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
});
```

## Global Exception Handling

Implement centralized error handling via middleware:

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = 400,
                Title = "Validation Error",
                Extensions = { ["errors"] = ex.Errors }
            });
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = 404,
                Title = "Not Found",
                Detail = ex.Message
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = 500,
                Title = "Internal Server Error"
            });
        }
    }
}
```

## Options Pattern for Configuration

```csharp
// Bind strongly-typed configuration
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    public string Host { get; init; } = string.Empty;
    public int Port { get; init; }
    public string Username { get; init; } = string.Empty;
}

// Registration
services.Configure<SmtpSettings>(configuration.GetSection(SmtpSettings.SectionName));

// Usage — inject IOptions<T>, never IConfiguration
public class EmailService
{
    private readonly SmtpSettings _settings;
    public EmailService(IOptions<SmtpSettings> options)
        => _settings = options.Value;
}
```

## Additional Resources

### Reference Files

For detailed patterns and structural guidance, consult:

- **`references/design_patterns.md`** — Repository, Unit of Work, Specification, Result pattern, domain events
- **`references/project_structure.md`** — Layer responsibilities, project references, naming conventions, folder organization
