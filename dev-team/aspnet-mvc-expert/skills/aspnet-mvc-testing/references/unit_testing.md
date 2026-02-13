# Unit Testing Advanced Patterns

Advanced unit testing patterns for ASP.NET Core MVC applications.

## Testing MediatR Pipeline Behaviors

```csharp
public class ValidationBehaviorTests
{
    [Fact]
    public async Task Handle_NoValidators_CallsNext()
    {
        // Arrange
        var validators = Enumerable.Empty<IValidator<TestCommand>>();
        var behavior = new ValidationBehavior<TestCommand, int>(validators);
        var nextCalled = false;

        // Act
        await behavior.Handle(
            new TestCommand(),
            () => { nextCalled = true; return Task.FromResult(42); },
            CancellationToken.None);

        // Assert
        Assert.True(nextCalled);
    }

    [Fact]
    public async Task Handle_ValidationFails_ThrowsValidationException()
    {
        // Arrange
        var validator = new InlineValidator<TestCommand>();
        validator.RuleFor(x => x.Name).NotEmpty();
        var behavior = new ValidationBehavior<TestCommand, int>(
            new[] { validator });

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(
            () => behavior.Handle(
                new TestCommand { Name = "" },
                () => Task.FromResult(42),
                CancellationToken.None));
    }
}
```

## Testing Domain Events

```csharp
public class OrderTests
{
    [Fact]
    public void Place_ValidOrder_RaisesOrderPlacedEvent()
    {
        // Arrange
        var order = new Order(customerId: 1);
        order.AddItem(new Product("Widget", 10m), 2);

        // Act
        order.Place();

        // Assert
        var domainEvent = Assert.Single(order.DomainEvents);
        var orderPlacedEvent = Assert.IsType<OrderPlacedEvent>(domainEvent);
        Assert.Equal(order.Id, orderPlacedEvent.OrderId);
    }

    [Fact]
    public void Place_EmptyOrder_ThrowsAndNoEvents()
    {
        // Arrange
        var order = new Order(customerId: 1);

        // Act & Assert
        Assert.Throws<DomainException>(() => order.Place());
        Assert.Empty(order.DomainEvents);
    }
}
```

## Testing with AutoFixture

Generate test data automatically:

```csharp
public class ProductHandlerTests
{
    private readonly IFixture _fixture;

    public ProductHandlerTests()
    {
        _fixture = new Fixture()
            .Customize(new AutoMoqCustomization());

        // Customize specific types
        _fixture.Customize<Product>(c => c
            .With(p => p.Price, _fixture.Create<decimal>() % 1000 + 1)
            .With(p => p.Name, _fixture.Create<string>()[..50]));
    }

    [Fact]
    public async Task Handle_ValidCommand_ReturnsId()
    {
        // Arrange
        var command = _fixture.Create<CreateProductCommand>();
        var handler = _fixture.Create<CreateProductHandler>();

        // Act
        var result = await handler.Handle(command, CancellationToken.None);

        // Assert
        Assert.True(result > 0);
    }
}
```

## Testing Action Filters

```csharp
public class ApiExceptionFilterTests
{
    [Fact]
    public void OnException_NotFoundException_Returns404()
    {
        // Arrange
        var filter = new ApiExceptionFilterAttribute();
        var context = CreateExceptionContext(
            new NotFoundException("Product", 1));

        // Act
        filter.OnException(context);

        // Assert
        var result = Assert.IsType<ObjectResult>(context.Result);
        Assert.Equal(404, result.StatusCode);
        var problemDetails = Assert.IsType<ProblemDetails>(result.Value);
        Assert.Equal("Not Found", problemDetails.Title);
    }

    [Fact]
    public void OnException_ValidationException_Returns400()
    {
        // Arrange
        var failures = new List<ValidationFailure>
        {
            new("Name", "Name is required"),
            new("Price", "Price must be positive")
        };
        var filter = new ApiExceptionFilterAttribute();
        var context = CreateExceptionContext(
            new ValidationException(failures));

        // Act
        filter.OnException(context);

        // Assert
        var result = Assert.IsType<ObjectResult>(context.Result);
        Assert.Equal(400, result.StatusCode);
    }

    private static ExceptionContext CreateExceptionContext(Exception exception)
    {
        var actionContext = new ActionContext(
            new DefaultHttpContext(),
            new RouteData(),
            new ActionDescriptor());
        return new ExceptionContext(actionContext, new List<IFilterMetadata>())
        {
            Exception = exception
        };
    }
}
```

## Testing Middleware

