# Testing Patterns in .NET

## Testing MediatR Handlers

```csharp
public class CreateOrderHandlerTests
{
    private readonly Mock<IOrderRepository> _repoMock = new();
    private readonly Mock<IUnitOfWork> _uowMock = new();
    private readonly CreateOrderHandler _sut;

    public CreateOrderHandlerTests() => _sut = new(_repoMock.Object, _uowMock.Object);

    [Fact]
    public async Task Handle_ValidCommand_CreatesOrderAndSaves()
    {
        var command = new CreateOrderCommand("Widget", 29.99m);
        _uowMock.Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>())).ReturnsAsync(1);

        var result = await _sut.Handle(command, CancellationToken.None);

        _repoMock.Verify(r => r.Add(It.Is<Product>(p => p.Name == "Widget")), Times.Once);
        _uowMock.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
        Assert.True(result > 0);
    }
}
```

## Testing Pipeline Behaviors

```csharp
[Fact]
public async Task ValidationBehavior_InvalidRequest_ThrowsValidationException()
{
    var validator = new CreateProductValidator();
    var behavior = new ValidationBehavior<CreateProductCommand, int>(new[] { validator });
    var invalidCommand = new CreateProductCommand("", -5);

    await Assert.ThrowsAsync<ValidationException>(() =>
        behavior.Handle(invalidCommand, () => Task.FromResult(1), CancellationToken.None));
}
```

## Testing Domain Events

```csharp
[Fact]
public void Submit_DraftOrder_RaisesOrderSubmittedEvent()
{
    var order = Order.Create(customerId: 1);
    order.AddItem(productId: 10, quantity: 2, unitPrice: 50m);

    order.Submit();

    order.DomainEvents.Should().ContainSingle()
        .Which.Should().BeOfType<OrderSubmittedEvent>();
}
```

## Testing Middleware

```csharp
[Fact]
public async Task ExceptionMiddleware_UnhandledException_Returns500()
{
    var middleware = new GlobalExceptionMiddleware(
        next: _ => throw new InvalidOperationException("Test error"),
        new Mock<ILogger<GlobalExceptionMiddleware>>().Object);

    var context = new DefaultHttpContext();
    context.Response.Body = new MemoryStream();

    await middleware.InvokeAsync(context);

    context.Response.StatusCode.Should().Be(500);
}
```

## Argument Capture with Moq

```csharp
var captured = new List<Order>();
_repoMock.Setup(x => x.AddAsync(Capture.In(captured), It.IsAny<CancellationToken>())).Returns(Task.CompletedTask);

await _sut.CreateOrderAsync(request, CancellationToken.None);

captured.Should().ContainSingle();
captured[0].CustomerId.Should().Be(1);
captured[0].Items.Should().HaveCount(2);
```

## Test Naming Convention

Format: `MethodUnderTest_Scenario_ExpectedResult`

```csharp
[Fact] public async Task CreateOrder_WithSufficientStock_ReturnsSuccess() { }
[Fact] public async Task CreateOrder_WithInsufficientStock_ReturnsFailure() { }
[Fact] public void Submit_AlreadySubmittedOrder_ThrowsDomainException() { }
```
