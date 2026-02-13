# Django 5.+ Best Practices

This reference covers Django 5.+ features, security hardening, performance optimization, and a production deployment checklist. Every recommendation targets Django 5.0 and later.

---

## Django 5.+ Features

### GeneratedField

`GeneratedField` defines database-level computed columns. The database engine computes and stores (or virtualizes) the value, eliminating application-level denormalization bugs.

```python
from django.db import models
from django.db.models.functions import Concat, Lower

class Customer(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    email = models.EmailField()

    full_name = models.GeneratedField(
        expression=Concat("first_name", models.Value(" "), "last_name"),
        output_field=models.CharField(max_length=201),
        db_persist=True,
    )
    email_domain = models.GeneratedField(
        expression=models.Func(
            models.F("email"),
            models.Value("@(.+)$"),
            function="REGEXP_SUBSTR",
        ),
        output_field=models.CharField(max_length=255),
        db_persist=True,
    )
```

Set `db_persist=True` for columns that need indexing. Set `db_persist=False` for virtual columns that save disk space but cannot be indexed.

### Async ORM

Django 5.+ provides async-native ORM methods. Use them in async views to avoid blocking the event loop.

```python
# Async view using async ORM
async def product_list(request):
    products = [
        product async for product in
        Product.objects.filter(is_active=True).order_by("-created_at")[:20]
    ]
    data = [{"id": p.id, "name": p.name} for p in products]
    return JsonResponse({"products": data})

# Individual async operations
product = await Product.objects.aget(pk=42)
count = await Product.objects.filter(is_active=True).acount()
exists = await Product.objects.filter(sku="ABC123").aexists()
await Product.objects.filter(pk=42).aupdate(stock=F("stock") - 1)
```

Do not mix sync ORM calls inside async views without wrapping them in `sync_to_async`. Prefer the native async methods when available.

### LoginRequiredMiddleware

Django 5.1+ includes `LoginRequiredMiddleware` that enforces authentication globally. Exempt public views with the `login_not_required` decorator.

```python
# settings
MIDDLEWARE = [
    # ...
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
]

# views.py
from django.contrib.auth.decorators import login_not_required

@login_not_required
def public_landing_page(request):
    return render(request, "landing.html")
```

### Form Rendering Improvements

Django 5.+ form rendering uses `as_div()` and field group templates for cleaner HTML output.

```python
# Template
{{ form.as_div }}

# Or per-field with field groups
{% for field in form %}
    {{ field.as_field_group }}
{% endfor %}
```

---

## Security

### CSRF Protection

Django's CSRF middleware is enabled by default. Follow these rules to maintain protection:

- Never exempt views from CSRF without a documented reason and an alternative protection mechanism (such as token authentication for APIs).
- Use `{% csrf_token %}` in every HTML form that targets a POST endpoint.
- For AJAX requests from JavaScript, read the `csrftoken` cookie and send it as the `X-CSRFToken` header.
- Set `CSRF_COOKIE_HTTPONLY = False` (the default) so JavaScript can read the cookie, but set `CSRF_COOKIE_SECURE = True` in production.

```python
# DRF views using token auth are exempt from CSRF by default via
# SessionAuthentication's enforce_csrf check. Ensure your API views
# use TokenAuthentication or JWTAuthentication, not SessionAuthentication,
# if they serve non-browser clients.
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
```

### XSS Prevention

Django's template engine auto-escapes all variables by default. Maintain this protection:

- Never use `|safe` or `{% autoescape off %}` on user-provided data.
- Sanitize rich-text input with `bleach` or `nh3` before saving to the database.
- Use `json_script` template tag to safely embed JSON in templates.

```html
<!-- Safe JSON embedding -->
{{ data|json_script:"page-data" }}
<script>
    const data = JSON.parse(document.getElementById("page-data").textContent);
</script>
```

### SQL Injection Prevention

The Django ORM parameterizes all queries. Follow these rules:

- Never use string formatting to build queries. Use queryset methods, `Q` objects, or parameterized `raw()`.
- When `raw()` is necessary, always use parameter substitution.

```python
# CORRECT: parameterized raw query
Product.objects.raw("SELECT * FROM product WHERE category_id = %s", [category_id])

# WRONG: string formatting (SQL injection risk)
Product.objects.raw(f"SELECT * FROM product WHERE category_id = {category_id}")
```

### Content Security Policy

Use `django-csp` to set Content Security Policy headers.

```python
# settings
MIDDLEWARE += ["csp.middleware.CSPMiddleware"]

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "cdn.example.com")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
CSP_IMG_SRC = ("'self'", "data:", "images.example.com")
CSP_FONT_SRC = ("'self'", "fonts.googleapis.com")
CSP_CONNECT_SRC = ("'self'", "api.example.com")
```

### Additional Security Settings

