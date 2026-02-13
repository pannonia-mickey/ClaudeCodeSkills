# Django HTMX Advanced Patterns

This reference covers advanced HTMX patterns for Django: modal dialogs, lazy loading, polling, sortable tables, bulk actions, inline validation, Django CBV integration, toast notifications, and testing strategies.

---

## Modal Dialogs

Render modals server-side and swap them into the page on demand.

### Setup: Modal Container

Add an empty modal container to your base template:

```html+django
{# templates/base.html #}
<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
    {% block body %}{% endblock %}

    {# Modal container - always present, empty by default #}
    <div id="modal-container"></div>
</body>
```

### Triggering a Modal

```html+django
<button hx-get="{% url 'product-create-modal' %}"
        hx-target="#modal-container"
        hx-swap="innerHTML">
    New Product
</button>
```

### Modal Template

```html+django
{# templates/products/_product_modal.html #}
<div class="modal-backdrop" onclick="this.remove()">
    <div class="modal" onclick="event.stopPropagation()">
        <div class="modal-header">
            <h2>{{ title }}</h2>
            <button onclick="this.closest('.modal-backdrop').remove()">&times;</button>
        </div>
        <div class="modal-body">
            <form hx-post="{% url 'product-create' %}"
                  hx-target="#product-list"
                  hx-swap="beforeend"
                  hx-on::after-request="if(event.detail.successful) this.closest('.modal-backdrop').remove()">
                {% csrf_token %}
                {{ form.as_div }}
                <button type="submit">Create</button>
            </form>
        </div>
    </div>
</div>
```

### Modal View

```python
def product_create_modal(request):
    form = ProductForm()
    return render(request, "products/_product_modal.html", {
        "form": form,
        "title": "New Product",
    })

def product_create(request):
    form = ProductForm(request.POST)
    if form.is_valid():
        product = form.save()
        return render(request, "products/_product_row.html", {"product": product})
    # Re-render the modal with errors
    return render(request, "products/_product_modal.html", {
        "form": form,
        "title": "New Product",
    })
```

## Lazy Loading

Load expensive content after the page renders using `hx-trigger="load"`.

```html+django
{# Main page loads fast, expensive sections load asynchronously #}
{% extends "base.html" %}

{% block body %}
<h1>Dashboard</h1>

<div hx-get="{% url 'dashboard-stats' %}"
     hx-trigger="load"
     hx-indicator="#stats-spinner">
    <span id="stats-spinner" class="htmx-indicator">Loading stats...</span>
</div>

<div hx-get="{% url 'dashboard-recent-activity' %}"
     hx-trigger="load"
     hx-indicator="#activity-spinner">
    <span id="activity-spinner" class="htmx-indicator">Loading activity...</span>
</div>

<div hx-get="{% url 'dashboard-charts' %}"
     hx-trigger="load delay:500ms"
     hx-indicator="#charts-spinner">
    <span id="charts-spinner" class="htmx-indicator">Loading charts...</span>
</div>
{% endblock %}
```

```python
def dashboard_stats(request):
    # Expensive aggregation query
    stats = Order.objects.aggregate(
        total_revenue=Sum("total"),
        order_count=Count("id"),
    )
    return render(request, "dashboard/_stats.html", {"stats": stats})
```

Use `delay:` in `hx-trigger` to stagger requests and avoid overwhelming the server.

## Polling

Periodically refresh content using `hx-trigger="every Ns"`.

```html+django
{# Poll every 5 seconds, stops when task is complete #}
<div id="task-status"
     hx-get="{% url 'task-status' task.pk %}"
     hx-trigger="every 5s"
     hx-swap="outerHTML">
    <p>Status: {{ task.status }}</p>
    <div class="progress-bar" style="width: {{ task.progress }}%"></div>
</div>
```

```python
from django_htmx.http import HttpResponseStopPolling

def task_status(request, pk):
    task = get_object_or_404(Task, pk=pk)
    if task.status == "complete":
        # Return 286 to stop polling
        return HttpResponseStopPolling()
    return render(request, "tasks/_task_status.html", {"task": task})
```

### Conditional Polling

Start polling only when a condition is met:

```html+django
<div hx-get="{% url 'notifications' %}"
     hx-trigger="every 30s [document.visibilityState === 'visible']"
     hx-swap="innerHTML">
</div>
```

## Sortable Tables

Column sorting with URL state management.

```python
def product_list(request):
    sort = request.GET.get("sort", "name")
    direction = request.GET.get("dir", "asc")

    order_field = sort if direction == "asc" else f"-{sort}"
    products = Product.objects.order_by(order_field)

    template = "products/_product_table.html" if request.htmx else "products/product_list.html"
    return render(request, template, {
        "products": products,
        "current_sort": sort,
        "current_dir": direction,
    })
```

