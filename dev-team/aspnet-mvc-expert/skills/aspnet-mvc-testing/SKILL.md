---
name: aspnet-mvc-testing
description: This skill should be used when the user asks to "write unit tests", "add integration tests", "test a controller", "test a service", "mock dependencies", "set up WebApplicationFactory", "test EF Core queries", "add test coverage", "create test fixtures", or needs guidance on testing ASP.NET Core MVC applications with xUnit, Moq, or integration testing patterns.
---

# ASP.NET MVC Testing

Expert guidance for testing ASP.NET Core MVC applications using xUnit, Moq, and integration testing with WebApplicationFactory.

## Test Project Structure

```
tests/
├── Domain.UnitTests/
│   ├── Entities/
│   │   └── ProductTests.cs
│   └── ValueObjects/
│       └── MoneyTests.cs
├── Application.UnitTests/
│   ├── Features/
│   │   └── Products/
│   │       ├── CreateProductHandlerTests.cs
│   │       └── GetProductQueryHandlerTests.cs
│   └── Common/
│       └── ValidationBehaviorTests.cs
├── Infrastructure.IntegrationTests/
│   ├── Persistence/
│   │   └── ProductRepositoryTests.cs
│   └── Services/
│       └── EmailServiceTests.cs
└── WebUI.IntegrationTests/
    ├── Controllers/
    │   └── ProductsControllerTests.cs
    ├── CustomWebApplicationFactory.cs
    └── IntegrationTestBase.cs
```

## Unit Testing with xUnit + Moq

### Testing MediatR Handlers (Arrange-Act-Assert)

```csharp
public class CreateProductHandlerTests
{
    private readonly Mock<IProductRepository> _repositoryMock;
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly CreateProductHandler _handler;

    public CreateProductHandlerTests()
    {
        _repositoryMock = new Mock<IProductRepository>();
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _handler = new CreateProductHandler(
            _repositoryMock.Object,
            _unitOfWorkMock.Object);
    }

    [Fact]
    public async Task Handle_ValidCommand_CreatesProductAndReturnsId()
    {
        // Arrange
        var command = new CreateProductCommand("Widget", 29.99m);
        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        _repositoryMock.Verify(
            r => r.Add(It.Is<Product>(p =>
                p.Name == "Widget" && p.Price == 29.99m)),
            Times.Once);
        _unitOfWorkMock.Verify(
            u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
            Times.Once);
        Assert.True(result > 0);
    }

    [Fact]
    public async Task Handle_EmptyName_ThrowsValidationException()
    {
        // Arrange
        var command = new CreateProductCommand("", 29.99m);

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(
            () => _handler.Handle(command, CancellationToken.None));
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100.50)]
    public async Task Handle_InvalidPrice_ThrowsValidationException(decimal price)
    {
        // Arrange
        var command = new CreateProductCommand("Widget", price);

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(
            () => _handler.Handle(command, CancellationToken.None));
    }
}
```

### Testing Domain Entities

```csharp
public class OrderTests
{
    [Fact]
    public void AddItem_ValidProduct_IncreasesItemCount()
    {
        // Arrange
        var order = new Order(customerId: 1);
        var product = new Product("Widget", 10.00m);

        // Act
        order.AddItem(product, quantity: 3);

        // Assert
        Assert.Single(order.Items);
        Assert.Equal(30.00m, order.Total);
    }

    [Fact]
    public void AddItem_ZeroQuantity_ThrowsDomainException()
    {
        // Arrange
        var order = new Order(customerId: 1);
        var product = new Product("Widget", 10.00m);

        // Act & Assert
        Assert.Throws<DomainException>(() => order.AddItem(product, quantity: 0));
    }

    [Fact]
    public void Cancel_AlreadyShipped_ThrowsDomainException()
    {
        // Arrange
        var order = OrderBuilder.Create()
            .WithStatus(OrderStatus.Shipped)
            .Build();

        // Act & Assert
        var ex = Assert.Throws<DomainException>(() => order.Cancel());
        Assert.Contains("shipped", ex.Message, StringComparison.OrdinalIgnoreCase);
    }
}
```

### Testing FluentValidation Validators

```csharp
public class CreateProductCommandValidatorTests
{
    private readonly CreateProductCommandValidator _validator = new();

    [Fact]
    public void Validate_ValidCommand_ReturnsNoErrors()
    {
        var command = new CreateProductCommand("Widget", 29.99m);
        var result = _validator.Validate(command);
        Assert.True(result.IsValid);
    }

    [Theory]
    [InlineData("")]
    [InlineData(null)]
    public void Validate_EmptyName_ReturnsError(string? name)
    {
        var command = new CreateProductCommand(name!, 29.99m);
        var result = _validator.Validate(command);
        Assert.Contains(result.Errors,
            e => e.PropertyName == nameof(CreateProductCommand.Name));
    }
}
```

## Integration Testing with WebApplicationFactory

