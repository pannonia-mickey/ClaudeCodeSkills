# ASP.NET Core Patterns

## Minimal APIs vs Controllers

| Factor | Minimal APIs | Controllers |
|--------|-------------|-------------|
| Boilerplate | Low | Higher |
| Features | Filters, grouping, binding | Full MVC pipeline, views |
| Best for | Microservices, simple APIs | Large APIs, MVC apps |
| Testing | Direct handler testing | WebApplicationFactory |

## Middleware Pipeline Ordering

Order matters — each middleware wraps the next:

```csharp
app.UseExceptionHandler();          // 1. Outermost: catch all
app.UseHsts();                      // 2. HTTP Strict Transport Security
app.UseHttpsRedirection();          // 3. Redirect HTTP → HTTPS
app.UseStaticFiles();               // 4. Serve static files (short-circuit)
app.UseRouting();                   // 5. Match endpoint
app.UseCors();                      // 6. CORS headers
app.UseAuthentication();            // 7. Identify user
app.UseAuthorization();             // 8. Check permissions
app.UseRateLimiter();               // 9. Throttle
app.UseOutputCache();               // 10. Cache responses
app.MapControllers();               // 11. Terminal: execute endpoint
```

## Options Pattern

```csharp
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    public string Host { get; init; } = "";
    public int Port { get; init; } = 587;
}

// IOptions<T> — singleton, read once at startup
// IOptionsSnapshot<T> — scoped, re-reads per request
// IOptionsMonitor<T> — singleton, notifies on change
services.Configure<SmtpSettings>(config.GetSection(SmtpSettings.SectionName));

public class EmailService(IOptions<SmtpSettings> options)
{
    private readonly SmtpSettings _settings = options.Value;
}
```

## Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!, name: "redis")
    .AddUrlGroup(new Uri("https://api.external.com/health"), name: "external-api");

app.MapHealthChecks("/health/live", new() { Predicate = _ => false }); // Liveness
app.MapHealthChecks("/health/ready", new() { ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse }); // Readiness
```

## Background Services

```csharp
public class OrderCleanupService(IServiceScopeFactory scopeFactory, ILogger<OrderCleanupService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var expired = await db.Orders.Where(o => o.Status == OrderStatus.Draft && o.CreatedAt < DateTime.UtcNow.AddDays(-7)).ToListAsync(ct);
            db.Orders.RemoveRange(expired);
            await db.SaveChangesAsync(ct);
            logger.LogInformation("Cleaned up {Count} expired draft orders", expired.Count);
            await Task.Delay(TimeSpan.FromHours(1), ct);
        }
    }
}
```

## Output Caching

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromMinutes(5)));
    options.AddPolicy("Products", builder => builder.Expire(TimeSpan.FromMinutes(10)).Tag("products"));
});

app.MapGet("/api/products", GetProducts).CacheOutput("Products");

// Invalidate
app.MapPost("/api/products", async (IOutputCacheStore cache, ...) =>
{
    // ... create product
    await cache.EvictByTagAsync("products", ct);
});
```

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", o => { o.PermitLimit = 100; o.Window = TimeSpan.FromMinutes(1); });
    options.AddSlidingWindowLimiter("auth", o => { o.PermitLimit = 5; o.Window = TimeSpan.FromMinutes(1); o.SegmentsPerWindow = 3; });
    options.AddTokenBucketLimiter("upload", o => { o.TokenLimit = 10; o.ReplenishmentPeriod = TimeSpan.FromSeconds(10); o.TokensPerPeriod = 2; });
    options.AddConcurrencyLimiter("webhook", o => { o.PermitLimit = 5; });
});
```
