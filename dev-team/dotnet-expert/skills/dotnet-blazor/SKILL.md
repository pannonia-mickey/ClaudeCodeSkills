---
name: .NET Blazor
description: This skill should be used when the user asks to "create a Blazor component", "build Blazor app", "Blazor Server vs WebAssembly", "Blazor forms", "Blazor state management", "Blazor JS interop", "Blazor authentication", or "Blazor lifecycle". It covers Blazor Server and WebAssembly hosting, component design, forms and validation, state management, JS interop, and authentication.
---

# Blazor Development

## Hosting Models

| Feature | Blazor Server | Blazor WebAssembly | Interactive Auto |
|---------|--------------|-------------------|-----------------|
| Execution | Server-side via SignalR | Client-side in browser | Server first, then WASM |
| Startup | Fast | Slower (download .NET) | Fast then seamless |
| Offline | No | Yes | Partial |
| Server load | Higher | Lower | Adaptive |

Choose Server for internal apps with low latency needs. Choose WASM for public apps or offline support. Choose Auto for the best of both.

## Component Lifecycle

```csharp
@code {
    [Parameter] public int ProductId { get; set; }
    [Parameter] public EventCallback<Product> OnProductSelected { get; set; }
    [CascadingParameter] public Theme CurrentTheme { get; set; } = default!;

    private Product? _product;
    private bool _isLoading = true;

    // Called once when component is first initialized
    protected override async Task OnInitializedAsync()
    {
        _product = await ProductService.GetByIdAsync(ProductId);
        _isLoading = false;
    }

    // Called when parameters change (including first render)
    protected override async Task OnParametersSetAsync()
    {
        if (_product?.Id != ProductId)
        {
            _isLoading = true;
            _product = await ProductService.GetByIdAsync(ProductId);
            _isLoading = false;
        }
    }

    // Called after each render — use for JS interop
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            await JS.InvokeVoidAsync("initializeChart", _chartElement);
    }
}
```

## Forms and Validation

```csharp
<EditForm Model="@_model" OnValidSubmit="@HandleSubmit" FormName="CreateProduct">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div>
        <label>Name</label>
        <InputText @bind-Value="_model.Name" />
        <ValidationMessage For="@(() => _model.Name)" />
    </div>

    <div>
        <label>Price</label>
        <InputNumber @bind-Value="_model.Price" />
        <ValidationMessage For="@(() => _model.Price)" />
    </div>

    <button type="submit" disabled="@_isSubmitting">Create</button>
</EditForm>

@code {
    private CreateProductModel _model = new();
    private bool _isSubmitting;

    private async Task HandleSubmit()
    {
        _isSubmitting = true;
        await ProductService.CreateAsync(_model);
        _isSubmitting = false;
        Navigation.NavigateTo("/products");
    }

    public class CreateProductModel
    {
        [Required, MaxLength(200)] public string Name { get; set; } = "";
        [Range(0.01, 999999.99)] public decimal Price { get; set; }
    }
}
```

## State Management

### CascadingValue for Theme/Auth

```csharp
// App.razor
<CascadingValue Value="@_theme">
    <Router AppAssembly="typeof(App).Assembly">
        <Found Context="routeData"><RouteView RouteData="routeData" /></Found>
    </Router>
</CascadingValue>
```

### Scoped Service for Feature State

```csharp
public class CartState
{
    private readonly List<CartItem> _items = [];
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public decimal Total => _items.Sum(i => i.Price * i.Quantity);
    public event Action? OnChange;

    public void AddItem(CartItem item)
    {
        _items.Add(item);
        OnChange?.Invoke();
    }
}

// Register as scoped (per-circuit in Server, per-tab in WASM)
builder.Services.AddScoped<CartState>();
```

## Generic Components

```csharp
@typeparam TItem

<table>
    <thead><tr>@HeaderTemplate</tr></thead>
    <tbody>
        @foreach (var item in Items)
        {
            <tr>@RowTemplate(item)</tr>
        }
    </tbody>
</table>

@code {
    [Parameter, EditorRequired] public IReadOnlyList<TItem> Items { get; set; } = [];
    [Parameter, EditorRequired] public RenderFragment HeaderTemplate { get; set; } = default!;
    [Parameter, EditorRequired] public RenderFragment<TItem> RowTemplate { get; set; } = default!;
}

// Usage
<DataTable TItem="Product" Items="@_products">
    <HeaderTemplate><th>Name</th><th>Price</th></HeaderTemplate>
    <RowTemplate Context="product">
        <td>@product.Name</td><td>@product.Price.ToString("C")</td>
    </RowTemplate>
</DataTable>
```

## JavaScript Interop

```csharp
// Module isolation (recommended)
private IJSObjectReference? _module;

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
        _module = await JS.InvokeAsync<IJSObjectReference>("import", "./Components/Chart.razor.js");
}

private async Task UpdateChart(object data)
{
    if (_module is not null)
        await _module.InvokeVoidAsync("updateChart", _chartElement, data);
}

public async ValueTask DisposeAsync()
{
    if (_module is not null) await _module.DisposeAsync();
}
```

## Authentication

```csharp
<AuthorizeView>
    <Authorized>
        <p>Hello, @context.User.Identity?.Name!</p>
        <button @onclick="Logout">Logout</button>
    </Authorized>
    <NotAuthorized>
        <a href="/login">Login</a>
    </NotAuthorized>
</AuthorizeView>

// Role-based
<AuthorizeView Roles="Admin">
    <Authorized><AdminPanel /></Authorized>
</AuthorizeView>
```

## References

- [Blazor Patterns](references/blazor-patterns.md) — Component communication, render fragments, Virtualize, error boundaries, lazy loading, CSS isolation.
- [Blazor JS Interop](references/blazor-interop.md) — JS interop patterns, module isolation, .NET from JS, DOM manipulation, third-party libraries.