### Custom Factory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real database
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null) services.Remove(descriptor);

            // Add in-memory database
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("TestDb_" + Guid.NewGuid()));

            // Seed test data
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
            SeedTestData(db);
        });

        builder.ConfigureTestServices(services =>
        {
            // Replace external services with fakes
            services.AddSingleton<IEmailService, FakeEmailService>();
        });
    }

    private static void SeedTestData(AppDbContext db)
    {
        db.Products.AddRange(
            new Product("Widget A", 10.00m),
            new Product("Widget B", 20.00m));
        db.SaveChanges();
    }
}
```

### Integration Test Base Class

```csharp
public abstract class IntegrationTestBase
    : IClassFixture<CustomWebApplicationFactory>, IDisposable
{
    protected readonly HttpClient Client;
    protected readonly CustomWebApplicationFactory Factory;

    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
    }

    protected async Task AuthenticateAsync(string role = "User")
    {
        // Generate test JWT token
        var token = TestTokenGenerator.Generate(role);
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
    }

    protected async Task<T?> DeserializeResponse<T>(HttpResponseMessage response)
    {
        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<T>(content,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
    }

    public void Dispose() => Client.Dispose();
}
```

### Controller Integration Tests

```csharp
public class ProductsControllerTests : IntegrationTestBase
{
    public ProductsControllerTests(CustomWebApplicationFactory factory)
        : base(factory) { }

    [Fact]
    public async Task GetAll_ReturnsOkWithProducts()
    {
        // Arrange
        await AuthenticateAsync();

        // Act
        var response = await Client.GetAsync("/api/products");

        // Assert
        response.EnsureSuccessStatusCode();
        var products = await DeserializeResponse<List<ProductDto>>(response);
        Assert.NotNull(products);
        Assert.NotEmpty(products);
    }

    [Fact]
    public async Task Create_ValidProduct_ReturnsCreated()
    {
        // Arrange
        await AuthenticateAsync("Admin");
        var command = new { Name = "New Widget", Price = 15.00m };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }

    [Fact]
    public async Task Create_Unauthorized_Returns401()
    {
        // Arrange — no authentication
        var command = new { Name = "Widget", Price = 15.00m };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task Create_InvalidData_Returns400WithValidationErrors()
    {
        // Arrange
        await AuthenticateAsync("Admin");
        var command = new { Name = "", Price = -1m };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", command);

        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        var problem = await DeserializeResponse<ProblemDetails>(response);
        Assert.NotNull(problem);
    }
}
```

## Test Data Builders

```csharp
public class ProductBuilder
{
    private string _name = "Default Product";
    private decimal _price = 9.99m;
    private int _categoryId = 1;
    private bool _isActive = true;

    public static ProductBuilder Create() => new();

    public ProductBuilder WithName(string name) { _name = name; return this; }
    public ProductBuilder WithPrice(decimal price) { _price = price; return this; }
    public ProductBuilder WithCategory(int id) { _categoryId = id; return this; }
    public ProductBuilder Inactive() { _isActive = false; return this; }

    public Product Build()
    {
        var product = new Product(_name, _price) { CategoryId = _categoryId };
        if (!_isActive) product.Deactivate();
        return product;
    }
}

// Usage in tests
var product = ProductBuilder.Create()
    .WithName("Premium Widget")
    .WithPrice(99.99m)
    .Build();
```

## Testing EF Core with In-Memory Database

```csharp
public class ProductRepositoryTests : IDisposable
{
    private readonly AppDbContext _db;
    private readonly ProductRepository _repository;

    public ProductRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;
        _db = new AppDbContext(options);
        _repository = new ProductRepository(_db);
    }

    [Fact]
    public async Task GetByCategoryAsync_ReturnsMatchingProducts()
    {
        // Arrange
        _db.Products.AddRange(
            new Product("A", 10m) { CategoryId = 1 },
            new Product("B", 20m) { CategoryId = 1 },
            new Product("C", 30m) { CategoryId = 2 });
        await _db.SaveChangesAsync();

        // Act
        var result = await _repository.GetByCategoryAsync(1, CancellationToken.None);

        // Assert
        Assert.Equal(2, result.Count);
        Assert.All(result, p => Assert.Equal(1, p.CategoryId));
    }

    public void Dispose() => _db.Dispose();
}
```

## Testing Best Practices

| Practice | Guideline |
|----------|-----------|
| Naming | `MethodUnderTest_Scenario_ExpectedResult` |
| Structure | Arrange-Act-Assert (AAA) in every test |
| Independence | No shared mutable state between tests |
| Mocking | Mock at boundaries, not internal classes |
| Assertions | One logical assertion per test |
| Data | Use builders for complex test data |
| Async | Always use `async Task`, never `async void` |
| Cleanup | Implement `IDisposable` for resource cleanup |

## Additional Resources

### Reference Files

For advanced testing patterns, consult:

- **`references/unit_testing.md`** — Advanced mocking patterns, testing domain events, testing middleware, snapshot testing, parameterized test strategies
- **`references/integration_testing.md`** — Database testing with Testcontainers, authentication test helpers, API contract testing, performance testing patterns
