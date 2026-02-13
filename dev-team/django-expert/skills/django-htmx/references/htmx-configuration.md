# django-htmx Configuration Reference

This reference covers django-htmx installation, middleware configuration, request attributes, response classes, and htmx script setup for Django projects.

---

## Installation

```bash
pip install django-htmx
```

Requires Django 4.2+ and Python 3.9+.

## Django Settings

```python
INSTALLED_APPS = [
    # ... existing apps ...
    "django_htmx",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    # Add after Django's built-in middleware
    "django_htmx.middleware.HtmxMiddleware",
]
```

The `HtmxMiddleware` adds the `request.htmx` attribute to every request. Place it after Django's standard middleware.

## Including htmx in Templates

### Option 1: CDN (Development)

```html+django
<script src="https://unpkg.com/htmx.org@2.0.4"></script>
```

### Option 2: Static File (Production)

Download htmx.min.js and serve it as a Django static file:

```html+django
{% load static %}
<script src="{% static 'js/htmx.min.js' %}"></script>
```

### Option 3: django-htmx Script Tag

django-htmx does not bundle the htmx script itself. You must include it separately using one of the above methods.

## request.htmx Attribute Reference

The middleware sets `request.htmx` to an `HtmxDetails` instance on every request. For non-HTMX requests, `bool(request.htmx)` returns `False`.

### Boolean Check

```python
if request.htmx:
    # This is an HTMX request (HX-Request header is "true")
    ...
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `boosted` | `bool` | `True` if the request came from an `hx-boost` element. Based on `HX-Boosted` header. |
| `current_url` | `str \| None` | The browser URL when the request was made. Based on `HX-Current-URL` header. |
| `current_url_abs_path` | `str \| None` | The path portion of `current_url` (no scheme/host). Useful for `redirect()`. |
| `history_restore_request` | `bool` | `True` if this is a history restoration request. Based on `HX-History-Restore-Request` header. |
| `prompt` | `str \| None` | The user's response to `hx-prompt`, if used. Based on `HX-Prompt` header. |
| `target` | `str \| None` | The `id` of the target element. Based on `HX-Target` header. |
| `trigger` | `str \| None` | The `id` of the element that triggered the request. Based on `HX-Trigger` header. |
| `trigger_name` | `str \| None` | The `name` of the element that triggered the request. Based on `HX-Trigger-Name` header. |
| `triggering_event` | `Any \| None` | Deserialized JSON of the triggering DOM event. Requires the htmx `event-header` extension. |

### Usage in Views

```python
def product_list(request):
    products = Product.objects.all()

    if request.htmx:
        # Return partial for HTMX requests
        if request.htmx.target == "search-results":
            return render(request, "products/_search_results.html", {"products": products})
        return render(request, "products/_product_list.html", {"products": products})

    # Return full page for normal requests
    return render(request, "products/product_list.html", {"products": products})
