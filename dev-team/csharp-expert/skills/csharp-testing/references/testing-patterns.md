# C# Testing Patterns

## Unit Testing Services

### Testing Business Logic in Services

Isolate the service under test by mocking all dependencies. Focus on verifying behavior, not implementation details.

```csharp
public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _repoMock = new();
    private readonly Mock<ILogger<ProductService>> _loggerMock = new();
    private readonly Mock<ICacheService> _cacheMock = new();
    private readonly ProductService _sut;

    public ProductServiceTests()
    {
        _sut = new ProductService(
            _repoMock.Object,
            _loggerMock.Object,
            _cacheMock.Object);
    }

    [Fact]
    public async Task GetByIdAsync_WhenCached_ReturnsCachedValue()
    {
        // Arrange
        var expected = new ProductDto(1, "Widget", 9.99m);
        _cacheMock
            .Setup(x => x.GetAsync<ProductDto>("product:1", It.IsAny<CancellationToken>()))
            .ReturnsAsync(expected);

        // Act
        var result = await _sut.GetByIdAsync(1, CancellationToken.None);

        // Assert
        result.Should().Be(expected);
        _repoMock.Verify(
            x => x.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()),
            Times.Never,
            "Should not hit the database when cache has the value");
    }

    [Fact]
    public async Task GetByIdAsync_WhenNotCached_QueriesDatabaseAndCaches()
    {
        // Arrange
        _cacheMock
            .Setup(x => x.GetAsync<ProductDto>("product:1", It.IsAny<CancellationToken>()))
            .ReturnsAsync((ProductDto?)null);

        var entity = new Product { Id = 1, Name = "Widget", Price = 9.99m };
        _repoMock
            .Setup(x => x.GetByIdAsync(1, It.IsAny<CancellationToken>()))
            .ReturnsAsync(entity);

        // Act
        var result = await _sut.GetByIdAsync(1, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Name.Should().Be("Widget");

        _cacheMock.Verify(
            x => x.SetAsync("product:1", It.IsAny<ProductDto>(),
                It.IsAny<TimeSpan>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task CreateAsync_WithDuplicateName_ReturnsFailure()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "Existing Product",
            Price = 29.99m,
            Category = "Electronics"
        };

        _repoMock
            .Setup(x => x.ExistsByNameAsync("Existing Product", It.IsAny<CancellationToken>()))
            .ReturnsAsync(true);

        // Act
        var result = await _sut.CreateAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("already exists");
    }
}
```

### Testing Validators

```csharp
public class CreateOrderRequestValidatorTests
{
    private readonly CreateOrderRequestValidator _sut = new();

    [Fact]
    public void Validate_ValidRequest_Passes()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            ShippingAddress = new AddressDto
            {
                Street = "123 Main St",
                City = "Springfield",
                State = "IL",
                ZipCode = "62701"
            },
            Items =
            [
                new OrderItemRequest { ProductId = 1, Quantity = 2 },
                new OrderItemRequest { ProductId = 2, Quantity = 1 }
            ]
        };

        var result = _sut.Validate(request);

        result.IsValid.Should().BeTrue();
        result.Errors.Should().BeEmpty();
    }

    [Fact]
    public void Validate_EmptyItems_HasError()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            ShippingAddress = ValidAddress(),
            Items = []
        };

        var result = _sut.Validate(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().ContainSingle()
            .Which.PropertyName.Should().Be("Items");
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public void Validate_InvalidQuantity_HasError(int quantity)
    {
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            ShippingAddress = ValidAddress(),
            Items = [new OrderItemRequest { ProductId = 1, Quantity = quantity }]
        };

        var result = _sut.Validate(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName.Contains("Quantity"));
    }

    private static AddressDto ValidAddress() => new()
    {
        Street = "123 Main St",
        City = "Springfield",
        State = "IL",
        ZipCode = "62701"
    };
}
```

## Unit Testing Controllers

Test controller logic in isolation. Verify correct status codes and response types.