```csharp
public class GlobalExceptionMiddlewareTests
{
    [Fact]
    public async Task InvokeAsync_NoException_CallsNext()
    {
        // Arrange
        var context = new DefaultHttpContext();
        context.Response.Body = new MemoryStream();
        var nextCalled = false;

        var middleware = new GlobalExceptionMiddleware(
            ctx => { nextCalled = true; return Task.CompletedTask; },
            Mock.Of<ILogger<GlobalExceptionMiddleware>>());

        // Act
        await middleware.InvokeAsync(context);

        // Assert
        Assert.True(nextCalled);
        Assert.Equal(200, context.Response.StatusCode);
    }

    [Fact]
    public async Task InvokeAsync_UnhandledException_Returns500()
    {
        // Arrange
        var context = new DefaultHttpContext();
        context.Response.Body = new MemoryStream();

        var middleware = new GlobalExceptionMiddleware(
            _ => throw new InvalidOperationException("Something broke"),
            Mock.Of<ILogger<GlobalExceptionMiddleware>>());

        // Act
        await middleware.InvokeAsync(context);

        // Assert
        Assert.Equal(500, context.Response.StatusCode);
    }
}
```

## Testing with NSubstitute (Alternative to Moq)

```csharp
public class ProductServiceTests
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ProductService _service;

    public ProductServiceTests()
    {
        _repository = Substitute.For<IProductRepository>();
        _unitOfWork = Substitute.For<IUnitOfWork>();
        _service = new ProductService(_repository, _unitOfWork);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingProduct_ReturnsProduct()
    {
        // Arrange
        var product = new Product("Widget", 10m);
        _repository.GetByIdAsync(1, Arg.Any<CancellationToken>())
            .Returns(product);

        // Act
        var result = await _service.GetByIdAsync(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Widget", result.Name);
    }

    [Fact]
    public async Task CreateAsync_ValidProduct_SavesAndNotifies()
    {
        // Arrange
        var command = new CreateProductCommand("Widget", 10m);

        // Act
        await _service.CreateAsync(command);

        // Assert
        _repository.Received(1).Add(Arg.Is<Product>(p => p.Name == "Widget"));
        await _unitOfWork.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }
}
```

## Testing Value Objects

```csharp
public class MoneyTests
{
    [Fact]
    public void Constructor_NegativeAmount_ThrowsDomainException()
    {
        Assert.Throws<DomainException>(() => new Money(-1, "USD"));
    }

    [Fact]
    public void Add_SameCurrency_ReturnsSummedAmount()
    {
        var a = new Money(10, "USD");
        var b = new Money(20, "USD");
        var result = a.Add(b);
        Assert.Equal(new Money(30, "USD"), result);
    }

    [Fact]
    public void Add_DifferentCurrency_ThrowsDomainException()
    {
        var usd = new Money(10, "USD");
        var eur = new Money(20, "EUR");
        Assert.Throws<DomainException>(() => usd.Add(eur));
    }

    [Theory]
    [InlineData(10, "USD", 10, "USD", true)]
    [InlineData(10, "USD", 20, "USD", false)]
    [InlineData(10, "USD", 10, "EUR", false)]
    public void Equals_VariousScenarios_ReturnsExpected(
        decimal amount1, string currency1,
        decimal amount2, string currency2,
        bool expected)
    {
        var a = new Money(amount1, currency1);
        var b = new Money(amount2, currency2);
        Assert.Equal(expected, a.Equals(b));
    }
}
```

## Snapshot Testing

For complex output verification:

```csharp
public class MappingProfileTests
{
    private readonly IMapper _mapper;

    public MappingProfileTests()
    {
        var config = new MapperConfiguration(cfg =>
            cfg.AddProfile<MappingProfile>());
        _mapper = config.CreateMapper();
    }

    [Fact]
    public void ProductToDto_MapsAllProperties()
    {
        // Arrange
        var product = new Product("Widget", 29.99m)
        {
            Id = 1,
            Category = new Category { Name = "Electronics" }
        };

        // Act
        var dto = _mapper.Map<ProductDto>(product);

        // Assert
        Assert.Equal(1, dto.Id);
        Assert.Equal("Widget", dto.Name);
        Assert.Equal(29.99m, dto.Price);
        Assert.Equal("Electronics", dto.CategoryName);
    }

    [Fact]
    public void AllMappings_AreValid()
    {
        var config = new MapperConfiguration(cfg =>
            cfg.AddProfile<MappingProfile>());
        config.AssertConfigurationIsValid();
    }
}
```

## Test Organization Conventions

### Naming Convention

```
[MethodUnderTest]_[Scenario]_[ExpectedResult]

Examples:
- GetByIdAsync_ExistingProduct_ReturnsProduct
- Place_EmptyOrder_ThrowsDomainException
- Handle_ValidationFails_ThrowsValidationException
- Add_SameCurrency_ReturnsSummedAmount
```

### Class Per Method Under Test (for complex classes)

```
tests/
├── Orders/
│   ├── Order_AddItemTests.cs
│   ├── Order_PlaceTests.cs
│   ├── Order_CancelTests.cs
│   └── Order_ShipTests.cs
```

### One Assert Per Test (Logical)

One **logical** assertion, not necessarily one `Assert.*` call:

```csharp
// Good — one logical assertion (verifying a DTO)
var dto = mapper.Map<ProductDto>(product);
Assert.Equal("Widget", dto.Name);
Assert.Equal(29.99m, dto.Price);  // Still testing one thing: "mapping works correctly"

// Bad — testing multiple unrelated behaviors
[Fact]
public void CreateProduct_Works()
{
    // Testing creation AND validation AND mapping in one test
}
```
