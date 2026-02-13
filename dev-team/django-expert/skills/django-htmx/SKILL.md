---
name: Django HTMX
description: This skill should be used when the user asks about "htmx", "django-htmx", "htmx Django", "partial templates", "html over the wire", "htmx swap", "htmx trigger", "inline editing", "htmx form", "htmx pagination", "htmx search", "active search", "htmx modal", "htmx delete", "htmx infinite scroll", "hx-get", "hx-post", "hx-swap", "hx-target", or "hypermedia". It covers HTMX integration with Django using the django-htmx package, partial template rendering, swap strategies, inline editing, active search, infinite scroll, modal dialogs, form handling, out-of-band swaps, and progressive enhancement. Use this skill for adding HTMX interactivity to Django templates, building dynamic UIs without JavaScript frameworks, handling HTMX requests in Django views, designing partial template architectures, or implementing common HTMX patterns like click-to-edit, live search, and lazy loading.
---

## Overview

HTMX lets you build dynamic, interactive Django applications by returning HTML fragments from the server instead of JSON. Combined with django-htmx, you get a clean server-side architecture where Django views return partial templates and HTMX swaps them into the page. No JavaScript framework required.

The core idea: Django views detect HTMX requests via `request.htmx`, return a partial template for HTMX requests or a full page for normal requests, and HTMX attributes on HTML elements control what gets requested and where the response is placed.

## Installation and Setup

Install the packages:

```bash
pip install django-htmx
```

Configure Django settings and middleware:

```python
# settings.py
INSTALLED_APPS = [
    # ... existing apps ...
    "django_htmx",
]

MIDDLEWARE = [
    # ... existing middleware ...
    "django_htmx.middleware.HtmxMiddleware",
]
```

Include htmx in your base template:

```html+django
{# templates/base.html #}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>{% block title %}My App{% endblock %}</title>
    <script src="https://unpkg.com/htmx.org@2.0.4"></script>
    {% block extra_head %}{% endblock %}
</head>
<body>
    {% block body %}{% endblock %}
</body>
</html>
```

For production, vendor the htmx script as a static file rather than loading from a CDN.

## The Core Pattern: Full Page vs Partial Response

Every HTMX-enabled view follows this pattern: return a full page for normal requests, return only the changed fragment for HTMX requests.

```python
# views.py
from django.shortcuts import render

def product_list(request):
    products = Product.objects.select_related("category").all()

    if request.htmx:
        template = "products/_product_list.html"
    else:
        template = "products/product_list.html"

    return render(request, template, {"products": products})
```

```html+django
{# templates/products/product_list.html #}
{% extends "base.html" %}

{% block body %}
<h1>Products</h1>
<div id="product-list">
    {% include "products/_product_list.html" %}
</div>
{% endblock %}
```

```html+django
{# templates/products/_product_list.html #}
{% for product in products %}
<div class="product-card" id="product-{{ product.pk }}">
    <h3>{{ product.name }}</h3>
    <p>{{ product.description }}</p>
    <span>${{ product.price }}</span>
</div>
{% empty %}
<p>No products found.</p>
{% endfor %}
```

Prefix partial templates with `_` to distinguish them from full-page templates.

## HTMX Attributes Quick Reference

The essential attributes you use in Django templates:

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `hx-get` | Issue GET request | `hx-get="{% url 'product-list' %}"` |
| `hx-post` | Issue POST request | `hx-post="{% url 'product-create' %}"` |
| `hx-put` | Issue PUT request | `hx-put="{% url 'product-update' pk=product.pk %}"` |
| `hx-delete` | Issue DELETE request | `hx-delete="{% url 'product-delete' pk=product.pk %}"` |
| `hx-target` | Where to place response | `hx-target="#product-list"` |
| `hx-swap` | How to place response | `hx-swap="outerHTML"`, `innerHTML`, `beforeend`, `afterend`, `delete` |
| `hx-trigger` | What event fires request | `hx-trigger="click"`, `keyup changed delay:300ms` |
| `hx-push-url` | Update browser URL | `hx-push-url="true"` |
| `hx-confirm` | Confirmation dialog | `hx-confirm="Delete this item?"` |
| `hx-indicator` | Loading indicator | `hx-indicator="#spinner"` |
| `hx-vals` | Extra values to send | `hx-vals='{"key": "value"}'` |
| `hx-boost` | Progressive enhancement | `hx-boost="true"` on links/forms |

Always use Django's `{% url %}` tag for URL values. Never hardcode URLs in `hx-get` or `hx-post`.

## The request.htmx Object

The `django_htmx.middleware.HtmxMiddleware` adds an `htmx` attribute to every request. Use it to detect and inspect HTMX requests:

