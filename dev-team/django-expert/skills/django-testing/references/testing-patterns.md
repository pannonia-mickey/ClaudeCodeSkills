# Django Testing Patterns

This reference covers testing patterns for Django components: models, views, serializers, signals, middleware, management commands, and integration/end-to-end tests.

---

## Unit Testing Models

### Testing Model Methods and Properties

```python
import pytest
from django.utils import timezone
from apps.orders.factories import OrderFactory

@pytest.mark.django_db
class TestOrderModel:
    def test_total_with_tax(self):
        order = OrderFactory(subtotal=100.00, tax_rate=0.08)
        assert order.total_with_tax == 108.00

    def test_is_overdue_when_past_due_date(self):
        order = OrderFactory(
            due_date=timezone.now() - timezone.timedelta(days=1),
            status="pending",
        )
        assert order.is_overdue is True

    def test_is_not_overdue_when_completed(self):
        order = OrderFactory(
            due_date=timezone.now() - timezone.timedelta(days=1),
            status="completed",
        )
        assert order.is_overdue is False
```

### Testing Model Constraints

```python
@pytest.mark.django_db
def test_unique_constraint_on_sku():
    ProductFactory(sku="ABC123")
    with pytest.raises(IntegrityError):
        ProductFactory(sku="ABC123")

@pytest.mark.django_db
def test_check_constraint_positive_price():
    with pytest.raises(IntegrityError):
        Product.objects.create(name="Bad Product", price=-10, category=CategoryFactory())
```

### Testing Custom Managers and QuerySets

```python
@pytest.mark.django_db
class TestProductQuerySet:
    def test_active_returns_only_active(self):
        ProductFactory(is_active=True)
        ProductFactory(is_active=False)
        assert Product.objects.active().count() == 1

    def test_in_stock_excludes_zero_quantity(self):
        ProductFactory(stock_quantity=10)
        ProductFactory(stock_quantity=0)
        assert Product.objects.in_stock().count() == 1

    def test_chainable_filters(self):
        cat = CategoryFactory()
        ProductFactory(is_active=True, stock_quantity=5, category=cat)
        ProductFactory(is_active=True, stock_quantity=0, category=cat)
        ProductFactory(is_active=False, stock_quantity=5, category=cat)
        result = Product.objects.active().in_stock().by_category(cat)
        assert result.count() == 1
```

## Unit Testing Serializers

### Input Validation

```python
@pytest.mark.django_db
class TestProductSerializer:
    def test_valid_data(self):
        category = CategoryFactory()
        data = {
            "name": "Widget",
            "price": "29.99",
            "category": category.pk,
        }
        serializer = ProductCreateSerializer(data=data)
        assert serializer.is_valid(), serializer.errors

    def test_missing_required_field(self):
        serializer = ProductCreateSerializer(data={"name": "Widget"})
        assert not serializer.is_valid()
        assert "price" in serializer.errors

    def test_negative_price_rejected(self):
        data = {"name": "Widget", "price": "-5.00", "category": CategoryFactory().pk}
        serializer = ProductCreateSerializer(data=data)
        assert not serializer.is_valid()
        assert "price" in serializer.errors

    def test_custom_validation_logic(self):
        data = {
            "name": "Widget",
            "price": "100.00",
            "sale_price": "150.00",  # sale_price > price should fail
            "category": CategoryFactory().pk,
        }
        serializer = ProductCreateSerializer(data=data)
        assert not serializer.is_valid()
        assert "non_field_errors" in serializer.errors
```

### Output Representation

```python
@pytest.mark.django_db
def test_product_list_serializer_output():
    product = ProductFactory(name="Widget", price=29.99)
    serializer = ProductListSerializer(product)
    data = serializer.data
    assert data["name"] == "Widget"
    assert "description" not in data  # List serializer excludes detail fields
    assert "id" in data
```

## Integration Testing Views

### Full Request/Response Cycle with APIClient

```python
@pytest.mark.django_db
class TestProductEndpoints:
    def test_create_product_returns_201(self, authenticated_client):
        category = CategoryFactory()
        payload = {"name": "Widget", "price": "29.99", "category": category.pk}
        response = authenticated_client.post("/api/v1/products/", payload, format="json")
        assert response.status_code == 201
        assert Product.objects.filter(name="Widget").exists()

    def test_update_product_returns_200(self, authenticated_client):
        product = ProductFactory()
        payload = {"name": "Updated Widget"}
        response = authenticated_client.patch(
            f"/api/v1/products/{product.pk}/", payload, format="json"
        )
        assert response.status_code == 200
        product.refresh_from_db()
        assert product.name == "Updated Widget"

    def test_delete_product_returns_204(self, authenticated_client):
        product = ProductFactory()
        response = authenticated_client.delete(f"/api/v1/products/{product.pk}/")
        assert response.status_code == 204

    def test_list_products_pagination(self, authenticated_client):
        ProductFactory.create_batch(30)
        response = authenticated_client.get("/api/v1/products/")
        assert response.status_code == 200
        assert "next" in response.data
        assert len(response.data["results"]) == 20  # Default page size
```

### Testing Permissions

```python
@pytest.mark.django_db
class TestProductPermissions:
    def test_anonymous_cannot_create(self, api_client):
        response = api_client.post("/api/v1/products/", {}, format="json")
        assert response.status_code == 401

    def test_regular_user_cannot_delete(self):
        user = UserFactory(is_staff=False)
        client = APIClient()
        client.force_authenticate(user)
        product = ProductFactory()
        response = client.delete(f"/api/v1/products/{product.pk}/")
        assert response.status_code == 403

    def test_admin_can_delete(self):
        admin = UserFactory(is_staff=True)
        client = APIClient()
        client.force_authenticate(admin)
        product = ProductFactory()
        response = client.delete(f"/api/v1/products/{product.pk}/")
        assert response.status_code == 204
```

