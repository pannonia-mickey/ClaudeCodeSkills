# Testing Tools for .NET

## xUnit Configuration

```json
// xunit.runner.json
{
  "parallelizeAssembly": false,
  "parallelizeTestCollections": true,
  "maxParallelThreads": 0
}
```

## Bogus for Fake Data

```csharp
var faker = new Faker<Product>()
    .RuleFor(p => p.Name, f => f.Commerce.ProductName())
    .RuleFor(p => p.Price, f => f.Finance.Amount(1, 1000))
    .RuleFor(p => p.Category, f => f.Commerce.Categories(1)[0]);

var products = faker.Generate(50);
```

## AutoFixture for Auto-Mocking

```csharp
var fixture = new Fixture().Customize(new AutoMoqCustomization());
var sut = fixture.Create<OrderService>(); // All dependencies auto-mocked
var mockRepo = fixture.Freeze<Mock<IOrderRepository>>(); // Freeze to get the same mock
```

## Testcontainers for Real Database

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder().Build();
    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync() => await _container.StartAsync();
    public async Task DisposeAsync() => await _container.DisposeAsync();
}

[Collection("Database")]
public class ProductRepositoryTests(DatabaseFixture db) { }
```

## Respawn for Database Reset

```csharp
private static Respawner _respawner = null!;

public static async Task ResetDatabaseAsync(string connectionString)
{
    _respawner ??= await Respawner.CreateAsync(connectionString, new RespawnerOptions
    {
        TablesToIgnore = new[] { new Table("__EFMigrationsHistory") }
    });
    await _respawner.ResetAsync(connectionString);
}
```

## Coverlet for Code Coverage

```bash
dotnet test --collect:"XPlat Code Coverage" -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover
dotnet tool run reportgenerator -reports:**/coverage.cobertura.xml -targetdir:coverage-report -reporttypes:Html
```

## Verify for Snapshot Testing

```csharp
[Fact]
public Task GetProduct_ReturnsExpectedShape()
{
    var product = new ProductDto { Id = 1, Name = "Widget", Price = 9.99m };
    return Verify(product); // Creates/compares .verified.txt snapshot
}
```