```python
# Production security settings
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_AGE = 3600  # 1 hour
CSRF_COOKIE_SECURE = True
X_FRAME_OPTIONS = "DENY"
SECURE_REFERRER_POLICY = "strict-origin-when-cross-origin"
```

---

## Performance

### Caching Strategy

Layer caching at multiple levels: per-view, per-template-fragment, and low-level cache API.

```python
# Per-view caching
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def product_list(request):
    ...

# Template fragment caching
{% load cache %}
{% cache 300 product_sidebar product.pk %}
    <div class="sidebar">{{ product.description }}</div>
{% endcache %}

# Low-level cache API
from django.core.cache import cache

def get_category_tree():
    cache_key = "category_tree_v1"
    tree = cache.get(cache_key)
    if tree is None:
        tree = list(Category.objects.filter(parent__isnull=True).prefetch_related("children"))
        cache.set(cache_key, tree, timeout=3600)
    return tree
```

Configure Redis as the cache backend in production:

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ.get("REDIS_URL", "redis://127.0.0.1:6379/1"),
    }
}
```

### Database Optimization

#### select_related and prefetch_related

Use `select_related` for foreign key and one-to-one relationships (SQL JOIN). Use `prefetch_related` for many-to-many and reverse foreign key relationships (separate query + Python-side join).

```python
# BAD: N+1 queries
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)  # One query per order

# GOOD: Single JOIN
orders = Order.objects.select_related("customer").all()
for order in orders:
    print(order.customer.name)  # No additional queries

# GOOD: Prefetch for reverse FK
customers = Customer.objects.prefetch_related(
    Prefetch("orders", queryset=Order.objects.filter(status="completed"))
).all()
```

#### Indexing

Add `db_index=True` to fields used in `filter()`, `order_by()`, and `exclude()`. Use `Meta.indexes` for composite indexes.

```python
class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["customer", "status"]),
            models.Index(fields=["-created_at"]),
            models.Index(
                fields=["status"],
                condition=models.Q(status="pending"),
                name="pending_orders_idx",
            ),
        ]
```

#### Bulk Operations

Use `bulk_create`, `bulk_update`, and `delete()` on querysets for batch operations instead of looping and saving individually.

```python
# Create in bulk
products = [Product(name=f"Product {i}", price=i * 10) for i in range(1000)]
Product.objects.bulk_create(products, batch_size=250)

# Update in bulk
Product.objects.filter(category="electronics").update(on_sale=True)

# bulk_update for heterogeneous changes
for product in products_to_update:
    product.price *= 1.1
Product.objects.bulk_update(products_to_update, ["price"], batch_size=250)
```

#### Query Analysis

Use `django-debug-toolbar` in development and `EXPLAIN ANALYZE` for slow queries.

```python
# Print the SQL for a queryset
print(Product.objects.filter(is_active=True).query)

# Use .explain() for execution plan
print(Product.objects.filter(is_active=True).explain(analyze=True))
```

---

## Deployment Checklist

Run `python manage.py check --deploy` before every production deployment. It catches missing security settings.

### Pre-Deployment

1. Set `DEBUG = False`.
2. Set `ALLOWED_HOSTS` to the production domain(s).
3. Set `SECRET_KEY` from an environment variable (minimum 50 characters).
4. Enable all security headers (`SECURE_SSL_REDIRECT`, `HSTS`, `CSP`).
5. Configure production database with connection pooling (use `django-db-connection-pool` or PgBouncer).
6. Configure Redis for caching and session storage.
7. Run `python manage.py collectstatic --noinput`.
8. Run `python manage.py migrate --run-syncdb`.
9. Set up structured logging with `django-structlog` or the built-in `LOGGING` dict.

### Runtime

1. Serve static files via a CDN or reverse proxy (nginx, CloudFront), not Django.
2. Run with gunicorn (WSGI) or uvicorn (ASGI) behind a reverse proxy.
3. Set up health check endpoints outside authentication middleware.
4. Monitor slow queries with `django-silk` or APM (Datadog, New Relic, Sentry).
5. Configure error tracking (Sentry with `sentry-sdk[django]`).
6. Set `CONN_MAX_AGE` to reuse database connections (e.g., `CONN_MAX_AGE = 600`).
7. Use `ATOMIC_REQUESTS = True` if every view should run in a transaction, or wrap critical operations in `transaction.atomic()` explicitly.

### Scaling

1. Horizontally scale web workers behind a load balancer.
2. Offload background tasks to Celery with Redis or RabbitMQ as broker.
3. Use database read replicas for read-heavy workloads with `DATABASE_ROUTERS`.
4. Cache aggressively at the view, fragment, and object levels.
5. Use CDN edge caching for static assets and public API responses.
6. Monitor and set alerts for response time p95, error rate, and database connection pool usage.
