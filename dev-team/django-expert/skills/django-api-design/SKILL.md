---
name: Django API Design
description: This skill should be used when the user asks about "DRF serializer", "Django REST framework", "DRF viewset", "DRF router", or "DRF permission". It covers Django REST Framework (DRF) API design including serializers, viewsets, routers, permissions, throttling, pagination, and filtering. Use this skill for building RESTful APIs with DRF, designing serializer hierarchies, configuring viewset routing, implementing custom permissions, setting up rate limiting, or adding pagination and filtering to API endpoints.
---

## Serializer Design

Serializers handle the conversion between complex Django model instances and Python primitives suitable for JSON rendering. Design separate serializers for different operations and response shapes.

### ModelSerializer

```python
from rest_framework import serializers
from apps.products.models import Product, Category

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ["id", "name", "slug"]

class ProductListSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source="category.name", read_only=True)

    class Meta:
        model = Product
        fields = ["id", "name", "slug", "price", "category_name", "thumbnail_url"]

class ProductDetailSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)

    class Meta:
        model = Product
        fields = [
            "id", "name", "slug", "description", "price", "sku",
            "stock_quantity", "category", "is_active", "created_at",
        ]

class ProductCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["name", "description", "price", "sku", "stock_quantity", "category"]

    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError("Price must be positive.")
        return value

    def validate(self, attrs):
        if attrs.get("sale_price") and attrs["sale_price"] >= attrs["price"]:
            raise serializers.ValidationError("Sale price must be less than regular price.")
        return attrs
```

Use `ProductListSerializer` for list endpoints (fewer fields, faster serialization). Use `ProductDetailSerializer` for retrieve endpoints. Use `ProductCreateSerializer` for create/update endpoints with input validation.

### Nested Writable Serializers

Handle nested writes by overriding `create()` and `update()`:

```python
class OrderCreateSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ["customer_note", "shipping_address", "items"]

    def create(self, validated_data):
        items_data = validated_data.pop("items")
        order = Order.objects.create(**validated_data)
        order_items = [OrderItem(order=order, **item) for item in items_data]
        OrderItem.objects.bulk_create(order_items)
        return order

    def update(self, instance, validated_data):
        items_data = validated_data.pop("items", None)
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        if items_data is not None:
            instance.items.all().delete()
            order_items = [OrderItem(order=instance, **item) for item in items_data]
            OrderItem.objects.bulk_create(order_items)
        return instance
```

### SerializerMethodField

Use for computed fields that do not map directly to a model field:

```python
class ProductDetailSerializer(serializers.ModelSerializer):
    average_rating = serializers.SerializerMethodField()
    is_in_stock = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = ["id", "name", "price", "average_rating", "is_in_stock"]

    def get_average_rating(self, obj):
        return obj.reviews.aggregate(avg=Avg("rating"))["avg"]

    def get_is_in_stock(self, obj):
        return obj.stock_quantity > 0
```

Annotate the queryset in the viewset's `get_queryset()` to avoid N+1 queries from `SerializerMethodField`.

## ViewSets and Routers

### ModelViewSet

`ModelViewSet` provides `list`, `create`, `retrieve`, `update`, `partial_update`, and `destroy` actions.

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related("category").all()
    permission_classes = [IsAuthenticated]
    filterset_class = ProductFilter
    search_fields = ["name", "description", "sku"]
    ordering_fields = ["price", "created_at", "name"]
    ordering = ["-created_at"]

    def get_serializer_class(self):
        if self.action == "list":
            return ProductListSerializer
        if self.action == "retrieve":
            return ProductDetailSerializer
        return ProductCreateSerializer

    def get_queryset(self):
        qs = super().get_queryset()
        if self.action in ("list", "retrieve"):
            qs = qs.annotate(
                avg_rating=Avg("reviews__rating"),
                review_count=Count("reviews"),
            )
        return qs

    @action(detail=True, methods=["post"])
    def archive(self, request, pk=None):
        product = self.get_object()
        product.is_active = False
        product.save(update_fields=["is_active"])
        return Response({"status": "archived"})

    @action(detail=False, methods=["post"])
    def bulk_update_prices(self, request):
        serializer = BulkPriceUpdateSerializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        # Process bulk update
        return Response({"updated": len(serializer.validated_data)})
```

### Router Configuration

```python
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r"products", ProductViewSet, basename="product")
router.register(r"categories", CategoryViewSet, basename="category")
router.register(r"orders", OrderViewSet, basename="order")