```html+django
{# templates/products/_product_table.html #}
<table id="product-table">
    <thead>
        <tr>
            {% for col in "name,price,created_at"|split:"," %}
            <th>
                {% if current_sort == col and current_dir == "asc" %}
                <a hx-get="{% url 'product-list' %}?sort={{ col }}&dir=desc"
                   hx-target="#product-table"
                   hx-swap="outerHTML"
                   hx-push-url="true">
                    {{ col|title }} ▲
                </a>
                {% else %}
                <a hx-get="{% url 'product-list' %}?sort={{ col }}&dir=asc"
                   hx-target="#product-table"
                   hx-swap="outerHTML"
                   hx-push-url="true">
                    {{ col|title }} {% if current_sort == col %}▼{% endif %}
                </a>
                {% endif %}
            </th>
            {% endfor %}
        </tr>
    </thead>
    <tbody>
        {% for product in products %}
        <tr id="product-{{ product.pk }}">
            <td>{{ product.name }}</td>
            <td>${{ product.price }}</td>
            <td>{{ product.created_at|date:"Y-m-d" }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
```

## Bulk Actions

Select multiple items and perform bulk operations.

```html+django
{# templates/products/_product_table_with_bulk.html #}
<form id="bulk-form">
    <div class="bulk-actions">
        <select name="action">
            <option value="">-- Bulk Actions --</option>
            <option value="delete">Delete Selected</option>
            <option value="archive">Archive Selected</option>
            <option value="publish">Publish Selected</option>
        </select>
        <button hx-post="{% url 'product-bulk-action' %}"
                hx-include="#bulk-form"
                hx-target="#product-table-body"
                hx-swap="innerHTML"
                hx-confirm="Apply action to selected items?">
            Apply
        </button>
    </div>

    <table>
        <thead>
            <tr>
                <th><input type="checkbox" onclick="toggleAll(this)"></th>
                <th>Name</th>
                <th>Price</th>
            </tr>
        </thead>
        <tbody id="product-table-body">
            {% for product in products %}
            <tr>
                <td><input type="checkbox" name="selected" value="{{ product.pk }}"></td>
                <td>{{ product.name }}</td>
                <td>${{ product.price }}</td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
</form>

<script>
function toggleAll(source) {
    document.querySelectorAll('input[name="selected"]').forEach(cb => {
        cb.checked = source.checked;
    });
}
</script>
```

```python
def product_bulk_action(request):
    ids = request.POST.getlist("selected")
    action = request.POST.get("action")
    products = Product.objects.filter(pk__in=ids)

    if action == "delete":
        products.delete()
    elif action == "archive":
        products.update(status="archived")
    elif action == "publish":
        products.update(status="published")

    # Re-render the table body
    all_products = Product.objects.all()
    return render(request, "products/_product_table_body.html", {
        "products": all_products,
    })
```

## Inline Form Validation

Validate individual fields as the user types.

```html+django
<form hx-post="{% url 'product-create' %}" hx-target="#product-list" hx-swap="beforeend">
    {% csrf_token %}

    <div id="field-name">
        <label for="name">Name</label>
        <input type="text" name="name" id="name"
               hx-post="{% url 'validate-product-field' %}"
               hx-trigger="blur"
               hx-target="#field-name-errors"
               hx-swap="innerHTML"
               hx-vals='{"field": "name"}'>
        <div id="field-name-errors"></div>
    </div>

    <div id="field-price">
        <label for="price">Price</label>
        <input type="number" name="price" id="price" step="0.01"
               hx-post="{% url 'validate-product-field' %}"
               hx-trigger="blur"
               hx-target="#field-price-errors"
               hx-swap="innerHTML"
               hx-vals='{"field": "price"}'>
        <div id="field-price-errors"></div>
    </div>

    <button type="submit">Create</button>
</form>
```

```python
def validate_product_field(request):
    field = request.POST.get("field")
    form = ProductForm(request.POST)
    form.is_valid()  # Run validation

    errors = form.errors.get(field, [])
    if errors:
        html = "".join(f'<p class="error">{e}</p>' for e in errors)
        return HttpResponse(html)
    return HttpResponse('<p class="valid">Looks good!</p>')
```

## Django Class-Based Views with HTMX

### Mixin for HTMX-Aware CBVs

```python
class HtmxMixin:
    """Mixin that returns a partial template for HTMX requests."""
    partial_template_name = None

    def get_template_names(self):
        if self.request.htmx and self.partial_template_name:
            return [self.partial_template_name]
        return super().get_template_names()
```

### ListView with HTMX

```python
from django.views.generic import ListView

class ProductListView(HtmxMixin, ListView):
    model = Product
    template_name = "products/product_list.html"
    partial_template_name = "products/_product_list.html"
    context_object_name = "products"
    paginate_by = 20

    def get_queryset(self):
        qs = super().get_queryset().select_related("category")
        query = self.request.GET.get("q")
        if query:
            qs = qs.filter(name__icontains=query)
        return qs
```

