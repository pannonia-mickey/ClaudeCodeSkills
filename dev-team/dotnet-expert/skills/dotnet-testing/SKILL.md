---
name: .NET Testing
description: This skill should be used when the user asks about "xUnit", "NUnit", "C# test", "Moq", "FluentAssertions", "WebApplicationFactory", "integration testing .NET", "mock dependencies C#", "test EF Core", or "test data builders .NET". It covers unit tests with xUnit, mocking with Moq, FluentAssertions, integration testing with WebApplicationFactory, and testing EF Core.
---

# .NET Testing

## Unit Test Structure (Arrange-Act-Assert)

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repositoryMock = new();
    private readonly Mock<IInventoryService> _inventoryMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repositoryMock.Object, _inventoryMock.Object);
    }

    [Fact]
    public async Task CreateOrder_WithSufficientStock_ReturnsSuccess()
    {
        // Arrange
        var request = new CreateOrderRequest { CustomerId = 1, Items = [new() { ProductId = 10, Quantity = 2 }] };
        _inventoryMock.Setup(x => x.CheckStockAsync(10, It.IsAny<CancellationToken>())).ReturnsAsync(50);

        // Act
        var result = await _sut.CreateOrderAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value!.CustomerId.Should().Be(1);
        _repositoryMock.Verify(x => x.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

## Data-Driven Tests

```csharp
[Theory]
[InlineData(100, 0.1, 90)]
[InlineData(200, 0.2, 160)]
[InlineData(50, 0, 50)]
public void ApplyDiscount_ReturnsCorrectPrice(decimal price, decimal rate, decimal expected)
{
    PriceCalculator.ApplyDiscount(price, rate).Should().Be(expected);
}

// MemberData for complex objects
public static TheoryData<CreateOrderRequest, bool> ValidationCases => new()
{
    { new CreateOrderRequest { CustomerId = 1, Items = [new() { ProductId = 1, Quantity = 1 }] }, true },
    { new CreateOrderRequest { CustomerId = 0, Items = [] }, false },
};

[Theory]
[MemberData(nameof(ValidationCases))]
public void Validate_ReturnsExpectedResult(CreateOrderRequest request, bool expectedIsValid)
{
    new CreateOrderRequestValidator().Validate(request).IsValid.Should().Be(expectedIsValid);
}
```

## Mocking with Moq

```csharp
// Setup return values
_repositoryMock.Setup(x => x.GetByIdAsync(42, It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Order { Id = 42, Total = 100 });

// Setup sequences
_serviceMock.SetupSequence(x => x.ProcessAsync(It.IsAny<Request>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(Result.Success())
    .ThrowsAsync(new TimeoutException());

// Argument capture
var captured = new List<Order>();
_repositoryMock.Setup(x => x.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
    .Callback<Order, CancellationToken>((order, _) => captured.Add(order))
    .Returns(Task.CompletedTask);

// Verify call count
_repositoryMock.Verify(x => x.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Exactly(2));
```

## FluentAssertions

```csharp
result.Should().NotBeNull();
result.Should().BeEquivalentTo(expected, o => o.Excluding(x => x.Id).Excluding(x => x.CreatedAt));

products.Should().HaveCount(5);
products.Should().OnlyContain(p => p.Price > 0);
products.Should().BeInAscendingOrder(p => p.Name);

var act = () => service.ProcessAsync(null!, CancellationToken.None);
await act.Should().ThrowAsync<ArgumentNullException>().WithParameterName("request");
```

## Integration Testing with WebApplicationFactory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null) services.Remove(descriptor);
            services.AddDbContext<AppDbContext>(options => options.UseInMemoryDatabase("TestDb_" + Guid.NewGuid()));

            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
            db.Products.AddRange(new Product("Widget", 9.99m), new Product("Gadget", 19.99m));
            db.SaveChanges();
        });
    }
}

public class ProductsApiTests(CustomWebApplicationFactory factory) : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetProducts_ReturnsOkWithProducts()
    {
        var response = await _client.GetAsync("/api/products");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateProduct_WithInvalidData_ReturnsBadRequest()
    {
        var response = await _client.PostAsJsonAsync("/api/products", new { Name = "", Price = -5 });
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

## Testing EF Core with In-Memory Database

```csharp
public class OrderRepositoryTests : IDisposable
{
    private readonly AppDbContext _db;
    private readonly OrderRepository _sut;

    public OrderRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseInMemoryDatabase(Guid.NewGuid().ToString()).Options;
        _db = new AppDbContext(options);
        _sut = new OrderRepository(_db);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingOrder_ReturnsOrder()
    {
        _db.Orders.Add(new Order { OrderNumber = "ORD-001", CustomerId = 1, Total = 100m });
        await _db.SaveChangesAsync();
        var result = await _sut.GetByIdAsync(1, CancellationToken.None);
        result.Should().NotBeNull();
        result!.OrderNumber.Should().Be("ORD-001");
    }

    public void Dispose() => _db.Dispose();
}
```

## Test Data Builders

```csharp
public class ProductBuilder
{
    private string _name = "Default Product";
    private decimal _price = 9.99m;
    public static ProductBuilder Create() => new();
    public ProductBuilder WithName(string name) { _name = name; return this; }
    public ProductBuilder WithPrice(decimal price) { _price = price; return this; }
    public Product Build() => new(_name, _price);
}

var product = ProductBuilder.Create().WithName("Premium Widget").WithPrice(99.99m).Build();
```

## Test Organization

```
tests/
  MyApp.UnitTests/
    Services/OrderServiceTests.cs
    Validators/CreateOrderValidatorTests.cs
    Domain/MoneyTests.cs
  MyApp.IntegrationTests/
    Api/ProductsApiTests.cs
    Fixtures/DatabaseFixture.cs
```

## References

- [Testing Patterns](references/testing-patterns.md) — Mocking patterns, testing domain events, MediatR handlers, middleware, snapshot testing.
- [Testing Tools](references/testing-tools.md) — xUnit config, Bogus, AutoFixture, Testcontainers, Respawn, coverlet.