```csharp
public class ProductsControllerTests
{
    private readonly Mock<IProductService> _serviceMock = new();
    private readonly ProductsController _sut;

    public ProductsControllerTests()
    {
        _sut = new ProductsController(_serviceMock.Object);
    }

    [Fact]
    public async Task GetById_ExistingProduct_ReturnsOk()
    {
        var product = new ProductDto(1, "Widget", 9.99m);
        _serviceMock
            .Setup(x => x.GetByIdAsync(1, It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        var result = await _sut.GetById(1, CancellationToken.None);

        var okResult = result.Should().BeOfType<OkObjectResult>().Subject;
        okResult.Value.Should().Be(product);
    }

    [Fact]
    public async Task GetById_NonExistentProduct_ReturnsNotFound()
    {
        _serviceMock
            .Setup(x => x.GetByIdAsync(999, It.IsAny<CancellationToken>()))
            .ReturnsAsync((ProductDto?)null);

        var result = await _sut.GetById(999, CancellationToken.None);

        result.Should().BeOfType<NotFoundResult>();
    }

    [Fact]
    public async Task Create_ValidRequest_ReturnsCreatedAtAction()
    {
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Price = 29.99m,
            Category = "Electronics"
        };

        var created = new ProductDto(1, "New Product", 29.99m);
        _serviceMock
            .Setup(x => x.CreateAsync(request, It.IsAny<CancellationToken>()))
            .ReturnsAsync(Result<ProductDto>.Success(created));

        var result = await _sut.Create(request, CancellationToken.None);

        var createdResult = result.Should().BeOfType<CreatedAtActionResult>().Subject;
        createdResult.ActionName.Should().Be(nameof(ProductsController.GetById));
        createdResult.Value.Should().Be(created);
    }
}
```

## Integration Testing with WebApplicationFactory

### Custom WebApplicationFactory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the real database registration
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (dbDescriptor != null) services.Remove(dbDescriptor);

            // Add test database
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase($"TestDb-{Guid.NewGuid()}"));

            // Replace external services with fakes
            var emailDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(IEmailService));
            if (emailDescriptor != null) services.Remove(emailDescriptor);
            services.AddSingleton<IEmailService, FakeEmailService>();

            // Seed test data
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
            TestDataSeeder.Seed(db);
        });

        builder.ConfigureTestServices(services =>
        {
            // Override authentication for testing
            services.AddAuthentication("Test")
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    "Test", options => { });
        });
    }
}

// Test authentication handler
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "test-user-id"),
            new Claim(ClaimTypes.Name, "Test User"),
            new Claim(ClaimTypes.Role, "Admin")
        };

        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

### Integration Test Base Class

```csharp
public abstract class IntegrationTestBase : IClassFixture<CustomWebApplicationFactory>, IDisposable
{
    protected readonly HttpClient Client;
    protected readonly CustomWebApplicationFactory Factory;
    private readonly IServiceScope _scope;
    protected readonly AppDbContext DbContext;

    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
        _scope = factory.Services.CreateScope();
        DbContext = _scope.ServiceProvider.GetRequiredService<AppDbContext>();
    }

    protected async Task<T?> GetAsync<T>(string url)
    {
        var response = await Client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>();
    }

    protected async Task<HttpResponseMessage> PostAsync<T>(string url, T body)
    {
        return await Client.PostAsJsonAsync(url, body);
    }

    public void Dispose()
    {
        _scope.Dispose();
        Client.Dispose();
    }
}

// Usage
public class OrdersApiTests(CustomWebApplicationFactory factory)
    : IntegrationTestBase(factory)
{
    [Fact]
    public async Task GetOrders_ReturnsAllOrders()
    {
        var orders = await GetAsync<List<OrderDto>>("/api/orders");

        orders.Should().NotBeNull();
        orders.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateOrder_PersistsToDatabase()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = 1,
            Items = [new OrderItemRequest { ProductId = 1, Quantity = 3 }]
        };

        var response = await PostAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);

        // Verify it was actually persisted
        var created = await response.Content.ReadFromJsonAsync<OrderDto>();
        var fromDb = await DbContext.Orders.FindAsync(created!.Id);
        fromDb.Should().NotBeNull();
    }
}
```

## Testing EF Core

### Testing with SQLite In-Memory

SQLite in-memory is more realistic than EF Core InMemoryDatabase because it supports SQL features like constraints and transactions.

