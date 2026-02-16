# Blazor Component Patterns

## Component Communication

### Parent → Child (Parameter)
```razor
<ProductCard Product="@_product" OnAddToCart="HandleAddToCart" />

@code {
    [Parameter, EditorRequired] public Product Product { get; set; } = default!;
    [Parameter] public EventCallback<Product> OnAddToCart { get; set; }
}
```

### Cascading Values
```razor
<CascadingValue Value="@_theme" Name="AppTheme">
    <Router>...</Router>
</CascadingValue>

// In child:
[CascadingParameter(Name = "AppTheme")] public Theme Theme { get; set; } = default!;
```

## Render Fragments

```razor
@typeparam TItem

<div class="card">
    <div class="card-header">@HeaderContent</div>
    <div class="card-body">
        @foreach (var item in Items)
        {
            @ItemTemplate(item)
        }
    </div>
    <div class="card-footer">@FooterContent</div>
</div>

@code {
    [Parameter, EditorRequired] public RenderFragment HeaderContent { get; set; } = default!;
    [Parameter, EditorRequired] public RenderFragment<TItem> ItemTemplate { get; set; } = default!;
    [Parameter] public RenderFragment? FooterContent { get; set; }
    [Parameter, EditorRequired] public IReadOnlyList<TItem> Items { get; set; } = [];
}
```

## Virtualization

```razor
<Virtualize Items="@_products" Context="product" ItemSize="50">
    <ItemContent>
        <div class="product-row">@product.Name — @product.Price.ToString("C")</div>
    </ItemContent>
    <Placeholder>
        <div class="loading-row">Loading...</div>
    </Placeholder>
</Virtualize>

// Or with ItemsProvider for server-side pagination:
<Virtualize ItemsProviderRequest="LoadProducts" ItemSize="50">...</Virtualize>

@code {
    private async ValueTask<ItemsProviderResult<Product>> LoadProducts(ItemsProviderRequest request)
    {
        var result = await ProductService.GetPageAsync(request.StartIndex, request.Count, request.CancellationToken);
        return new ItemsProviderResult<Product>(result.Items, result.TotalCount);
    }
}
```

## Error Boundaries

```razor
<ErrorBoundary @ref="_errorBoundary">
    <ChildContent><RiskyComponent /></ChildContent>
    <ErrorContent Context="ex">
        <div class="alert alert-danger">
            <p>Something went wrong: @ex.Message</p>
            <button @onclick="() => _errorBoundary.Recover()">Try Again</button>
        </div>
    </ErrorContent>
</ErrorBoundary>

@code { private ErrorBoundary _errorBoundary = default!; }
```

## CSS Isolation

```css
/* ProductCard.razor.css — scoped to this component */
.card { border: 1px solid #ddd; }
::deep .child-element { color: blue; } /* Affect child components */
```

## Lazy Loading Assemblies

```csharp
// In Router
<Router AppAssembly="typeof(App).Assembly" AdditionalAssemblies="@_lazyLoadedAssemblies" OnNavigateAsync="OnNavigateAsync">

@code {
    private List<Assembly> _lazyLoadedAssemblies = [];
    private async Task OnNavigateAsync(NavigationContext context)
    {
        if (context.Path == "admin")
        {
            var assemblies = await AssemblyLoader.LoadAssembliesAsync(new[] { "AdminModule.wasm" });
            _lazyLoadedAssemblies.AddRange(assemblies);
        }
    }
}
```
