# ASP.NET Core Patterns

## Minimal APIs vs Controllers

### When to Use Minimal APIs

Minimal APIs are best for small to medium services, microservices, and feature-based architectures where the overhead of controllers is unnecessary.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Simple route group
var products = app.MapGroup("/api/products")
    .WithTags("Products")
    .RequireAuthorization();

products.MapGet("/", async (
    [AsParameters] ProductQueryParams queryParams,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(new ListProductsQuery(queryParams), ct);
    return Results.Ok(result);
});

products.MapGet("/{id:int}", async (
    int id,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(new GetProductByIdQuery(id), ct);
    return result is not null ? Results.Ok(result) : Results.NotFound();
})
.WithName("GetProductById")
.Produces<ProductDto>()
.ProducesProblem(StatusCodes.Status404NotFound);

products.MapPost("/", async (
    CreateProductCommand command,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(command, ct);
    return result.ToHttpResult();
})
.AddEndpointFilter<ValidationFilter<CreateProductCommand>>();

// Parameter binding with AsParameters
public record ProductQueryParams(
    string? Search,
    string? Category,
    decimal? MinPrice,
    decimal? MaxPrice,
    int Page = 1,
    int PageSize = 20);
```

### When to Use Controllers

Controllers are better for large APIs with shared filters, complex model binding, or when the team prefers the MVC pattern.

```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class OrdersController(IMediator mediator) : ControllerBase
{
    /// <summary>
    /// Get an order by ID
    /// </summary>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var order = await mediator.Send(new GetOrderByIdQuery(id), ct);
        return order is not null ? Ok(order) : NotFound();
    }

    /// <summary>
    /// Create a new order
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken ct)
    {
        var result = await mediator.Send(command, ct);
        return result.Match<IActionResult>(
            order => CreatedAtAction(nameof(GetById), new { id = order.Id }, order),
            error => BadRequest(new { error }));
    }

    /// <summary>
    /// Cancel an order
    /// </summary>
    [HttpPost("{id:int}/cancel")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Cancel(int id, [FromBody] CancelOrderRequest request, CancellationToken ct)
    {
        var result = await mediator.Send(new CancelOrderCommand(id, request.Reason), ct);
        return result.Match<IActionResult>(
            _ => NoContent(),
            error => NotFound(new { error }));
    }
}
```

### Endpoint Filters (Minimal API Middleware)

```csharp
// Validation filter for minimal APIs
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var argument = context.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null)
            return Results.BadRequest("Invalid request body");

        var validationResult = await validator.ValidateAsync(argument);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary());
        }

        return await next(context);
    }
}
```

## Middleware Pipeline

### Custom Middleware

```csharp
// Correlation ID middleware — adds a unique ID to every request for tracing
public class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}

// Exception handling middleware with ProblemDetails
public class GlobalExceptionMiddleware(
    RequestDelegate next,
    ILogger<GlobalExceptionMiddleware> logger,
    IHostEnvironment env)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new ValidationProblemDetails(
                ex.Errors.GroupBy(e => e.PropertyName)
                    .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray()))
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Validation failed"
            });
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Resource not found",
                Detail = ex.Message
            });
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception processing {Method} {Path}",
                context.Request.Method, context.Request.Path);

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "An error occurred",
                Detail = env.IsDevelopment() ? ex.Message : "An unexpected error occurred"
            });
        }
    }
}

// Registration order
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<GlobalExceptionMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
```

## Options Pattern

Bind configuration sections to strongly-typed objects with validation.

```csharp
// Settings class with validation attributes
public class DatabaseSettings
{
    public const string SectionName = "Database";

    [Required]
    public required string ConnectionString { get; init; }

    [Range(1, 100)]
    public int MaxRetryCount { get; init; } = 3;

    [Range(1, 300)]
    public int CommandTimeoutSeconds { get; init; } = 30;

    public bool EnableSensitiveDataLogging { get; init; }
}

public class CacheSettings
{
    public const string SectionName = "Cache";

    [Required]
    public required string RedisConnectionString { get; init; }

    [Range(1, 3600)]
    public int DefaultExpirationSeconds { get; init; } = 300;

    public string InstanceName { get; init; } = "myapp:";
}

// Registration with validation
builder.Services
    .AddOptions<DatabaseSettings>()
    .BindConfiguration(DatabaseSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services
    .AddOptions<CacheSettings>()
    .BindConfiguration(CacheSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Consumption — use IOptions<T> for singleton, IOptionsSnapshot<T> for scoped
public class ProductService(
    IOptions<CacheSettings> cacheOptions,
    IOptionsMonitor<DatabaseSettings> dbOptions)
{
    // IOptions<T> — read once at startup, never changes
    private readonly CacheSettings _cacheSettings = cacheOptions.Value;

    // IOptionsMonitor<T> — reflects changes when configuration is reloaded
    public int CurrentTimeout => dbOptions.CurrentValue.CommandTimeoutSeconds;
}

// Named options for multiple instances
builder.Services.Configure<EmailSettings>("Transactional",
    builder.Configuration.GetSection("Email:Transactional"));
builder.Services.Configure<EmailSettings>("Marketing",
    builder.Configuration.GetSection("Email:Marketing"));

// Consume named options
public class EmailService(IOptionsSnapshot<EmailSettings> options)
{
    public Task SendTransactionalAsync(string to, string subject, string body)
    {
        var settings = options.Get("Transactional");
        // ...
    }
}
```

## Health Checks

Expose health endpoints for load balancers and monitoring systems.

```csharp
// Register health checks
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "sqlserver",
        tags: ["db", "ready"])
    .AddRedis(
        redisConnectionString: builder.Configuration["Cache:RedisConnectionString"]!,
        name: "redis",
        tags: ["cache", "ready"])
    .AddCheck<ExternalApiHealthCheck>(
        name: "external-api",
        tags: ["external", "ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

// Custom health check
public class ExternalApiHealthCheck(
    IHttpClientFactory httpClientFactory) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            var client = httpClientFactory.CreateClient("ExternalApi");
            var response = await client.GetAsync("/health", ct);

            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("External API is reachable")
                : HealthCheckResult.Degraded($"External API returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External API is unreachable", ex);
        }
    }
}

