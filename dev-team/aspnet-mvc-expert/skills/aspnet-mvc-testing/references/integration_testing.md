# Integration Testing Patterns for ASP.NET Core

Advanced integration testing patterns using WebApplicationFactory, Testcontainers, and API contract testing.

## WebApplicationFactory Setup

### Custom Factory with Service Replacement

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    public Mock<IEmailService> MockEmailService { get; } = new();
    public Mock<IDateTime> MockDateTime { get; } = new();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        builder.ConfigureServices(services =>
        {
            // Replace real database with in-memory
            RemoveService<DbContextOptions<AppDbContext>>(services);
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase($"TestDb_{Guid.NewGuid()}"));

            // Replace external services with mocks
            RemoveService<IEmailService>(services);
            services.AddSingleton(MockEmailService.Object);

            RemoveService<IDateTime>(services);
            MockDateTime.Setup(d => d.UtcNow).Returns(new DateTime(2025, 1, 15, 12, 0, 0));
            services.AddSingleton(MockDateTime.Object);
        });

        builder.ConfigureTestServices(services =>
        {
            // Additional test-only services
            services.AddSingleton<TestTokenGenerator>();
        });
    }

    public async Task SeedDataAsync(Func<AppDbContext, Task> seedAction)
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await seedAction(db);
    }

    private static void RemoveService<T>(IServiceCollection services)
    {
        var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(T));
        if (descriptor is not null) services.Remove(descriptor);
    }
}
```

### Base Test Class with Common Utilities

```csharp
public abstract class IntegrationTestBase
    : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    protected readonly CustomWebApplicationFactory Factory;
    protected readonly HttpClient Client;

    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
    }

    // Authentication helpers
    protected void AuthenticateAs(string userId, string role = "User")
    {
        var token = TestTokenGenerator.Generate(userId, role);
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
    }

    protected void AuthenticateAsAdmin()
        => AuthenticateAs("admin-user-id", "Admin");

    // Request helpers
    protected async Task<T?> GetAsync<T>(string url)
    {
        var response = await Client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>();
    }

    protected async Task<HttpResponseMessage> PostAsync<T>(string url, T body)
        => await Client.PostAsJsonAsync(url, body);

    // Database helpers
    protected async Task<T> WithDbContextAsync<T>(Func<AppDbContext, Task<T>> action)
    {
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await action(db);
    }

    // Lifecycle
    public virtual Task InitializeAsync() => Task.CompletedTask;
    public virtual Task DisposeAsync() => Task.CompletedTask;
}
```

## Testing with Testcontainers (Real Database)

For tests that require real SQL Server behavior:

```csharp
public class SqlServerFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .WithPassword("YourStrong!Password123")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}

public class DatabaseIntegrationTests
    : IClassFixture<SqlServerFixture>, IAsyncLifetime
{
    private readonly SqlServerFixture _fixture;
    private AppDbContext _db = null!;

    public DatabaseIntegrationTests(SqlServerFixture fixture)
        => _fixture = fixture;

    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_fixture.ConnectionString)
            .Options;

        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync();
    }

    [Fact]
    public async Task Product_GlobalFilter_ExcludesDeletedItems()
    {
        // Arrange
        _db.Products.Add(new Product("Active", 10m));
        var deleted = new Product("Deleted", 20m) { IsDeleted = true };
        _db.Products.Add(deleted);
        await _db.SaveChangesAsync();

        // Act
        var products = await _db.Products.ToListAsync();

        // Assert — global filter applied
        Assert.Single(products);
        Assert.Equal("Active", products[0].Name);

        // Verify deleted product still exists
        var allProducts = await _db.Products.IgnoreQueryFilters().ToListAsync();
        Assert.Equal(2, allProducts.Count);
    }

    public Task DisposeAsync()
    {
        _db.Dispose();
        return Task.CompletedTask;
    }
}
```

## Authentication Testing Helpers

### Test Token Generator

```csharp
public static class TestTokenGenerator
{
    private const string TestSecret = "ThisIsATestSecretKeyThatIsLongEnoughForHmacSha256!!";
    private const string TestIssuer = "TestIssuer";
    private const string TestAudience = "TestAudience";

    public static string Generate(
        string userId = "test-user",
        string role = "User",
        Dictionary<string, string>? additionalClaims = null)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId),
            new(ClaimTypes.Email, $"{userId}@test.com"),
            new(ClaimTypes.Role, role),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };

        if (additionalClaims is not null)
        {
            claims.AddRange(additionalClaims.Select(
                kvp => new Claim(kvp.Key, kvp.Value)));
        }

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(TestSecret));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: TestIssuer,
            audience: TestAudience,
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    // Configure factory to accept test tokens
    public static void ConfigureTestAuthentication(IServiceCollection services)
    {
        services.PostConfigure<JwtBearerOptions>(
            JwtBearerDefaults.AuthenticationScheme,
            options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = TestIssuer,
                    ValidAudience = TestAudience,
                    IssuerSigningKey = new SymmetricSecurityKey(
                        Encoding.UTF8.GetBytes(TestSecret))
                };
            });
    }
}
```

## Testing Authorization

```csharp
public class AuthorizationTests : IntegrationTestBase
{
    public AuthorizationTests(CustomWebApplicationFactory factory) : base(factory) { }

