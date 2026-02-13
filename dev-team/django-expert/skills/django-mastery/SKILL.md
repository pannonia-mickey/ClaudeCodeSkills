---
name: Django Mastery
description: This skill should be used when the user asks about "Django project", "Django settings", "Django view", "Django middleware", "Django URL", "class-based view", or "Django 5". It covers Django 5.+ core patterns including project structure, settings organization, URL routing, class-based and function-based views, middleware pipeline design, and async views. Use this skill for setting up a Django project, configuring settings for multiple environments, designing URL schemes, choosing between CBV and FBV, writing custom middleware, or adopting Django 5.+ async capabilities.
---

## Project Structure

Organize Django projects with a clear separation between configuration and application code. Place the project configuration package at the top level alongside application packages.

```
myproject/
    manage.py
    config/
        __init__.py
        settings/
            __init__.py
            base.py
            development.py
            production.py
            testing.py
        urls.py
        wsgi.py
        asgi.py
    apps/
        accounts/
        products/
        orders/
    libs/
        services/
        utils/
    templates/
    static/
    tests/
```

Keep each Django app focused on a single domain concept. An app should be removable without breaking unrelated functionality. Place shared utilities in a `libs/` directory rather than creating a catch-all `common` app.

## Settings Organization

Split settings into a base module and environment-specific overrides. Load secrets from environment variables, never hardcode them.

```python
# config/settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # Third-party
    "rest_framework",
    "django_filters",
    # Local
    "apps.accounts",
    "apps.products",
]

DATABASES = {
    "default": {
        "ENGINE": os.environ.get("DB_ENGINE", "django.db.backends.postgresql"),
        "NAME": os.environ.get("DB_NAME", "myproject"),
        "USER": os.environ.get("DB_USER", "postgres"),
        "PASSWORD": os.environ.get("DB_PASSWORD", ""),
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
    }
}
```

```python
# config/settings/development.py
from .base import *  # noqa: F401,F403

DEBUG = True
ALLOWED_HOSTS = ["*"]
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE.insert(0, "debug_toolbar.middleware.DebugToolbarMiddleware")
INTERNAL_IPS = ["127.0.0.1"]
```

```python
# config/settings/production.py
from .base import *  # noqa: F401,F403

DEBUG = False
ALLOWED_HOSTS = os.environ["ALLOWED_HOSTS"].split(",")
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
```

Select the settings module via the `DJANGO_SETTINGS_MODULE` environment variable. For local development, set it to `config.settings.development`.

## URL Routing

Organize URLs hierarchically. The root URL configuration delegates to app-level URL modules using `include()`.

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/v1/accounts/", include("apps.accounts.urls", namespace="accounts")),
    path("api/v1/products/", include("apps.products.urls", namespace="products")),
]
```

```python
# apps/products/urls.py
from django.urls import path
from . import views

app_name = "products"

urlpatterns = [
    path("", views.ProductListView.as_view(), name="product-list"),
    path("<int:pk>/", views.ProductDetailView.as_view(), name="product-detail"),
    path("<int:pk>/reviews/", views.ProductReviewsView.as_view(), name="product-reviews"),
]
```

Use `app_name` in every app URL module to enable namespaced URL reversing with `reverse("products:product-detail", kwargs={"pk": 1})`.

## Class-Based Views vs Function-Based Views

Choose function-based views for simple, one-off endpoints with straightforward logic. Choose class-based views when inheritance, mixins, or method dispatch provide concrete reuse benefits.

### Function-Based View

```python
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods

@require_http_methods(["GET"])
def health_check(request):
    return JsonResponse({"status": "ok"})
```

### Class-Based View with Mixins

```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView

class ProductListView(LoginRequiredMixin, PermissionRequiredMixin, ListView):
    model = Product
    template_name = "products/product_list.html"
    context_object_name = "products"
    permission_required = "products.view_product"
    paginate_by = 25

    def get_queryset(self):
        qs = super().get_queryset().select_related("category")
        search = self.request.GET.get("search")
        if search:
            qs = qs.filter(name__icontains=search)
        return qs