```

### Usage in Templates

```html+django
{% if request.htmx %}
    {# Only rendered during HTMX requests #}
    <div>Loaded dynamically</div>
{% endif %}

{# Access specific attributes #}
{% if request.htmx.boosted %}
    {# Boosted navigation #}
{% endif %}
```

## Response Classes

django-htmx provides response classes that set htmx-specific response headers.

### HttpResponseClientRedirect

Tells htmx to perform a full page redirect (client-side navigation):

```python
from django_htmx.http import HttpResponseClientRedirect

def product_create(request):
    form = ProductForm(request.POST)
    if form.is_valid():
        form.save()
        return HttpResponseClientRedirect(reverse("product-list"))
    return render(request, "products/_product_form.html", {"form": form})
```

Sets the `HX-Redirect` header. The browser navigates to the URL as a full page load.

### HttpResponseClientRefresh

Tells htmx to perform a full page refresh:

```python
from django_htmx.http import HttpResponseClientRefresh

def clear_cache(request):
    cache.clear()
    return HttpResponseClientRefresh()
```

Sets the `HX-Refresh` header to `"true"`.

### HttpResponseStopPolling

Stops htmx polling by returning HTTP 286:

```python
from django_htmx.http import HttpResponseStopPolling

def poll_status(request, task_id):
    task = get_object_or_404(Task, pk=task_id)
    if task.is_complete:
        return HttpResponseStopPolling()
    return render(request, "tasks/_task_status.html", {"task": task})
```

### trigger_client_event

Triggers a custom event on the client from a server response:

```python
from django_htmx.http import trigger_client_event

def product_save(request, pk):
    product = get_object_or_404(Product, pk=pk)
    form = ProductForm(request.POST, instance=product)
    if form.is_valid():
        form.save()
        response = render(request, "products/_product_row.html", {"product": product})
        # Trigger event after swap (default)
        trigger_client_event(response, "productSaved", {"id": product.pk})
        return response
    return render(request, "products/_product_form.html", {"form": form})
```

Parameters:
- `response`: The `HttpResponse` to modify.
- `name`: The event name to trigger.
- `params`: Optional dict of event detail data.
- `after`: When to trigger. Options: `"receive"` (before swap), `"settle"` (after settle), `"swap"` (after swap, default).

The function sets `HX-Trigger`, `HX-Trigger-After-Swap`, or `HX-Trigger-After-Settle` headers accordingly.

```python
# Trigger before swap
trigger_client_event(response, "showSpinner", after="receive")

# Trigger after settle
trigger_client_event(response, "initTooltips", after="settle")
```

Listen for triggered events in the template:

```html+django
<body hx-on:productSaved="alert('Saved!')">
```

Or with JavaScript:

```javascript
document.body.addEventListener("productSaved", function(event) {
    console.log("Product saved:", event.detail.id);
});
```

### push_url and replace_url

Control browser history from responses:

```python
from django_htmx.http import push_url, replace_url

def filtered_list(request):
    response = render(request, "products/_list.html", context)
    # Push a new URL to browser history
    push_url(response, f"/products/?{request.GET.urlencode()}")
    return response

def detail_panel(request, pk):
    response = render(request, "products/_detail.html", context)
    # Replace current URL without adding history entry
    replace_url(response, f"/products/{pk}/")
    return response
```

## CSRF Configuration

### Method 1: hx-headers on body (Recommended)

```html+django
<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
```

### Method 2: Meta tag with htmx config

```html+django
<head>
    <meta name="csrf-token" content="{{ csrf_token }}">
</head>
```

```javascript
document.body.addEventListener("htmx:configRequest", function(event) {
    event.detail.headers["X-CSRFToken"] = document.querySelector(
        'meta[name="csrf-token"]'
    ).content;
});
```

### Method 3: Hidden input per form

```html+django
<form hx-post="{% url 'my-view' %}">
    {% csrf_token %}
    ...
</form>
```

htmx automatically includes form inputs in POST requests, so the standard `{% csrf_token %}` template tag works for forms.

## htmx Extensions

Load htmx extensions after the main script:

```html+django
<script src="{% static 'js/htmx.min.js' %}"></script>
<script src="https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js"></script>
```

### response-targets Extension

Allows different swap targets based on HTTP status codes. Useful for form validation errors:

```html+django
<div hx-ext="response-targets">
    <form hx-post="{% url 'product-create' %}"
          hx-target="#product-list"
          hx-swap="beforeend"
          hx-target-422="#form-container"
          hx-swap-422="innerHTML">
        {% csrf_token %}
        {{ form.as_div }}
        <button type="submit">Create</button>
    </form>
</div>
```

```python
# views.py
def product_create(request):
    form = ProductForm(request.POST)
    if form.is_valid():
        product = form.save()
        return render(request, "products/_product_row.html", {"product": product})
    # Return 422 so response-targets routes to the form container
    return render(
        request, "products/_product_form.html", {"form": form}, status=422
    )
```

### Other Useful Extensions

- **head-support**: Merges `<head>` content on page transitions (useful with `hx-boost`).
- **loading-states**: Declarative loading indicators via `data-loading` attributes.
- **preload**: Preload content on `mouseenter` for faster perceived navigation.
- **sse**: Server-Sent Events integration for real-time updates.
- **ws**: WebSocket integration for bidirectional communication.

## htmx Configuration

Configure htmx behavior globally via `meta` tags:

```html+django
<head>
    {# Default swap style for all requests #}
    <meta name="htmx-config" content='{"defaultSwapStyle": "outerHTML"}'>
</head>
```

Common configuration options:

| Option | Default | Description |
|--------|---------|-------------|
| `defaultSwapStyle` | `"innerHTML"` | Default swap strategy |
| `defaultSwapDelay` | `0` | Delay in ms before swap |
| `defaultSettleDelay` | `20` | Delay in ms before settle |
| `includeIndicatorStyles` | `true` | Include default `.htmx-indicator` CSS |
| `historyCacheSize` | `10` | Number of pages cached for history |
| `refreshOnHistoryMiss` | `false` | Full page reload on history cache miss |
| `selfRequestsOnly` | `true` | Only allow requests to same domain |
| `scrollBehavior` | `"instant"` | Scroll behavior on swap: `"smooth"` or `"instant"` |

## Debugging

Enable htmx logging in the browser console:

```javascript
htmx.logAll();
```

Or via a meta tag:

```html+django
<meta name="htmx-config" content='{"logLevel": "debug"}'>
```

Watch for events in Django views during development:

```python
import logging

logger = logging.getLogger(__name__)

def my_view(request):
    if request.htmx:
        logger.debug(
            "HTMX request: trigger=%s target=%s url=%s",
            request.htmx.trigger,
            request.htmx.target,
            request.htmx.current_url,
        )
    ...
```