// Map health endpoints with filtering
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

## Background Services

Use `BackgroundService` for long-running tasks, periodic jobs, and message consumers.

```csharp
// Periodic background task
public class OrderCleanupService(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderCleanupService> logger) : BackgroundService
{
    private readonly PeriodicTimer _timer = new(TimeSpan.FromHours(1));

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Order cleanup service starting");

        while (await _timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

                var cutoff = DateTime.UtcNow.AddDays(-90);
                var deleted = await dbContext.Orders
                    .Where(o => o.Status == OrderStatus.Cancelled && o.CreatedAt < cutoff)
                    .ExecuteDeleteAsync(stoppingToken);

                logger.LogInformation("Cleaned up {Count} cancelled orders older than {Cutoff}",
                    deleted, cutoff);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Error during order cleanup");
            }
        }
    }
}

// Queue-based background processor
public class OrderProcessingWorker(
    IServiceScopeFactory scopeFactory,
    Channel<int> orderChannel,
    ILogger<OrderProcessingWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var orderId in orderChannel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var mediator = scope.ServiceProvider.GetRequiredService<IMediator>();
                await mediator.Send(new ProcessOrderCommand(orderId), stoppingToken);

                logger.LogInformation("Processed order {OrderId}", orderId);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Failed to process order {OrderId}", orderId);
            }
        }
    }
}

// Registration
builder.Services.AddSingleton(Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
}));
builder.Services.AddHostedService<OrderCleanupService>();
builder.Services.AddHostedService<OrderProcessingWorker>();
```

## Output Caching (.NET 7+)

Cache entire HTTP responses at the server level.

```csharp
builder.Services.AddOutputCache(options =>
{
    // Default policy: cache for 60 seconds
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromSeconds(60)));

    // Named policy for product listings
    options.AddPolicy("ProductList", builder =>
        builder.Expire(TimeSpan.FromMinutes(5))
               .Tag("products")
               .SetVaryByQuery("category", "page", "pageSize"));

    // Policy that varies by authenticated user
    options.AddPolicy("UserDashboard", builder =>
        builder.Expire(TimeSpan.FromMinutes(2))
               .SetVaryByHeader("Authorization"));
});

var app = builder.Build();
app.UseOutputCache();

// Apply to endpoints
app.MapGet("/api/products", async (IMediator mediator, CancellationToken ct) =>
    Results.Ok(await mediator.Send(new ListProductsQuery(), ct)))
    .CacheOutput("ProductList");

// Invalidate cache by tag
app.MapPost("/api/products", async (
    CreateProductCommand command,
    IMediator mediator,
    IOutputCacheStore cacheStore,
    CancellationToken ct) =>
{
    var result = await mediator.Send(command, ct);
    await cacheStore.EvictByTagAsync("products", ct); // Invalidate product cache
    return result.ToHttpResult();
});
```

## Rate Limiting (.NET 7+)

Protect APIs from abuse with built-in rate limiting.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Fixed window: 100 requests per minute
    options.AddFixedWindowLimiter("fixed", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiter.QueueLimit = 10;
    });

    // Sliding window: smoother rate limiting
    options.AddSlidingWindowLimiter("sliding", limiter =>
    {
        limiter.PermitLimit = 60;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.SegmentsPerWindow = 6; // 10-second segments
    });

    // Token bucket: allows bursts
    options.AddTokenBucketLimiter("token", limiter =>
    {
        limiter.TokenLimit = 20;
        limiter.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        limiter.TokensPerPeriod = 5;
    });

    // Per-user rate limiting
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User?.FindFirstValue(ClaimTypes.NameIdentifier) ?? "anonymous";
        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 50,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    // Custom response for rate-limited requests
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
        }

        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title = "Too many requests",
            Detail = "Rate limit exceeded. Please try again later."
        }, ct);
    };
});

app.UseRateLimiter();

// Apply to route groups
var api = app.MapGroup("/api").RequireRateLimiting("per-user");

// Apply to individual endpoints
app.MapPost("/api/auth/login", HandleLogin)
    .RequireRateLimiting("fixed");
```

## HttpClient Factory

Manage HTTP client instances properly to avoid socket exhaustion.

```csharp
// Named client
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
})
.AddStandardResilienceHandler(); // Polly retry, circuit breaker, timeout

// Typed client
public class GitHubClient(HttpClient httpClient)
{
    public async Task<GitHubUser?> GetUserAsync(string username, CancellationToken ct) =>
        await httpClient.GetFromJsonAsync<GitHubUser>($"/users/{username}", ct);
}

builder.Services.AddHttpClient<GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
})
.AddStandardResilienceHandler();

// Resilience with Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient("ExternalApi")
    .AddResilienceHandler("pipeline", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(500),
            BackoffType = DelayBackoffType.Exponential
        });

        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(15)
        });

        builder.AddTimeout(TimeSpan.FromSeconds(5));
    });
```
