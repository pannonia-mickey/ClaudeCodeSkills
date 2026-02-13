# C# Testing Tools

## xUnit Configuration

### Project Setup

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Moq" Version="4.*" />
    <PackageReference Include="FluentAssertions" Version="7.*" />
    <PackageReference Include="Bogus" Version="35.*" />
    <PackageReference Include="coverlet.collector" Version="6.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
```

### xUnit Configuration File

```json
// xunit.runner.json — place in test project root
{
  "$schema": "https://xunit.net/schema/current/xunit.runner.schema.json",
  "diagnosticMessages": true,
  "maxParallelThreads": 4,
  "parallelizeTestCollections": true,
  "methodDisplay": "classAndMethod",
  "methodDisplayOptions": "replacePeriodWithComma"
}
```

### Custom Test Ordering

```csharp
// Run integration tests sequentially
[TestCaseOrderer("MyApp.Tests.PriorityOrderer", "MyApp.Tests")]
public class SequentialIntegrationTests
{
    [Fact, TestPriority(1)]
    public async Task Step1_CreateResource() { /* ... */ }

    [Fact, TestPriority(2)]
    public async Task Step2_UpdateResource() { /* ... */ }

    [Fact, TestPriority(3)]
    public async Task Step3_DeleteResource() { /* ... */ }
}

[AttributeUsage(AttributeTargets.Method)]
public class TestPriorityAttribute(int priority) : Attribute
{
    public int Priority { get; } = priority;
}

public class PriorityOrderer : ITestCaseOrderer
{
    public IEnumerable<TTestCase> OrderTestCases<TTestCase>(
        IEnumerable<TTestCase> testCases) where TTestCase : ITestCase
    {
        return testCases.OrderBy(tc =>
        {
            var priority = tc.TestMethod.Method
                .GetCustomAttributes(typeof(TestPriorityAttribute))
                .FirstOrDefault();
            return priority?.GetNamedArgument<int>(nameof(TestPriorityAttribute.Priority)) ?? 0;
        });
    }
}
```

### Custom Traits for Filtering

```csharp
// Tag tests for selective execution
[Trait("Category", "Integration")]
[Trait("Database", "Required")]
public class DatabaseIntegrationTests { /* ... */ }

[Trait("Category", "Unit")]
public class BusinessLogicTests { /* ... */ }

// Run only unit tests: dotnet test --filter "Category=Unit"
// Exclude integration tests: dotnet test --filter "Category!=Integration"
```

## Moq Advanced Patterns

### Setup with Argument Matching

```csharp
// Match any argument
mock.Setup(x => x.GetAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Product());