### CreateView with HTMX

```python
from django.views.generic import CreateView
from django_htmx.http import HttpResponseClientRedirect, trigger_client_event

class ProductCreateView(CreateView):
    model = Product
    form_class = ProductForm
    template_name = "products/_product_form.html"

    def form_valid(self, form):
        self.object = form.save()
        if self.request.htmx:
            response = render(
                self.request,
                "products/_product_row.html",
                {"product": self.object},
            )
            trigger_client_event(response, "productCreated")
            return response
        return HttpResponseClientRedirect(self.object.get_absolute_url())

    def form_invalid(self, form):
        return render(self.request, self.template_name, {"form": form})
```

### DeleteView with HTMX

```python
from django.views.generic import DeleteView

class ProductDeleteView(DeleteView):
    model = Product

    def delete(self, request, *args, **kwargs):
        self.object = self.get_object()
        self.object.delete()
        if request.htmx:
            return HttpResponse("")  # Empty response removes the target
        return HttpResponseClientRedirect(reverse("product-list"))
```

## Toast Notifications

Display temporary success/error messages triggered from the server.

### Base Template Setup

```html+django
{# templates/base.html #}
<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
    {% block body %}{% endblock %}

    <div id="toast-container"></div>
    <div id="modal-container"></div>
</body>
```

### Toast Partial

```html+django
{# templates/_toast.html #}
<div class="toast toast-{{ level }}"
     style="animation: fadeInOut 3s forwards">
    {{ message }}
</div>
```

### Triggering Toasts via OOB Swap

```python
def product_save(request, pk):
    product = get_object_or_404(Product, pk=pk)
    form = ProductForm(request.POST, instance=product)
    if form.is_valid():
        form.save()
        response = render(request, "products/_product_row.html", {"product": product})
        # Append a toast via OOB swap
        toast_html = render_to_string("_toast.html", {
            "message": "Product saved successfully!",
            "level": "success",
        })
        response.content += f'<div id="toast-container" hx-swap-oob="beforeend">{toast_html}</div>'.encode()
        return response
    return render(request, "products/_product_form.html", {"form": form})
```

### Alternative: Trigger Client Event

```python
from django_htmx.http import trigger_client_event

def product_save(request, pk):
    ...
    response = render(request, "products/_product_row.html", {"product": product})
    trigger_client_event(response, "showToast", {
        "message": "Product saved!",
        "level": "success",
    })
    return response
```

```javascript
document.body.addEventListener("showToast", function(event) {
    const { message, level } = event.detail;
    const container = document.getElementById("toast-container");
    const toast = document.createElement("div");
    toast.className = `toast toast-${level}`;
    toast.textContent = message;
    container.appendChild(toast);
    setTimeout(() => toast.remove(), 3000);
});
```

## Tabs / Tab Navigation

Server-rendered tabs where each tab loads content via HTMX.

```html+django
{# templates/products/product_detail.html #}
{% extends "base.html" %}

{% block body %}
<h1>{{ product.name }}</h1>

<div class="tabs">
    <button class="tab active"
            hx-get="{% url 'product-overview' product.pk %}"
            hx-target="#tab-content"
            hx-swap="innerHTML"
            hx-push-url="true">
        Overview
    </button>
    <button class="tab"
            hx-get="{% url 'product-reviews' product.pk %}"
            hx-target="#tab-content"
            hx-swap="innerHTML"
            hx-push-url="true">
        Reviews
    </button>
    <button class="tab"
            hx-get="{% url 'product-related' product.pk %}"
            hx-target="#tab-content"
            hx-swap="innerHTML"
            hx-push-url="true">
        Related
    </button>
</div>

<div id="tab-content">
    {% include "products/_product_overview.html" %}
</div>
{% endblock %}
```

Use `hx-on::after-request` or CSS classes with `hx-indicator` to toggle the active tab state.

## Dependent Dropdowns

The second dropdown updates based on the first dropdown's selection.

```html+django
<form>
    <select name="country"
            hx-get="{% url 'load-cities' %}"
            hx-target="#city-select"
            hx-swap="innerHTML"
            hx-trigger="change"
            hx-indicator="#city-spinner">
        <option value="">Select Country</option>
        {% for country in countries %}
        <option value="{{ country.pk }}">{{ country.name }}</option>
        {% endfor %}
    </select>

    <span id="city-spinner" class="htmx-indicator">Loading...</span>
    <select name="city" id="city-select">
        <option value="">Select City</option>
    </select>
</form>
```

```python
def load_cities(request):
    country_id = request.GET.get("country")
    cities = City.objects.filter(country_id=country_id) if country_id else City.objects.none()
    return render(request, "locations/_city_options.html", {"cities": cities})
```