```csharp
public class EfCoreTestBase : IDisposable
{
    protected readonly AppDbContext DbContext;
    private readonly SqliteConnection _connection;

    protected EfCoreTestBase()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        DbContext = new AppDbContext(options);
        DbContext.Database.EnsureCreated();
    }

    public void Dispose()
    {
        DbContext.Dispose();
        _connection.Dispose();
    }
}

public class OrderRepositoryTests : EfCoreTestBase
{
    private readonly OrderRepository _sut;

    public OrderRepositoryTests()
    {
        _sut = new OrderRepository(DbContext);
    }

    [Fact]
    public async Task AddAsync_PersistsOrderWithItems()
    {
        // Arrange
        var order = new Order
        {
            OrderNumber = "ORD-001",
            CustomerId = 1,
            Total = 150m,
            Status = OrderStatus.Pending,
            Items =
            [
                new OrderItem { ProductId = 1, Quantity = 2, UnitPrice = 50m },
                new OrderItem { ProductId = 2, Quantity = 1, UnitPrice = 50m }
            ]
        };

        // Act
        await _sut.AddAsync(order, CancellationToken.None);
        await _sut.SaveChangesAsync(CancellationToken.None);

        // Assert — use a fresh query to verify persistence
        var saved = await DbContext.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.OrderNumber == "ORD-001");

        saved.Should().NotBeNull();
        saved!.Items.Should().HaveCount(2);
        saved.Total.Should().Be(150m);
    }

    [Fact]
    public async Task SoftDelete_SetsIsDeletedFlag()
    {
        // Arrange
        var order = new Order
        {
            OrderNumber = "ORD-002",
            CustomerId = 1,
            Total = 100m,
            Status = OrderStatus.Active
        };
        DbContext.Orders.Add(order);
        await DbContext.SaveChangesAsync();

        // Act
        order.IsDeleted = true;
        await DbContext.SaveChangesAsync();

        // Assert — query with IgnoreQueryFilters to find soft-deleted entities
        var deleted = await DbContext.Orders
            .IgnoreQueryFilters()
            .FirstOrDefaultAsync(o => o.OrderNumber == "ORD-002");

        deleted.Should().NotBeNull();
        deleted!.IsDeleted.Should().BeTrue();

        // Normal query should not return it
        var normal = await DbContext.Orders
            .FirstOrDefaultAsync(o => o.OrderNumber == "ORD-002");
        normal.Should().BeNull();
    }
}
```

## Testing Domain Logic

### Value Object Tests

```csharp
public class MoneyTests
{
    [Fact]
    public void Add_SameCurrency_ReturnsSum()
    {
        var a = new Money(10m, "USD");
        var b = new Money(20m, "USD");

        var result = a.Add(b);

        result.Amount.Should().Be(30m);
        result.Currency.Should().Be("USD");
    }

    [Fact]
    public void Add_DifferentCurrency_Throws()
    {
        var usd = new Money(10m, "USD");
        var eur = new Money(20m, "EUR");

        var act = () => usd.Add(eur);

        act.Should().Throw<InvalidOperationException>()
            .WithMessage("*Currency mismatch*");
    }

    [Fact]
    public void Records_WithSameValues_AreEqual()
    {
        var a = new Money(10m, "USD");
        var b = new Money(10m, "USD");

        a.Should().Be(b);
        (a == b).Should().BeTrue();
    }
}
```

### Aggregate Root Tests

```csharp
public class OrderTests
{
    [Fact]
    public void AddItem_IncreasesTotalAndItemCount()
    {
        var order = Order.Create(customerId: 1);

        order.AddItem(productId: 10, quantity: 2, unitPrice: 25m);

        order.Items.Should().HaveCount(1);
        order.Total.Should().Be(50m);
    }

    [Fact]
    public void AddItem_SameProduct_IncreasesQuantity()
    {
        var order = Order.Create(customerId: 1);

        order.AddItem(productId: 10, quantity: 2, unitPrice: 25m);
        order.AddItem(productId: 10, quantity: 3, unitPrice: 25m);

        order.Items.Should().HaveCount(1);
        order.Items[0].Quantity.Should().Be(5);
        order.Total.Should().Be(125m);
    }

    [Fact]
    public void Cancel_ActiveOrder_SetsStatusAndRaisesDomainEvent()
    {
        var order = Order.Create(customerId: 1);
        order.AddItem(productId: 10, quantity: 1, unitPrice: 100m);

        order.Cancel("Customer requested");

        order.Status.Should().Be(OrderStatus.Cancelled);
        order.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderCancelledEvent>();
    }

    [Fact]
    public void Cancel_AlreadyCancelled_Throws()
    {
        var order = Order.Create(customerId: 1);
        order.Cancel("First cancel");

        var act = () => order.Cancel("Second cancel");

        act.Should().Throw<InvalidOperationException>()
            .WithMessage("*already cancelled*");
    }
}
```