// Match specific condition
mock.Setup(x => x.GetAsync(It.Is<int>(id => id > 0), It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Product());

// Match with regex
mock.Setup(x => x.SearchAsync(It.IsRegex("^[A-Z]"), It.IsAny<CancellationToken>()))
    .ReturnsAsync(new List<Product>());

// Capture arguments
var capturedEmails = new List<string>();
mock.Setup(x => x.SendAsync(Capture.In(capturedEmails), It.IsAny<string>(),
        It.IsAny<string>(), It.IsAny<CancellationToken>()))
    .Returns(Task.CompletedTask);
```

### Mock Protected Members

```csharp
// Mock protected methods (useful for HttpMessageHandler)
var handlerMock = new Mock<HttpMessageHandler>();
handlerMock
    .Protected()
    .Setup<Task<HttpResponseMessage>>(
        "SendAsync",
        ItExpr.Is<HttpRequestMessage>(m => m.RequestUri!.PathAndQuery.Contains("/api/products")),
        ItExpr.IsAny<CancellationToken>())
    .ReturnsAsync(new HttpResponseMessage
    {
        StatusCode = HttpStatusCode.OK,
        Content = JsonContent.Create(new[] { new ProductDto(1, "Widget", 9.99m) })
    });

var client = new HttpClient(handlerMock.Object)
{
    BaseAddress = new Uri("https://api.example.com")
};
```

### Mock Auto Properties and Events

```csharp
// Auto-implement properties
var mock = new Mock<IUserSession>();
mock.SetupAllProperties(); // All properties become auto-properties
mock.Object.UserId = "123";
mock.Object.UserId.Should().Be("123");

// Raise events
var mock = new Mock<IEventBus>();
mock.Raise(x => x.MessageReceived += null, new MessageEventArgs("test"));
```

## Bogus for Fake Data Generation

Generate realistic test data with the Bogus library.

```csharp
public static class TestDataFactory
{
    private static readonly Faker<CreateProductRequest> ProductRequestFaker = new Faker<CreateProductRequest>()
        .RuleFor(p => p.Name, f => f.Commerce.ProductName())
        .RuleFor(p => p.Price, f => f.Finance.Amount(1, 1000))
        .RuleFor(p => p.Category, f => f.Commerce.Categories(1)[0])
        .RuleFor(p => p.Description, f => f.Commerce.ProductDescription());

    private static readonly Faker<Customer> CustomerFaker = new Faker<Customer>()
        .RuleFor(c => c.Name, f => f.Person.FullName)
        .RuleFor(c => c.Email, f => f.Person.Email)
        .RuleFor(c => c.Phone, f => f.Phone.PhoneNumber())
        .RuleFor(c => c.Address, f => new Address
        {
            Street = f.Address.StreetAddress(),
            City = f.Address.City(),
            State = f.Address.State(),
            ZipCode = f.Address.ZipCode(),
            Country = "US"
        });

    private static readonly Faker<Order> OrderFaker = new Faker<Order>()
        .RuleFor(o => o.OrderNumber, f => $"ORD-{f.Random.AlphaNumeric(8).ToUpper()}")
        .RuleFor(o => o.Status, f => f.PickRandom<OrderStatus>())
        .RuleFor(o => o.CreatedAt, f => f.Date.Recent(30))
        .RuleFor(o => o.Items, f => Enumerable.Range(1, f.Random.Int(1, 5))
            .Select(_ => new OrderItem
            {
                ProductId = f.Random.Int(1, 100),
                Quantity = f.Random.Int(1, 10),
                UnitPrice = f.Finance.Amount(5, 200)
            }).ToList());

    public static CreateProductRequest CreateProductRequest() => ProductRequestFaker.Generate();
    public static List<CreateProductRequest> CreateProductRequests(int count) => ProductRequestFaker.Generate(count);
    public static Customer CreateCustomer() => CustomerFaker.Generate();
    public static Order CreateOrder() => OrderFaker.Generate();
    public static List<Order> CreateOrders(int count) => OrderFaker.Generate(count);

    // Deterministic generation with seed
    public static List<Product> CreateSeededProducts(int count, int seed = 42) =>
        new Faker<Product>()
            .UseSeed(seed)
            .RuleFor(p => p.Name, f => f.Commerce.ProductName())
            .RuleFor(p => p.Price, f => f.Finance.Amount(1, 500))
            .Generate(count);
}

// Usage in tests
[Fact]
public async Task CreateProduct_WithValidData_Succeeds()
{
    var request = TestDataFactory.CreateProductRequest();
    var result = await _sut.CreateAsync(request, CancellationToken.None);
    result.IsSuccess.Should().BeTrue();
}
```

## AutoFixture

Auto-generate test data with minimal configuration.

```csharp
public class OrderServiceTests
{
    private readonly IFixture _fixture;

    public OrderServiceTests()
    {
        _fixture = new Fixture()
            .Customize(new AutoMoqCustomization { ConfigureMembers = true });

        // Customize specific types
        _fixture.Customize<CreateOrderRequest>(c => c
            .With(x => x.CustomerId, 1)
            .With(x => x.Items, _fixture.CreateMany<OrderItemRequest>(3).ToList()));
    }

    [Theory, AutoData]
    public async Task GetById_ExistingId_ReturnsProduct(int id, ProductDto expected)
    {
        // AutoData generates id and expected automatically
        var serviceMock = _fixture.Freeze<Mock<IProductService>>();
        serviceMock.Setup(x => x.GetByIdAsync(id, It.IsAny<CancellationToken>()))
            .ReturnsAsync(expected);

        var sut = _fixture.Create<ProductsController>();
        var result = await sut.GetById(id, CancellationToken.None);

        var okResult = result.Should().BeOfType<OkObjectResult>().Subject;
        okResult.Value.Should().Be(expected);
    }
}
```

## Testcontainers

Run real database instances in Docker for integration tests.

```csharp
public class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        // Apply migrations
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options;

        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync();
    }

    public async Task DisposeAsync() => await _container.DisposeAsync();
}