```python
def my_view(request):
    # Boolean check: is this an HTMX request?
    if request.htmx:
        ...

    # Available attributes:
    request.htmx.boosted      # bool - came from hx-boost element
    request.htmx.current_url  # str | None - browser URL when request was made
    request.htmx.target       # str | None - id of target element
    request.htmx.trigger      # str | None - id of triggered element
    request.htmx.trigger_name # str | None - name of triggered element
    request.htmx.prompt       # str | None - user response to hx-prompt
```

## CSRF Protection

HTMX POST/PUT/DELETE requests need the CSRF token. Configure htmx to include it automatically:

```html+django
{# templates/base.html - add to <body> tag #}
<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>
```

This sets the CSRF token header on all HTMX requests within the body, matching Django's `CsrfViewMiddleware` expectations.

## Common Patterns

### Click-to-Edit (Inline Editing)

Display a value that becomes an editable form on click, then saves and returns to display mode.

```python
# views.py
def product_detail_partial(request, pk):
    product = get_object_or_404(Product, pk=pk)
    return render(request, "products/_product_row.html", {"product": product})


def product_edit_form(request, pk):
    product = get_object_or_404(Product, pk=pk)
    if request.method == "POST":
        form = ProductForm(request.POST, instance=product)
        if form.is_valid():
            form.save()
            return render(request, "products/_product_row.html", {"product": product})
    else:
        form = ProductForm(instance=product)
    return render(request, "products/_product_edit_form.html", {
        "form": form, "product": product,
    })
```

```html+django
{# templates/products/_product_row.html #}
<tr id="product-{{ product.pk }}">
    <td>{{ product.name }}</td>
    <td>${{ product.price }}</td>
    <td>
        <button hx-get="{% url 'product-edit-form' product.pk %}"
                hx-target="#product-{{ product.pk }}"
                hx-swap="outerHTML">
            Edit
        </button>
    </td>
</tr>
```

```html+django
{# templates/products/_product_edit_form.html #}
<tr id="product-{{ product.pk }}">
    <td colspan="3">
        <form hx-post="{% url 'product-edit-form' product.pk %}"
              hx-target="#product-{{ product.pk }}"
              hx-swap="outerHTML">
            {% csrf_token %}
            {{ form.as_div }}
            <button type="submit">Save</button>
            <button hx-get="{% url 'product-detail-partial' product.pk %}"
                    hx-target="#product-{{ product.pk }}"
                    hx-swap="outerHTML">
                Cancel
            </button>
        </form>
    </td>
</tr>
```

### Active Search (Live Search)

Search results update as the user types, with a debounce delay.

```python
# views.py
def product_search(request):
    query = request.GET.get("q", "")
    products = Product.objects.all()
    if query:
        products = products.filter(name__icontains=query)

    if request.htmx:
        return render(request, "products/_search_results.html", {"products": products})
    return render(request, "products/product_search.html", {
        "products": products, "query": query,
    })
```

```html+django
{# templates/products/product_search.html #}
{% extends "base.html" %}

{% block body %}
<h1>Search Products</h1>
<input type="search"
       name="q"
       value="{{ query }}"
       placeholder="Search products..."
       hx-get="{% url 'product-search' %}"
       hx-trigger="keyup changed delay:300ms"
       hx-target="#search-results"
       hx-push-url="true"
       hx-indicator="#search-spinner">
<span id="search-spinner" class="htmx-indicator">Searching...</span>
<div id="search-results">
    {% include "products/_search_results.html" %}
</div>
{% endblock %}
```

### Infinite Scroll / Load More

Load the next page of results when the user scrolls to the bottom or clicks "Load more".

```python
# views.py
from django.core.paginator import Paginator

def product_list(request):
    paginator = Paginator(Product.objects.all(), 20)
    page = paginator.get_page(request.GET.get("page", 1))

    if request.htmx:
        return render(request, "products/_product_page.html", {"page_obj": page})
    return render(request, "products/product_list.html", {"page_obj": page})
```

```html+django
{# templates/products/_product_page.html #}
{% for product in page_obj %}
<div class="product-card">
    <h3>{{ product.name }}</h3>
    <p>${{ product.price }}</p>
</div>
{% endfor %}

{% if page_obj.has_next %}
<div hx-get="{% url 'product-list' %}?page={{ page_obj.next_page_number }}"
     hx-trigger="revealed"
     hx-swap="outerHTML"
     hx-indicator="#load-spinner">
    <span id="load-spinner" class="htmx-indicator">Loading more...</span>
</div>
{% endif %}
```

The `hx-trigger="revealed"` fires when the element scrolls into view, creating infinite scroll behavior. The sentinel div replaces itself with the next page of results plus a new sentinel.

### Delete with Confirmation

