# SOLID Principles in Django

This reference applies the five SOLID principles to Django-specific patterns with production-ready code examples. Each principle is illustrated with concrete Django patterns that demonstrate both the violation and the correct implementation.

---

## Single Responsibility Principle (SRP)

A class should have only one reason to change. In Django, this means separating concerns between models, views, services, and serializers.

### Fat Models, Thin Views

Move business logic out of views into model methods or a service layer. The view's only job is to handle HTTP: parse the request, call the right logic, and return a response.

**Violation: Business logic in the view**

```python
class OrderCreateView(APIView):
    def post(self, request):
        items = request.data["items"]
        total = sum(item["price"] * item["quantity"] for item in items)
        if total > request.user.credit_limit:
            return Response({"error": "Credit limit exceeded"}, status=400)
        order = Order.objects.create(user=request.user, total=total)
        for item in items:
            OrderItem.objects.create(order=order, **item)
        send_mail("Order Confirmation", f"Order {order.id}", None, [request.user.email])
        return Response(OrderSerializer(order).data, status=201)
```

**Correct: Service layer handles business logic**

```python
# services/order_service.py
from django.core.mail import send_mail
from django.db import transaction

class OrderService:
    @transaction.atomic
    def create_order(self, user, items_data):
        total = self._calculate_total(items_data)
        self._validate_credit_limit(user, total)
        order = Order.objects.create(user=user, total=total)
        order_items = [
            OrderItem(order=order, **item) for item in items_data
        ]
        OrderItem.objects.bulk_create(order_items)
        self._send_confirmation(user, order)
        return order

    def _calculate_total(self, items_data):
        return sum(item["price"] * item["quantity"] for item in items_data)

    def _validate_credit_limit(self, user, total):
        if total > user.credit_limit:
            raise ValidationError("Credit limit exceeded")

    def _send_confirmation(self, user, order):
        send_mail("Order Confirmation", f"Order {order.id}", None, [user.email])
```

```python
# views.py
class OrderCreateView(APIView):
    def post(self, request):
        service = OrderService()
        order = service.create_order(request.user, request.data["items"])
        return Response(OrderSerializer(order).data, status=201)
```

### Service Layer Pattern

Create a `services/` module within each app. Each service class owns one domain process. Services call other services for cross-domain coordination but never import view classes.

```
apps/orders/
    services/
        __init__.py
        order_service.py
        payment_service.py
        notification_service.py
    views.py
    models.py
    serializers.py
```

---

## Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification. In Django, achieve this through middleware chains, CBV mixins, and custom template tags.

### Middleware Chain

Each middleware adds behavior without modifying existing middleware. New cross-cutting concerns become new middleware classes added to the `MIDDLEWARE` list.

```python
class AuditLogMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        if request.method in ("POST", "PUT", "PATCH", "DELETE"):
            AuditLog.objects.create(
                user=getattr(request, "user", None),
                method=request.method,
                path=request.path,
                status_code=response.status_code,
            )
        return response
```

Adding audit logging requires no changes to any view, model, or existing middleware.

### CBV Mixins

Extend class-based views by composing mixins. Each mixin addresses a single concern.

```python
class TenantScopedMixin:
    """Filters queryset to the current tenant."""
    def get_queryset(self):
        return super().get_queryset().filter(tenant=self.request.tenant)

class SoftDeleteMixin:
    """Excludes soft-deleted records from queryset."""
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)

class CachedListMixin:
    """Caches list responses for the specified duration."""
    cache_timeout = 300

    def list(self, request, *args, **kwargs):
        cache_key = f"{self.__class__.__name__}:{request.get_full_path()}"
        cached = cache.get(cache_key)
        if cached:
            return Response(cached)
        response = super().list(request, *args, **kwargs)
        cache.set(cache_key, response.data, self.cache_timeout)
        return response

class ProductViewSet(TenantScopedMixin, SoftDeleteMixin, CachedListMixin, ModelViewSet):
    serializer_class = ProductSerializer
    queryset = Product.objects.all()
```

### Custom Template Tags

Extend template rendering without modifying existing templates or template engine code.

```python
# templatetags/currency_tags.py
from django import template
import decimal

register = template.Library()

@register.filter
def currency(value, currency_code="USD"):
    symbols = {"USD": "$", "EUR": "\u20ac", "GBP": "\u00a3"}
    symbol = symbols.get(currency_code, currency_code)
    return f"{symbol}{decimal.Decimal(value):,.2f}"
```

---

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering the correctness of the program. In Django, this applies to model inheritance strategies.

### Abstract Models

Abstract models define shared fields and methods. Concrete subclasses add domain-specific fields while remaining interchangeable where only the base interface is needed.