    [Fact]
    public async Task CreateProduct_AsAdmin_ReturnsCreated()
    {
        AuthenticateAsAdmin();
        var command = new { Name = "Widget", Price = 10.0m };

        var response = await PostAsync("/api/products", command);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }

    [Fact]
    public async Task CreateProduct_AsRegularUser_ReturnsForbidden()
    {
        AuthenticateAs("user-1", "User");
        var command = new { Name = "Widget", Price = 10.0m };

        var response = await PostAsync("/api/products", command);

        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }

    [Fact]
    public async Task CreateProduct_Unauthenticated_ReturnsUnauthorized()
    {
        // No authentication header set
        var command = new { Name = "Widget", Price = 10.0m };

        var response = await PostAsync("/api/products", command);

        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task UpdateOrder_OwnOrder_ReturnsOk()
    {
        AuthenticateAs("user-1");
        await Factory.SeedDataAsync(async db =>
        {
            db.Orders.Add(new Order { Id = 1, UserId = "user-1" });
            await db.SaveChangesAsync();
        });

        var response = await Client.PutAsJsonAsync("/api/orders/1",
            new { Status = "Processing" });

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task UpdateOrder_OtherUserOrder_ReturnsForbidden()
    {
        AuthenticateAs("user-2");
        await Factory.SeedDataAsync(async db =>
        {
            db.Orders.Add(new Order { Id = 2, UserId = "user-1" }); // Different user
            await db.SaveChangesAsync();
        });

        var response = await Client.PutAsJsonAsync("/api/orders/2",
            new { Status = "Processing" });

        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
}
```

## API Contract Testing

Verify API response shapes match expected contracts:

```csharp
public class ProductApiContractTests : IntegrationTestBase
{
    public ProductApiContractTests(CustomWebApplicationFactory factory)
        : base(factory) { }

    [Fact]
    public async Task GetProduct_ResponseMatchesContract()
    {
        // Arrange
        AuthenticateAs("test-user");
        await Factory.SeedDataAsync(async db =>
        {
            db.Products.Add(new Product("Widget", 29.99m) { Id = 1 });
            await db.SaveChangesAsync();
        });

        // Act
        var response = await Client.GetAsync("/api/products/1");
        var json = await response.Content.ReadAsStringAsync();
        var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        // Assert — verify JSON structure
        Assert.True(root.TryGetProperty("id", out var id));
        Assert.Equal(JsonValueKind.Number, id.ValueKind);

        Assert.True(root.TryGetProperty("name", out var name));
        Assert.Equal(JsonValueKind.String, name.ValueKind);

        Assert.True(root.TryGetProperty("price", out var price));
        Assert.Equal(JsonValueKind.Number, price.ValueKind);
    }

    [Fact]
    public async Task GetProducts_PaginatedResponse_MatchesContract()
    {
        AuthenticateAs("test-user");
        var response = await Client.GetAsync("/api/products?pageNumber=1&pageSize=10");
        var json = await response.Content.ReadAsStringAsync();
        var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        // Verify pagination envelope
        Assert.True(root.TryGetProperty("items", out _));
        Assert.True(root.TryGetProperty("pageNumber", out _));
        Assert.True(root.TryGetProperty("totalPages", out _));
        Assert.True(root.TryGetProperty("totalCount", out _));
        Assert.True(root.TryGetProperty("hasPreviousPage", out _));
        Assert.True(root.TryGetProperty("hasNextPage", out _));
    }
}
```

## Performance Testing with BenchmarkDotNet

```csharp
[MemoryDiagnoser]
public class RepositoryBenchmarks
{
    private AppDbContext _db = null!;
    private ProductRepository _repository = null!;

    [GlobalSetup]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase("BenchmarkDb")
            .Options;
        _db = new AppDbContext(options);
        _repository = new ProductRepository(_db);

        // Seed 10,000 products
        for (int i = 0; i < 10_000; i++)
            _db.Products.Add(new Product($"Product {i}", i * 1.5m));
        _db.SaveChanges();
    }

    [Benchmark]
    public async Task GetAllProducts_WithTracking()
        => await _db.Products.ToListAsync();

    [Benchmark]
    public async Task GetAllProducts_NoTracking()
        => await _db.Products.AsNoTracking().ToListAsync();

    [Benchmark]
    public async Task GetAllProducts_Projected()
        => await _db.Products
            .Select(p => new { p.Id, p.Name, p.Price })
            .ToListAsync();
}
```

## Test Collection Organization

```csharp
// Share expensive resources across test classes
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<SqlServerFixture> { }

[Collection("Database")]
public class ProductRepositoryTests
{
    private readonly SqlServerFixture _fixture;
    public ProductRepositoryTests(SqlServerFixture fixture) => _fixture = fixture;
}

[Collection("Database")]
public class OrderRepositoryTests
{
    private readonly SqlServerFixture _fixture;
    public OrderRepositoryTests(SqlServerFixture fixture) => _fixture = fixture;
}
```