```html+django
<button hx-delete="{% url 'product-delete' product.pk %}"
        hx-target="#product-{{ product.pk }}"
        hx-swap="outerHTML swap:500ms"
        hx-confirm="Delete {{ product.name }}?">
    Delete
</button>
```

```python
# views.py
from django.http import HttpResponse

def product_delete(request, pk):
    product = get_object_or_404(Product, pk=pk)
    product.delete()
    return HttpResponse("")  # Empty response removes the target element
```

### Form Submission with Validation Errors

Return the form partial with errors for invalid submissions, or swap in the success content.

```python
# views.py
def product_create(request):
    if request.method == "POST":
        form = ProductForm(request.POST)
        if form.is_valid():
            product = form.save()
            return render(request, "products/_product_row.html", {"product": product})
    else:
        form = ProductForm()
    return render(request, "products/_product_form.html", {"form": form})
```

```html+django
{# templates/products/_product_form.html #}
<form hx-post="{% url 'product-create' %}"
      hx-target="#product-table-body"
      hx-swap="beforeend">
    {% csrf_token %}
    {{ form.as_div }}
    <button type="submit">Create</button>
</form>
```

When validation fails, the server re-renders the form partial with error messages. Override `hx-target` on the form to replace itself when errors occur:

```html+django
<form id="product-form"
      hx-post="{% url 'product-create' %}"
      hx-target="#product-table-body"
      hx-swap="beforeend"
      hx-target-error="#product-form"
      hx-swap-error="outerHTML">
```

## HTMX Response Headers

Use django-htmx response classes to control client-side behavior from the server:

```python
from django_htmx.http import (
    HttpResponseClientRedirect,
    HttpResponseClientRefresh,
    HttpResponseStopPolling,
    trigger_client_event,
)

def product_create(request):
    form = ProductForm(request.POST)
    if form.is_valid():
        form.save()
        # Redirect the browser (full page navigation)
        return HttpResponseClientRedirect(reverse("product-list"))

def save_and_notify(request):
    # ... save logic ...
    response = render(request, "products/_product_row.html", {"product": product})
    # Trigger a custom event on the client
    trigger_client_event(response, "showNotification", {"message": "Product saved!"})
    return response
```

## Progressive Enhancement with hx-boost

Add `hx-boost="true"` to a container to automatically convert all links and forms inside it to HTMX requests. The page still works without JavaScript.

```html+django
<div hx-boost="true">
    <a href="{% url 'product-detail' product.pk %}">{{ product.name }}</a>
    <form method="post" action="{% url 'product-create' %}">
        {% csrf_token %}
        {{ form.as_div }}
        <button type="submit">Create</button>
    </form>
</div>
```

Boosted requests swap the entire `<body>` by default and update the browser URL. Your views need no changes since they already return full pages.

## Out-of-Band (OOB) Swaps

Update multiple parts of the page from a single response by including extra elements with `hx-swap-oob`:

```python
# views.py
def add_to_cart(request, pk):
    product = get_object_or_404(Product, pk=pk)
    cart = request.session.get("cart", [])
    cart.append(pk)
    request.session["cart"] = cart
    return render(request, "cart/_cart_update.html", {
        "product": product,
        "cart_count": len(cart),
    })
```

```html+django
{# templates/cart/_cart_update.html #}
{# Primary swap target: confirmation message #}
<div id="add-confirmation">
    Added {{ product.name }} to cart!
</div>

{# OOB swap: update cart badge in the navbar #}
<span id="cart-count" hx-swap-oob="true">{{ cart_count }}</span>
```

The primary content swaps into the target specified by `hx-target`. Any element with `hx-swap-oob="true"` is matched by `id` and swapped independently anywhere on the page.

## Template Organization

```
templates/
    base.html
    products/
        product_list.html          # Full page (extends base.html)
        product_detail.html        # Full page
        product_search.html        # Full page
        _product_list.html         # Partial: product list fragment
        _product_row.html          # Partial: single product row
        _product_form.html         # Partial: product form
        _product_edit_form.html    # Partial: inline edit form
        _search_results.html       # Partial: search results
```

Conventions:
- Full-page templates extend `base.html` and include partials with `{% include %}`.
- Partial templates are prefixed with `_` and never extend a base template.
- Each partial has a single root element with a stable `id` for targeting.
- Partials are self-contained: they contain all the `hx-*` attributes needed for their own interactions.

---

## References

For django-htmx configuration, middleware details, response classes, and all `request.htmx` attributes, see [htmx-configuration.md](references/htmx-configuration.md).

For advanced patterns including modal dialogs, lazy loading, polling, sortable tables, bulk actions, inline validation, and Django CBV integration, see [htmx-patterns.md](references/htmx-patterns.md).
