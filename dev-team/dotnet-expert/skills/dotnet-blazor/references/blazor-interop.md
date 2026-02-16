# Blazor JavaScript Interop

## Invoke JS from .NET

```csharp
// Inline call
await JS.InvokeVoidAsync("alert", "Hello from Blazor!");
var result = await JS.InvokeAsync<string>("prompt", "Enter your name:");

// Module isolation (recommended)
private IJSObjectReference? _module;

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
        _module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/chart.js");
}

private async Task RenderChart(object data)
{
    if (_module is not null)
        await _module.InvokeVoidAsync("renderChart", _canvasElement, data);
}

public async ValueTask DisposeAsync()
{
    if (_module is not null)
        await _module.DisposeAsync();
}
```

```javascript
// wwwroot/js/chart.js
export function renderChart(canvas, data) {
    const ctx = canvas.getContext('2d');
    new Chart(ctx, { type: 'bar', data: data });
}
```

## Invoke .NET from JS

```csharp
// Instance method
private DotNetObjectReference<MyComponent>? _dotNetRef;

protected override void OnInitialized() => _dotNetRef = DotNetObjectReference.Create(this);

[JSInvokable]
public async Task OnJsEvent(string data) => await ProcessAsync(data);

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
        await JS.InvokeVoidAsync("registerCallback", _dotNetRef);
}

public void Dispose() => _dotNetRef?.Dispose();
```

```javascript
function registerCallback(dotNetRef) {
    document.addEventListener('custom-event', async (e) => {
        await dotNetRef.invokeMethodAsync('OnJsEvent', e.detail);
    });
}
```

## Static .NET Method from JS

```csharp
[JSInvokable]
public static string GetAppVersion() => "1.0.0";
```

```javascript
const version = await DotNet.invokeMethodAsync('MyApp', 'GetAppVersion');
```

## DOM Manipulation

```razor
<canvas @ref="_canvasRef" width="400" height="300"></canvas>
<input @ref="_inputRef" />

@code {
    private ElementReference _canvasRef;
    private ElementReference _inputRef;

    private async Task FocusInput() => await _inputRef.FocusAsync();
}
```

## Third-Party Libraries

```csharp
// Load library dynamically
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        _leafletModule = await JS.InvokeAsync<IJSObjectReference>("import", "./js/leaflet-wrapper.js");
        await _leafletModule.InvokeVoidAsync("initMap", _mapElement, new { lat = 51.505, lng = -0.09 });
    }
}
```