[CollectionDefinition("Postgres")]
public class PostgresCollection : ICollectionFixture<PostgresFixture>;

[Collection("Postgres")]
public class OrderRepositoryIntegrationTests(PostgresFixture fixture)
{
    [Fact]
    public async Task AddAndRetrieve_Order_RoundTrips()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(fixture.ConnectionString)
            .Options;

        await using var context = new AppDbContext(options);
        var repo = new OrderRepository(context);

        var order = new Order
        {
            OrderNumber = "ORD-TC-001",
            CustomerId = 1,
            Total = 100m,
            Status = OrderStatus.Pending
        };

        await repo.AddAsync(order, CancellationToken.None);
        await repo.SaveChangesAsync(CancellationToken.None);

        var retrieved = await repo.GetByIdAsync(order.Id, CancellationToken.None);
        retrieved.Should().NotBeNull();
        retrieved!.OrderNumber.Should().Be("ORD-TC-001");
    }
}
```

## Respawn for Database Cleanup

Reset database state between tests without recreating the schema.

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private Respawner _respawner = null!;
    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        await using var context = CreateContext();
        await context.Database.MigrateAsync();

        // Configure Respawner — skip tables that should not be reset
        using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();
        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            TablesToIgnore = ["__EFMigrationsHistory"],
            SchemasToInclude = ["public"]
        });
    }

    public async Task ResetDatabaseAsync()
    {
        using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();
        await _respawner.ResetAsync(connection);
    }

    public AppDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options;
        return new AppDbContext(options);
    }

    public async Task DisposeAsync() => await _container.DisposeAsync();
}

// Each test class resets the database before each test
public class OrderTests : IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;

    public OrderTests(DatabaseFixture fixture) => _fixture = fixture;

    public async Task InitializeAsync() => await _fixture.ResetDatabaseAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task CreateOrder_InCleanDatabase_HasIdOne()
    {
        await using var context = _fixture.CreateContext();
        // Test starts with a clean database
    }
}
```

## Coverlet for Code Coverage

### Configuration

```xml
<!-- In test project .csproj -->
<ItemGroup>
  <PackageReference Include="coverlet.collector" Version="6.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

### Running Coverage

```bash
# Generate coverage report
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

# Generate coverage with specific thresholds
dotnet test /p:CollectCoverage=true \
  /p:CoverletOutputFormat=cobertura \
  /p:CoverletOutput=./coverage/ \
  /p:Threshold=80 \
  /p:ThresholdType=line \
  /p:ThresholdStat=total

# Generate HTML report (requires reportgenerator tool)
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
  -reports:"./coverage/**/coverage.cobertura.xml" \
  -targetdir:"./coverage/report" \
  -reporttypes:Html
```

### Coverage Configuration in runsettings

```xml
<!-- coverlet.runsettings -->
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Exclude>[*]*.Migrations.*,[*]*.Program</Exclude>
          <Include>[MyApp.Domain]*,[MyApp.Application]*</Include>
          <ExcludeByAttribute>
            GeneratedCodeAttribute,ObsoleteAttribute,ExcludeFromCodeCoverageAttribute
          </ExcludeByAttribute>
          <SingleHit>false</SingleHit>
          <UseSourceLink>true</UseSourceLink>
          <IncludeTestAssembly>false</IncludeTestAssembly>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

```bash
# Run with runsettings
dotnet test --settings coverlet.runsettings --results-directory ./coverage
```

## Snapshot Testing with Verify

```csharp
// Verify serializes the result and compares against a stored snapshot
[Fact]
public async Task GetOrderSummary_ReturnsExpectedShape()
{
    var order = TestDataFactory.CreateSeededOrder(seed: 42);
    var summary = _sut.GetSummary(order);

    await Verify(summary);
    // First run creates OrderTests.GetOrderSummary_ReturnsExpectedShape.verified.txt
    // Subsequent runs compare against it
}

// Customize serialization
[Fact]
public async Task GetProducts_ReturnsExpectedList()
{
    var products = await _sut.GetAllAsync(CancellationToken.None);

    await Verify(products)
        .IgnoreMember("Id")
        .IgnoreMember("CreatedAt");
}
```
