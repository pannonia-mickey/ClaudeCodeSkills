---
name: C# Testing
description: This skill should be used when the user asks about "xUnit", "NUnit", "C# test", "Moq", "FluentAssertions", or "WebApplicationFactory". It covers unit tests with xUnit/NUnit, mocking with Moq, FluentAssertions, integration testing with WebApplicationFactory, test design, fixtures, and testing EF Core.
---

# C# Testing

## Unit Test Structure

Follow the Arrange-Act-Assert pattern consistently. Name tests clearly to describe the scenario and expected outcome.

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repositoryMock = new();
    private readonly Mock<IInventoryService> _inventoryMock = new();
    private readonly Mock<ILogger<OrderService>> _loggerMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(
            _repositoryMock.Object,
            _inventoryMock.Object,
            _loggerMock.Object);
    }

    [Fact]
    public async Task CreateOrder_WithSufficientStock_ReturnsSuccess()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            Items = [new OrderItemRequest { ProductId = 10, Quantity = 2 }]
        };

        _inventoryMock
            .Setup(x => x.CheckStockAsync(10, It.IsAny<CancellationToken>()))
            .ReturnsAsync(50);

        // Act
        var result = await _sut.CreateOrderAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
        result.Value!.CustomerId.Should().Be(1);

        _repositoryMock.Verify(
            x => x.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);

        _repositoryMock.Verify(
            x => x.SaveChangesAsync(It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task CreateOrder_WithInsufficientStock_ReturnsFailure()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            Items = [new OrderItemRequest { ProductId = 10, Quantity = 100 }]
        };

        _inventoryMock
            .Setup(x => x.CheckStockAsync(10, It.IsAny<CancellationToken>()))
            .ReturnsAsync(5); // Only 5 available

        // Act
        var result = await _sut.CreateOrderAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("Insufficient stock");

        _repositoryMock.Verify(
            x => x.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Never);
    }
}
```

## Data-Driven Tests

### xUnit Theory with InlineData

```csharp
public class PriceCalculatorTests
{
    [Theory]
    [InlineData(100, 0.1, 90)]
    [InlineData(200, 0.2, 160)]
    [InlineData(50, 0, 50)]
    [InlineData(100, 1.0, 0)]
    public void ApplyDiscount_ReturnsCorrectPrice(
        decimal originalPrice,
        decimal discountRate,
        decimal expectedPrice)
    {
        var result = PriceCalculator.ApplyDiscount(originalPrice, discountRate);
        result.Should().Be(expectedPrice);
    }
}
```

### xUnit Theory with MemberData

```csharp
public class OrderValidatorTests
{
    public static TheoryData<CreateOrderRequest, bool> ValidationCases => new()
    {
        {
            new CreateOrderRequest { CustomerId = 1, Items = [new() { ProductId = 1, Quantity = 1 }] },
            true
        },
        {
            new CreateOrderRequest { CustomerId = 0, Items = [new() { ProductId = 1, Quantity = 1 }] },
            false
        },
        {
            new CreateOrderRequest { CustomerId = 1, Items = [] },
            false
        }
    };

    [Theory]
    [MemberData(nameof(ValidationCases))]
    public void Validate_ReturnsExpectedResult(CreateOrderRequest request, bool expectedIsValid)
    {
        var validator = new CreateOrderRequestValidator();
        var result = validator.Validate(request);
        result.IsValid.Should().Be(expectedIsValid);
    }
}
```

### xUnit Theory with ClassData

```csharp
public class DiscountTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return [new Order { Total = 50 }, 0m];
        yield return [new Order { Total = 150 }, 0.05m];
        yield return [new Order { Total = 500 }, 0.10m];
        yield return [new Order { Total = 1500 }, 0.20m];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(DiscountTestData))]