urlpatterns = [
    path("api/v1/", include(router.urls)),
]
```

The router generates URLs:
- `GET /api/v1/products/` - list
- `POST /api/v1/products/` - create
- `GET /api/v1/products/{pk}/` - retrieve
- `PUT /api/v1/products/{pk}/` - update
- `PATCH /api/v1/products/{pk}/` - partial update
- `DELETE /api/v1/products/{pk}/` - destroy
- `POST /api/v1/products/{pk}/archive/` - custom action

## Permissions

### Built-in Permissions

```python
from rest_framework.permissions import (
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
    AllowAny,
)

class ProductViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
```

### Custom Permissions

```python
from rest_framework.permissions import BasePermission

class IsOwnerOrReadOnly(BasePermission):
    """Allow write access only to the object's owner."""

    def has_object_permission(self, request, view, obj):
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return True
        return obj.owner == request.user

class IsStaffOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return True
        return request.user and request.user.is_staff
```

### Per-Action Permissions

```python
class ProductViewSet(viewsets.ModelViewSet):
    def get_permissions(self):
        if self.action in ("create", "update", "partial_update", "destroy"):
            return [IsAuthenticated(), IsStaffOrReadOnly()]
        return [AllowAny()]
```

## Throttling

Configure rate limiting to protect against abuse:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/hour",
        "user": "1000/hour",
    },
}
```

### Custom Throttles

```python
from rest_framework.throttling import SimpleRateThrottle

class BurstRateThrottle(SimpleRateThrottle):
    scope = "burst"
    rate = "10/minute"

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            return self.cache_format % {
                "scope": self.scope,
                "ident": request.user.pk,
            }
        return self.get_ident(request)

class OrderCreationThrottle(SimpleRateThrottle):
    scope = "order_creation"
    rate = "5/hour"

    def get_cache_key(self, request, view):
        return self.cache_format % {
            "scope": self.scope,
            "ident": request.user.pk,
        }
```

Apply per-view:

```python
class OrderViewSet(viewsets.ModelViewSet):
    def get_throttles(self):
        if self.action == "create":
            return [OrderCreationThrottle()]
        return super().get_throttles()
```

## Pagination

### Global Pagination

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

### Custom Pagination

```python
from rest_framework.pagination import CursorPagination, LimitOffsetPagination

class ProductCursorPagination(CursorPagination):
    page_size = 20
    ordering = "-created_at"
    cursor_query_param = "cursor"

class LargeResultSetPagination(LimitOffsetPagination):
    default_limit = 50
    max_limit = 200

class ProductViewSet(viewsets.ModelViewSet):
    pagination_class = ProductCursorPagination
```

Use `CursorPagination` for large datasets where consistent ordering is required and users do not need to jump to arbitrary pages. Use `PageNumberPagination` for traditional page-based navigation. Use `LimitOffsetPagination` for flexible offset-based access.

## Filtering

### django-filter Integration

```bash
pip install django-filter
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
}
```

```python
import django_filters
from apps.products.models import Product

class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr="gte")
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr="lte")
    category = django_filters.CharFilter(field_name="category__slug")
    created_after = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="gte")

    class Meta:
        model = Product
        fields = ["is_active", "category", "min_price", "max_price"]

class ProductViewSet(viewsets.ModelViewSet):
    filterset_class = ProductFilter
    search_fields = ["name", "description", "sku"]
    ordering_fields = ["price", "created_at", "name"]
```

Query examples:
- `GET /api/v1/products/?min_price=10&max_price=100`
- `GET /api/v1/products/?search=widget`
- `GET /api/v1/products/?ordering=-price`
- `GET /api/v1/products/?category=electronics&is_active=true`

## API Documentation with drf-spectacular

```bash
pip install drf-spectacular
```

```python
# settings.py
INSTALLED_APPS += ["drf_spectacular"]

REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

SPECTACULAR_SETTINGS = {
    "TITLE": "Product API",
    "DESCRIPTION": "API for managing products and orders",
    "VERSION": "1.0.0",
    "SERVE_INCLUDE_SCHEMA": False,
}
```

```python
# urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
]
```

Annotate custom actions with `@extend_schema`:

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter

class ProductViewSet(viewsets.ModelViewSet):
    @extend_schema(
        parameters=[
            OpenApiParameter(name="category", type=str, description="Filter by category slug"),
        ],
        responses={200: ProductListSerializer(many=True)},
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

---

## References

For advanced API patterns including nested serializers, custom actions, versioning, content negotiation, and bulk operations, see [api-patterns.md](references/api-patterns.md).

For API security covering token authentication, JWT, OAuth2, object-level permissions, rate limiting, and CORS, see [api-security.md](references/api-security.md).
