# DRF API Patterns

This reference covers advanced Django REST Framework patterns for building production-grade APIs: nested serializers, custom viewset actions, API versioning, content negotiation, OpenAPI documentation with drf-spectacular, and bulk operations.

---

## Nested Serializers

### Read-Only Nested Serializers

Nest serializers for read operations to provide rich, denormalized responses without requiring additional API calls.

```python
class ReviewSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.get_full_name", read_only=True)

    class Meta:
        model = Review
        fields = ["id", "rating", "comment", "author_name", "created_at"]

class ProductDetailSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)

    class Meta:
        model = Product
        fields = [
            "id", "name", "description", "price", "sku",
            "category", "reviews", "tags", "created_at",
        ]
```

Optimize the viewset queryset to prevent N+1 queries:

```python
class ProductViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        qs = Product.objects.all()
        if self.action == "retrieve":
            qs = qs.select_related("category").prefetch_related(
                Prefetch("reviews", queryset=Review.objects.select_related("author").order_by("-created_at")[:10]),
                "tags",
            )
        elif self.action == "list":
            qs = qs.select_related("category")
        return qs
```

### Writable Nested Serializers

For create/update operations with nested data, override `create()` and `update()` on the serializer.

```python
class AddressSerializer(serializers.ModelSerializer):
    class Meta:
        model = Address
        fields = ["street", "city", "state", "zip_code", "country"]

class CustomerCreateSerializer(serializers.ModelSerializer):
    billing_address = AddressSerializer()
    shipping_address = AddressSerializer()

    class Meta:
        model = Customer
        fields = ["name", "email", "billing_address", "shipping_address"]

    def create(self, validated_data):
        billing_data = validated_data.pop("billing_address")
        shipping_data = validated_data.pop("shipping_address")
        billing = Address.objects.create(**billing_data)
        shipping = Address.objects.create(**shipping_data)
        customer = Customer.objects.create(
            billing_address=billing,
            shipping_address=shipping,
            **validated_data,
        )
        return customer

    def update(self, instance, validated_data):
        billing_data = validated_data.pop("billing_address", None)
        shipping_data = validated_data.pop("shipping_address", None)

        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        if billing_data:
            for attr, value in billing_data.items():
                setattr(instance.billing_address, attr, value)
            instance.billing_address.save()

        if shipping_data:
            for attr, value in shipping_data.items():
                setattr(instance.shipping_address, attr, value)
            instance.shipping_address.save()

        return instance
```

### PrimaryKeyRelatedField for Write, Serializer for Read

Accept foreign key IDs on write but return full objects on read:

```python
class ProductCreateSerializer(serializers.ModelSerializer):
    category = serializers.PrimaryKeyRelatedField(queryset=Category.objects.all())

    class Meta:
        model = Product
        fields = ["name", "price", "category"]

    def to_representation(self, instance):
        data = super().to_representation(instance)
        data["category"] = CategorySerializer(instance.category).data
        return data
```

---

## Custom ViewSet Actions

### Detail Actions (Operate on a Single Object)

```python
class OrderViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=["post"], url_path="cancel")
    def cancel(self, request, pk=None):
        order = self.get_object()
        if order.status not in ("pending", "processing"):
            return Response(
                {"error": "Only pending or processing orders can be cancelled."},
                status=status.HTTP_400_BAD_REQUEST,
            )
        order.status = "cancelled"
        order.cancelled_at = timezone.now()
        order.save(update_fields=["status", "cancelled_at"])
        return Response(OrderDetailSerializer(order).data)

    @action(detail=True, methods=["get"], url_path="invoice")
    def invoice(self, request, pk=None):
        order = self.get_object()
        pdf_content = generate_invoice_pdf(order)
        response = HttpResponse(pdf_content, content_type="application/pdf")
        response["Content-Disposition"] = f'attachment; filename="invoice_{order.pk}.pdf"'
        return response

    @action(detail=True, methods=["post"], url_path="add-note")
    def add_note(self, request, pk=None):
        order = self.get_object()
        serializer = OrderNoteSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(order=order, author=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### List Actions (Operate on the Collection)

```python
class ProductViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=["get"], url_path="featured")
    def featured(self, request):
        featured = self.get_queryset().filter(is_featured=True)[:10]
        serializer = self.get_serializer(featured, many=True)
        return Response(serializer.data)

    @action(detail=False, methods=["get"], url_path="stats")
    def stats(self, request):
        qs = self.get_queryset()
        data = qs.aggregate(
            total_products=Count("id"),
            avg_price=Avg("price"),
            total_value=Sum(F("price") * F("stock_quantity")),
        )
        return Response(data)

    @action(detail=False, methods=["post"], url_path="import")
    def import_products(self, request):
        file = request.FILES.get("file")
        if not file:
            return Response({"error": "No file provided"}, status=400)
        task = import_products_task.delay(file.read())
        return Response({"task_id": task.id}, status=202)
```

---

## API Versioning

### URL Path Versioning (Recommended)

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
}

# urls.py
urlpatterns = [
    path("api/v1/", include("apps.api.v1.urls")),
    path("api/v2/", include("apps.api.v2.urls")),
]
```

### Version-Specific Serializers

```python
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == "v2":
            return ProductSerializerV2
        return ProductSerializerV1
```