## Testing Signals

Test signals by verifying the side effect, not by inspecting signal internals.

```python
@pytest.mark.django_db
def test_user_profile_created_on_user_creation():
    """post_save signal should create a UserProfile for new users."""
    user = User.objects.create_user(username="testuser", password="pass123")
    assert UserProfile.objects.filter(user=user).exists()

@pytest.mark.django_db
def test_order_notification_sent_on_status_change():
    """post_save signal should send notification when order status changes to shipped."""
    order = OrderFactory(status="processing")
    with patch("apps.orders.signals.send_notification") as mock_notify:
        order.status = "shipped"
        order.save()
        mock_notify.assert_called_once_with(order)
```

### Disconnecting Signals in Tests

When a signal causes unwanted side effects during testing, disconnect it temporarily:

```python
from django.db.models.signals import post_save
from apps.orders.signals import send_order_notification

@pytest.fixture
def no_order_signals():
    post_save.disconnect(send_order_notification, sender=Order)
    yield
    post_save.connect(send_order_notification, sender=Order)

@pytest.mark.django_db
def test_order_creation_without_notification(no_order_signals):
    order = OrderFactory()
    # Notification signal is disconnected
    assert order.status == "pending"
```

## Testing Middleware

### Unit Testing Middleware

```python
from django.test import RequestFactory
from apps.core.middleware import RequestTimingMiddleware

def test_timing_middleware_adds_header():
    factory = RequestFactory()
    request = factory.get("/test/")

    def get_response(request):
        from django.http import HttpResponse
        return HttpResponse("OK")

    middleware = RequestTimingMiddleware(get_response)
    response = middleware(request)
    assert "X-Request-Duration" in response

def test_tenant_middleware_sets_tenant():
    factory = RequestFactory()
    request = factory.get("/test/", HTTP_X_TENANT_ID="acme")

    def get_response(request):
        from django.http import HttpResponse
        return HttpResponse("OK")

    middleware = TenantMiddleware(get_response)
    middleware(request)
    assert hasattr(request, "tenant")
```

### Integration Testing Middleware

```python
@pytest.mark.django_db
def test_auth_middleware_rejects_unauthenticated(client):
    response = client.get("/api/v1/products/")
    assert response.status_code == 401
```

## Testing Management Commands

```python
from django.core.management import call_command
from io import StringIO

@pytest.mark.django_db
def test_cleanup_expired_orders_command():
    OrderFactory(status="pending", created_at=timezone.now() - timedelta(days=31))
    OrderFactory(status="pending", created_at=timezone.now() - timedelta(days=1))

    out = StringIO()
    call_command("cleanup_expired_orders", "--days=30", stdout=out)

    assert "Cancelled 1 expired order" in out.getvalue()
    assert Order.objects.filter(status="cancelled").count() == 1
    assert Order.objects.filter(status="pending").count() == 1
```

## Testing Celery Tasks

```python
from apps.orders.tasks import send_order_confirmation

@pytest.mark.django_db
def test_send_order_confirmation_task():
    order = OrderFactory()
    with patch("apps.orders.tasks.send_mail") as mock_mail:
        send_order_confirmation(order.pk)
        mock_mail.assert_called_once()
        assert order.user.email in mock_mail.call_args[0][3]

@pytest.mark.django_db
def test_send_order_confirmation_handles_missing_order():
    with pytest.raises(Order.DoesNotExist):
        send_order_confirmation(99999)
```

## End-to-End Testing

Use Django's `LiveServerTestCase` with Selenium or Playwright for browser-based e2e tests.

```python
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from playwright.sync_api import sync_playwright

class TestCheckoutFlow(StaticLiveServerTestCase):
    def test_user_can_complete_checkout(self):
        user = UserFactory()
        ProductFactory(name="Widget", price=29.99, stock_quantity=10)

        with sync_playwright() as p:
            browser = p.chromium.launch()
            page = browser.new_page()

            # Login
            page.goto(f"{self.live_server_url}/login/")
            page.fill("#id_username", user.username)
            page.fill("#id_password", "password123")
            page.click("button[type=submit]")

            # Add to cart
            page.goto(f"{self.live_server_url}/products/")
            page.click("text=Widget")
            page.click("text=Add to Cart")

            # Checkout
            page.click("text=Checkout")
            assert page.url.endswith("/checkout/confirm/")

            browser.close()
```

## Test Isolation Best Practices

- Use `@pytest.mark.django_db` to enable database access. Tests without this marker cannot touch the database.
- Use `@pytest.mark.django_db(transaction=True)` when testing code that uses `transaction.on_commit()`.
- Use `factory_boy` instead of JSON fixtures. Factories are composable and self-documenting.
- Use `freezegun` or `time_machine` for time-dependent tests.
- Use `override_settings` for tests that depend on specific settings values.

```python
from django.test import override_settings

@override_settings(MAX_UPLOAD_SIZE=1024)
def test_upload_rejects_large_files(authenticated_client):
    large_file = SimpleUploadedFile("big.txt", b"x" * 2048)
    response = authenticated_client.post("/api/v1/uploads/", {"file": large_file})
    assert response.status_code == 400
```