```html+django
{# templates/locations/_city_options.html #}
<option value="">Select City</option>
{% for city in cities %}
<option value="{{ city.pk }}">{{ city.name }}</option>
{% endfor %}
```

## Toggle / Expand-Collapse

```html+django
<div id="section-{{ section.pk }}">
    <h3>
        <button hx-get="{% url 'section-content' section.pk %}"
                hx-target="#section-body-{{ section.pk }}"
                hx-swap="innerHTML">
            {{ section.title }}
        </button>
    </h3>
    <div id="section-body-{{ section.pk }}">
        {# Content loaded on demand #}
    </div>
</div>
```

## Testing HTMX Views

### Unit Testing with Django Test Client

```python
from django.test import TestCase

class ProductListHTMXTest(TestCase):
    def test_full_page_request(self):
        response = self.client.get("/products/")
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, "products/product_list.html")

    def test_htmx_request_returns_partial(self):
        response = self.client.get(
            "/products/",
            HTTP_HX_REQUEST="true",
        )
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, "products/_product_list.html")

    def test_htmx_request_with_target(self):
        response = self.client.get(
            "/products/",
            HTTP_HX_REQUEST="true",
            HTTP_HX_TARGET="search-results",
        )
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, "products/_search_results.html")
```

### Testing with pytest-django

```python
import pytest
from django.test import RequestFactory
from django_htmx.middleware import HtmxMiddleware

@pytest.fixture
def htmx_rf():
    """Request factory that adds HTMX headers."""
    factory = RequestFactory()

    class HTMXRequestFactory:
        def get(self, path, htmx=True, **kwargs):
            headers = {}
            if htmx:
                headers["HTTP_HX_REQUEST"] = "true"
            headers.update(kwargs)
            request = factory.get(path, **headers)
            middleware = HtmxMiddleware(lambda r: r)
            middleware(request)
            return request

        def post(self, path, data=None, htmx=True, **kwargs):
            headers = {}
            if htmx:
                headers["HTTP_HX_REQUEST"] = "true"
            headers.update(kwargs)
            request = factory.post(path, data=data or {}, **headers)
            middleware = HtmxMiddleware(lambda r: r)
            middleware(request)
            return request

    return HTMXRequestFactory()

def test_product_list_htmx(htmx_rf, product_factory):
    product_factory.create_batch(5)
    request = htmx_rf.get("/products/")
    response = product_list(request)
    assert response.status_code == 200

def test_product_list_non_htmx(htmx_rf, product_factory):
    request = htmx_rf.get("/products/", htmx=False)
    response = product_list(request)
    assert response.status_code == 200
```

### Testing HTMX Response Headers

```python
def test_client_redirect_response(self):
    response = self.client.post(
        "/products/create/",
        data={"name": "Widget", "price": "9.99"},
        HTTP_HX_REQUEST="true",
    )
    self.assertEqual(response.headers.get("HX-Redirect"), "/products/")

def test_trigger_event_response(self):
    response = self.client.post(
        f"/products/{self.product.pk}/save/",
        data={"name": "Updated"},
        HTTP_HX_REQUEST="true",
    )
    self.assertIn("HX-Trigger", response.headers)
```

### Browser Testing with Playwright

For end-to-end tests that verify HTMX interactions in a real browser:

```python
from playwright.sync_api import expect

def test_inline_edit(live_server, page, product):
    page.goto(f"{live_server.url}/products/")

    # Click edit button
    page.click(f"#product-{product.pk} button:text('Edit')")

    # Wait for the form to appear
    form = page.locator(f"#product-{product.pk} form")
    expect(form).to_be_visible()

    # Fill and submit
    form.locator("input[name='name']").fill("Updated Name")
    form.locator("button[type='submit']").click()

    # Verify the row updated
    row = page.locator(f"#product-{product.pk}")
    expect(row).to_contain_text("Updated Name")
```

## Performance Considerations

- **Minimize partial size**: Return only the HTML that changed. Smaller responses mean faster swaps.
- **Use `hx-swap="outerHTML"`**: Replacing the entire target element (including its `id`) keeps the DOM clean. Use `innerHTML` only when appending to a container.
- **Add `hx-indicator`**: Always show loading states for requests that might be slow. Users should never stare at a frozen page.
- **Debounce inputs**: Use `hx-trigger="keyup changed delay:300ms"` for search inputs to avoid flooding the server.
- **Use `hx-push-url`**: Maintain browser history for navigational requests so users can use back/forward buttons and share URLs.
- **Cache partials**: Use Django's `cache_page` or fragment caching on expensive partial views.
- **Avoid deep nesting**: Keep your partial template hierarchy flat. A partial should not include another partial that triggers another HTMX request that loads another partial.