public void GetDiscountRate_ReturnsCorrectRate(Order order, decimal expectedRate)
{
    var rate = _calculator.GetDiscountRate(order);
    rate.Should().Be(expectedRate);
}
```

## Mocking Patterns

### Mock Setup and Verification

```csharp
// Setup return values
_repositoryMock
    .Setup(x => x.GetByIdAsync(42, It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Order { Id = 42, Total = 100 });

// Setup sequences (different results on successive calls)
_serviceMock
    .SetupSequence(x => x.ProcessAsync(It.IsAny<Request>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(Result.Success())
    .ReturnsAsync(Result.Success())
    .ThrowsAsync(new TimeoutException());

// Setup with argument matching
_emailMock
    .Setup(x => x.SendAsync(
        It.Is<string>(email => email.Contains("@")),
        It.IsAny<string>(),
        It.IsAny<string>(),
        It.IsAny<CancellationToken>()))
    .Returns(Task.CompletedTask);

// Setup callbacks for inspection
var capturedOrders = new List<Order>();
_repositoryMock
    .Setup(x => x.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
    .Callback<Order, CancellationToken>((order, _) => capturedOrders.Add(order))
    .Returns(Task.CompletedTask);

// Verify call count
_repositoryMock.Verify(
    x => x.SaveChangesAsync(It.IsAny<CancellationToken>()),
    Times.Exactly(2));

// Verify no other calls were made
_repositoryMock.VerifyNoOtherCalls();
```

### Mocking HttpClient

```csharp
public class ExternalApiServiceTests
{
    [Fact]
    public async Task GetData_ReturnsDeserializedResponse()
    {
        // Arrange
        var handler = new Mock<HttpMessageHandler>();
        handler.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.OK,
                Content = new StringContent("""{"id": 1, "name": "Test"}""",
                    Encoding.UTF8, "application/json")
            });

        var httpClient = new HttpClient(handler.Object)
        {
            BaseAddress = new Uri("https://api.example.com")
        };

        var service = new ExternalApiService(httpClient);

        // Act
        var result = await service.GetDataAsync(1, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Name.Should().Be("Test");
    }
}
```

## Integration Testing with WebApplicationFactory

Build integration tests that run the full ASP.NET Core pipeline in memory.

```csharp
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly WebApplicationFactory<Program> _factory;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace the real database with an in-memory one
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                if (descriptor != null) services.Remove(descriptor);

                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));

                // Seed test data
                var sp = services.BuildServiceProvider();
                using var scope = sp.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                db.Database.EnsureCreated();
                SeedTestData(db);
            });
        });

        _client = _factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsOkWithProducts()
    {
        var response = await _client.GetAsync("/api/products");

        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var products = await response.Content
            .ReadFromJsonAsync<List<ProductDto>>();

        products.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateProduct_WithValidData_ReturnsCreated()
    {
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Price = 29.99m,
            Category = "Electronics"
        };

        var response = await _client.PostAsJsonAsync("/api/products", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        created!.Name.Should().Be("New Product");
    }

    [Fact]
    public async Task CreateProduct_WithInvalidData_ReturnsBadRequest()
    {
        var request = new CreateProductRequest
        {
            Name = "",  // Invalid: empty name
            Price = -5, // Invalid: negative price
            Category = "Electronics"
        };

        var response = await _client.PostAsJsonAsync("/api/products", request);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    private static void SeedTestData(AppDbContext db)
    {
        db.Products.AddRange(
            new Product { Name = "Widget", Price = 9.99m, Category = "Tools" },
            new Product { Name = "Gadget", Price = 19.99m, Category = "Electronics" }
        );
        db.SaveChanges();
    }
}
```

## Testing EF Core with In-Memory Database

```csharp
public class OrderRepositoryTests : IDisposable
{
    private readonly AppDbContext _dbContext;
    private readonly OrderRepository _sut;

    public OrderRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;

        _dbContext = new AppDbContext(options);
        _sut = new OrderRepository(_dbContext);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingOrder_ReturnsOrder()
    {
        // Arrange
        var order = new Order
        {
            OrderNumber = "ORD-001",
            CustomerId = 1,
            Total = 100m,
            Items = [new OrderItem { ProductId = 1, Quantity = 2, UnitPrice = 50m }]
        };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();

        // Act
        var result = await _sut.GetByIdAsync(order.Id, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.OrderNumber.Should().Be("ORD-001");
        result.Items.Should().HaveCount(1);
    }

    [Fact]
    public async Task GetByIdAsync_NonExistentOrder_ReturnsNull()
    {
        var result = await _sut.GetByIdAsync(999, CancellationToken.None);
        result.Should().BeNull();
    }

    public void Dispose() => _dbContext.Dispose();
}
```

## Fixture and Collection Patterns

### Shared Fixture

```csharp
// Expensive setup shared across tests in a class
public class DatabaseFixture : IAsyncLifetime
{
    public AppDbContext DbContext { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=TestDb;Trusted_Connection=True")
            .Options;

        DbContext = new AppDbContext(options);
        await DbContext.Database.EnsureCreatedAsync();
        await SeedDataAsync();
    }

    public async Task DisposeAsync()
    {
        await DbContext.Database.EnsureDeletedAsync();
        await DbContext.DisposeAsync();
    }

    private async Task SeedDataAsync()
    {
        DbContext.Products.AddRange(TestData.Products);
        await DbContext.SaveChangesAsync();
    }
}

public class ProductQueryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public ProductQueryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task GetActiveProducts_ReturnsOnlyActive()
    {
        var products = await _fixture.DbContext.Products
            .Where(p => p.IsActive)
            .ToListAsync();

        products.Should().OnlyContain(p => p.IsActive);
    }
}
```

### Collection Fixture (Shared Across Multiple Test Classes)

```csharp
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>;

[Collection("Database")]
public class OrderQueryTests(DatabaseFixture fixture)
{
    [Fact]
    public async Task GetRecentOrders_ReturnsOrderedByDate()
    {
        var orders = await fixture.DbContext.Orders
            .OrderByDescending(o => o.CreatedAt)
            .Take(10)
            .ToListAsync();

        orders.Should().BeInDescendingOrder(o => o.CreatedAt);
    }
}
```

## FluentAssertions Patterns

```csharp
// Object assertions
result.Should().NotBeNull();
result.Should().BeEquivalentTo(expected, options =>
    options.Excluding(o => o.Id)
           .Excluding(o => o.CreatedAt));

// Collection assertions
products.Should().HaveCount(5);
products.Should().OnlyContain(p => p.Price > 0);
products.Should().ContainSingle(p => p.Name == "Widget");
products.Should().BeInAscendingOrder(p => p.Name);
products.Should().AllSatisfy(p =>
{
    p.Name.Should().NotBeNullOrEmpty();
    p.Price.Should().BePositive();
});

// Exception assertions
var act = () => service.ProcessAsync(null!, CancellationToken.None);
await act.Should().ThrowAsync<ArgumentNullException>()
    .WithParameterName("request");

// Execution time assertions
var act = () => service.ComputeAsync(data, CancellationToken.None);
await act.Should().CompleteWithinAsync(TimeSpan.FromSeconds(5));
```

## Test Organization

Organize tests to mirror the source project structure. Place unit tests and integration tests in separate projects.

```
tests/
  MyApp.UnitTests/
    Services/
      OrderServiceTests.cs
      ProductServiceTests.cs
    Validators/
      CreateOrderRequestValidatorTests.cs
    Domain/
      MoneyTests.cs
  MyApp.IntegrationTests/
    Api/
      ProductsApiTests.cs
      OrdersApiTests.cs
    Repositories/
      OrderRepositoryTests.cs
    Fixtures/
      DatabaseFixture.cs
      WebAppFixture.cs
```

## References

- [Testing Patterns](references/testing-patterns.md) — Unit testing services and controllers, integration testing with WebApplicationFactory, fixtures, mocking strategies, and testing EF Core.
- [Testing Tools](references/testing-tools.md) — xUnit configuration, Moq patterns, Bogus for fake data, AutoFixture, Testcontainers, Respawn, and coverlet for code coverage.