### Header Versioning

```python
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.AcceptHeaderVersioning",
}

# Client sends: Accept: application/json; version=2.0
```

---

## Content Negotiation

### Custom Renderers

```python
from rest_framework.renderers import BaseRenderer
import csv
import io

class CSVRenderer(BaseRenderer):
    media_type = "text/csv"
    format = "csv"

    def render(self, data, accepted_media_type=None, renderer_context=None):
        if not data:
            return ""
        output = io.StringIO()
        if isinstance(data, list):
            writer = csv.DictWriter(output, fieldnames=data[0].keys())
            writer.writeheader()
            writer.writerows(data)
        return output.getvalue()

class ProductViewSet(viewsets.ModelViewSet):
    renderer_classes = [JSONRenderer, CSVRenderer]
```

Request CSV format: `GET /api/v1/products/?format=csv` or `Accept: text/csv`.

---

## drf-spectacular Advanced Usage

### Inline Schema Annotations

```python
from drf_spectacular.utils import extend_schema, inline_serializer, OpenApiExample

class OrderViewSet(viewsets.ModelViewSet):
    @extend_schema(
        request=OrderCreateSerializer,
        responses={
            201: OrderDetailSerializer,
            400: inline_serializer(
                name="OrderCreateError",
                fields={"error": serializers.CharField()},
            ),
        },
        examples=[
            OpenApiExample(
                "Create order",
                value={
                    "items": [{"product_id": 1, "quantity": 2}],
                    "shipping_address": "123 Main St",
                },
                request_only=True,
            ),
        ],
    )
    def create(self, request, *args, **kwargs):
        return super().create(request, *args, **kwargs)
```

### Tags and Grouping

```python
@extend_schema(tags=["Products"])
class ProductViewSet(viewsets.ModelViewSet):
    pass

@extend_schema(tags=["Orders"])
class OrderViewSet(viewsets.ModelViewSet):
    pass
```

### Schema Customization

```python
# settings.py
SPECTACULAR_SETTINGS = {
    "TITLE": "E-Commerce API",
    "DESCRIPTION": "Full-featured e-commerce API",
    "VERSION": "1.0.0",
    "COMPONENT_SPLIT_REQUEST": True,  # Separate request/response schemas
    "SCHEMA_PATH_PREFIX": r"/api/v[0-9]+",
    "SERVE_PERMISSIONS": ["rest_framework.permissions.IsAdminUser"],
    "POSTPROCESSING_HOOKS": [
        "drf_spectacular.hooks.postprocess_schema_enums",
    ],
}
```

---

## Bulk Operations

### Bulk Create

```python
class BulkProductCreateSerializer(serializers.ListSerializer):
    child = ProductCreateSerializer()

    def create(self, validated_data):
        products = [Product(**item) for item in validated_data]
        return Product.objects.bulk_create(products)

class ProductViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=["post"], url_path="bulk-create")
    def bulk_create(self, request):
        serializer = BulkProductCreateSerializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        products = serializer.save()
        output = ProductListSerializer(products, many=True)
        return Response(output.data, status=status.HTTP_201_CREATED)
```

### Bulk Update

```python
class BulkPriceUpdateSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    price = serializers.DecimalField(max_digits=10, decimal_places=2)

class ProductViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=["patch"], url_path="bulk-update-prices")
    def bulk_update_prices(self, request):
        serializer = BulkPriceUpdateSerializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)

        product_ids = [item["id"] for item in serializer.validated_data]
        products = {p.id: p for p in Product.objects.filter(id__in=product_ids)}

        updated = []
        for item in serializer.validated_data:
            product = products.get(item["id"])
            if product:
                product.price = item["price"]
                updated.append(product)

        Product.objects.bulk_update(updated, ["price"])
        return Response({"updated": len(updated)})
```

### Bulk Delete

```python
class BulkDeleteSerializer(serializers.Serializer):
    ids = serializers.ListField(child=serializers.IntegerField(), min_length=1, max_length=100)

class ProductViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=["post"], url_path="bulk-delete")
    def bulk_delete(self, request):
        serializer = BulkDeleteSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        deleted_count, _ = Product.objects.filter(
            id__in=serializer.validated_data["ids"]
        ).delete()
        return Response({"deleted": deleted_count})
```

---

## Error Handling

### Custom Exception Handler

```python
from rest_framework.views import exception_handler
from rest_framework.exceptions import APIException

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)

    if response is not None:
        response.data = {
            "error": {
                "code": response.status_code,
                "message": get_error_message(response.data),
                "details": response.data if isinstance(response.data, dict) else None,
            }
        }
    return response

def get_error_message(data):
    if isinstance(data, dict):
        if "detail" in data:
            return str(data["detail"])
        first_key = next(iter(data))
        first_value = data[first_key]
        if isinstance(first_value, list):
            return f"{first_key}: {first_value[0]}"
    return "An error occurred"
```

```python
# settings.py
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "apps.core.exceptions.custom_exception_handler",
}
```

### Custom API Exceptions

```python
class BusinessLogicException(APIException):
    status_code = 422
    default_detail = "A business rule was violated."
    default_code = "business_logic_error"

class InsufficientStockException(BusinessLogicException):
    default_detail = "Insufficient stock for this product."
    default_code = "insufficient_stock"
```