```

Prefer `ListView`, `DetailView`, `CreateView`, `UpdateView`, and `DeleteView` for standard CRUD. Override `get_queryset`, `get_context_data`, and `form_valid` to customize behavior without rewriting boilerplate.

## Middleware Pipeline

Middleware processes requests and responses in a layered pipeline. The order of `MIDDLEWARE` in settings matters: request processing flows top-down, response processing flows bottom-up.

Write custom middleware as a callable class:

```python
import time
import logging

logger = logging.getLogger(__name__)

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = time.monotonic() - start
        logger.info(
            "method=%s path=%s status=%s duration=%.3fs",
            request.method,
            request.path,
            response.status_code,
            duration,
        )
        response["X-Request-Duration"] = f"{duration:.3f}"
        return response
```

For middleware that needs to hook into view processing, implement `process_view`, `process_exception`, or `process_template_response` methods on the class.

```python
class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tenant_header = request.headers.get("X-Tenant-ID")
        if tenant_header:
            request.tenant = Tenant.objects.get(slug=tenant_header)
        else:
            request.tenant = None
        return self.get_response(request)

    def process_view(self, request, view_func, view_args, view_kwargs):
        if getattr(view_func, "requires_tenant", False) and request.tenant is None:
            return JsonResponse({"error": "Tenant header required"}, status=400)
        return None
```

## Async Views

Django 5.+ provides native async view support. Use async views for I/O-bound operations such as calling external APIs, reading from async-compatible caches, or performing concurrent database queries with the async ORM interface.

```python
import httpx
from django.http import JsonResponse

async def fetch_external_data(request):
    async with httpx.AsyncClient() as client:
        weather_resp = await client.get("https://api.weather.example.com/current")
        news_resp = await client.get("https://api.news.example.com/latest")
    return JsonResponse({
        "weather": weather_resp.json(),
        "news": news_resp.json(),
    })
```

For async class-based views, Django 5.+ supports async method handlers:

```python
from django.views import View
from django.http import JsonResponse

class AsyncDataView(View):
    async def get(self, request):
        results = await sync_to_async(self._fetch_from_db)()
        return JsonResponse({"results": results})

    def _fetch_from_db(self):
        return list(Product.objects.values("id", "name")[:50])
```

When mixing sync ORM calls inside async views, wrap them with `sync_to_async` from `asgiref.sync`. Prefer the async ORM methods (`Model.objects.aget()`, `Model.objects.aall()`, etc.) when available.

Deploy async views under an ASGI server (uvicorn, daphne, or hypercorn). Ensure the ASGI application is configured in `config/asgi.py`.

## Django 5.+ Features

Take advantage of Django 5.+ additions:

- **`GeneratedField`**: Define database-computed columns that stay in sync automatically.
- **Field group templates**: Simplify form rendering with reusable field group templates.
- **Simplified form rendering**: Use `as_div()`, `as_field_group()` for cleaner template output.
- **Async ORM methods**: `aget()`, `acreate()`, `aupdate()`, `adelete()`, `acount()`, `aexists()`, and async iteration over querysets.
- **Login required by default**: Configure `LoginRequiredMiddleware` to require authentication globally, then exempt specific views with the `login_not_required` decorator.

```python
from django.db import models

class Product(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    full_name = models.GeneratedField(
        expression=models.functions.Concat(
            models.F("first_name"), models.Value(" "), models.F("last_name")
        ),
        output_field=models.CharField(max_length=201),
        db_persist=True,
    )
```

---

## References

For Django-specific SOLID design principles and architectural patterns, see [solid-patterns.md](references/solid-patterns.md).

For Django 5.+ best practices covering security, performance, and deployment, see [best-practices.md](references/best-practices.md).