```python
class Publishable(models.Model):
    status = models.CharField(
        max_length=20,
        choices=[("draft", "Draft"), ("published", "Published"), ("archived", "Archived")],
        default="draft",
    )
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def publish(self):
        self.status = "published"
        self.published_at = timezone.now()
        self.save(update_fields=["status", "published_at"])

    def is_visible(self):
        return self.status == "published"

class Article(Publishable):
    title = models.CharField(max_length=200)
    body = models.TextField()

class Event(Publishable):
    name = models.CharField(max_length=200)
    event_date = models.DateTimeField()
```

Both `Article` and `Event` honor the `Publishable` contract. Any code that calls `publish()` or `is_visible()` works identically on both.

### Proxy Models

Proxy models change behavior (methods, default managers, ordering) without changing the database table. They must honor the parent model's interface.

```python
class ActiveProductManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_active=True)

class ActiveProduct(Product):
    objects = ActiveProductManager()

    class Meta:
        proxy = True
        ordering = ["-created_at"]
```

`ActiveProduct` can substitute for `Product` anywhere `Product` is expected; it simply filters the default queryset.

### Multi-Table Inheritance

Use sparingly. Each child model creates a separate table with an implicit `OneToOneField` to the parent. The parent's interface remains valid on the child, satisfying LSP, but queries against the parent table do not automatically include child fields without explicit joins.

```python
class Place(models.Model):
    name = models.CharField(max_length=200)
    address = models.TextField()

class Restaurant(Place):
    cuisine_type = models.CharField(max_length=100)
    serves_alcohol = models.BooleanField(default=False)
```

---

## Interface Segregation Principle (ISP)

Clients should not be forced to depend on interfaces they do not use. In Django, apply this to serializers, managers, and queryset methods.

### Lean Serializers

Create separate serializers for different use cases instead of one monolithic serializer with conditional field inclusion.

```python
class ProductListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["id", "name", "price", "thumbnail_url"]

class ProductDetailSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)

    class Meta:
        model = Product
        fields = [
            "id", "name", "description", "price", "sku",
            "category", "reviews", "created_at",
        ]

class ProductCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["name", "description", "price", "sku", "category"]
```

### Targeted Manager and QuerySet Methods

Define focused queryset methods that compose together rather than one manager method that accepts many parameters.

```python
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def in_stock(self):
        return self.filter(stock_quantity__gt=0)

    def in_category(self, category):
        return self.filter(category=category)

    def expensive(self, threshold=100):
        return self.filter(price__gte=threshold)

class Product(models.Model):
    objects = ProductQuerySet.as_manager()

# Usage: composable, each method does one thing
Product.objects.active().in_stock().in_category(electronics).expensive(50)
```

---

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions. In Django, achieve this through abstract base classes for services, settings-based injection, and the repository pattern.

### Service Layer with ABCs

Define abstract interfaces for services so that high-level orchestration code does not depend on concrete implementations.

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount, currency, token):
        ...

    @abstractmethod
    def refund(self, transaction_id, amount):
        ...

class StripeGateway(PaymentGateway):
    def charge(self, amount, currency, token):
        return stripe.Charge.create(amount=amount, currency=currency, source=token)

    def refund(self, transaction_id, amount):
        return stripe.Refund.create(charge=transaction_id, amount=amount)

class MockGateway(PaymentGateway):
    def charge(self, amount, currency, token):
        return {"id": "mock_txn_123", "status": "succeeded"}

    def refund(self, transaction_id, amount):
        return {"id": "mock_refund_123", "status": "succeeded"}
```

### Settings-Based Injection

Use Django settings to select the concrete implementation at runtime.

```python
# settings/base.py
PAYMENT_GATEWAY_CLASS = "apps.payments.gateways.StripeGateway"

# settings/testing.py
PAYMENT_GATEWAY_CLASS = "apps.payments.gateways.MockGateway"
```

```python
# utils/injection.py
from django.conf import settings
from django.utils.module_loading import import_string

def get_payment_gateway():
    gateway_class = import_string(settings.PAYMENT_GATEWAY_CLASS)
    return gateway_class()
```

### Repository Pattern

Isolate data access behind a repository interface so that business logic does not depend on Django ORM specifics.

```python
class ProductRepository(ABC):
    @abstractmethod
    def get_by_id(self, product_id):
        ...

    @abstractmethod
    def list_active(self, category=None):
        ...

class DjangoProductRepository(ProductRepository):
    def get_by_id(self, product_id):
        return Product.objects.get(pk=product_id)

    def list_active(self, category=None):
        qs = Product.objects.filter(is_active=True)
        if category:
            qs = qs.filter(category=category)
        return qs.select_related("category")
```

This allows swapping the data layer (for example, replacing ORM queries with an external API client) without changing any service or view code that depends on `ProductRepository`.
