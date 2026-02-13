---
name: Django Testing
description: This skill should be used when the user asks about "Django test", "pytest-django", "Django fixture", "test Django view", or "factory_boy Django". It covers Django testing strategies using both Django's built-in TestCase and pytest-django, along with factory_boy for test data generation, API testing with DRF's APIClient, and mocking techniques. Use this skill for writing unit tests for Django models, views, or serializers, setting up integration tests for API endpoints, configuring pytest-django, creating factory_boy factories, or establishing test isolation patterns.
---

## Testing Framework Choice

Prefer pytest-django over Django's built-in unittest-style TestCase. pytest offers simpler assertions, powerful fixtures, parametrize support, and a large plugin ecosystem.

### pytest-django Setup

```ini
# pytest.ini or pyproject.toml [tool.pytest.ini_options]
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.testing
python_files = tests.py test_*.py *_tests.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short --strict-markers --reuse-db
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks integration tests
```

```python
# config/settings/testing.py
from .base import *  # noqa

DEBUG = False
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",
    }
}
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
CACHES = {
    "default": {"BACKEND": "django.core.cache.backends.locmem.LocMemCache"}
}
DEFAULT_FILE_STORAGE = "django.core.files.storage.InMemoryStorage"
```

Using `:memory:` SQLite and MD5 password hasher speeds up test execution significantly.

## Test Organization

Organize tests in a `tests/` package within each app, with separate modules for models, views, serializers, and services.

```
apps/products/
    tests/
        __init__.py
        test_models.py
        test_views.py
        test_serializers.py
        test_services.py
    factories.py
```

## factory_boy Factories

Define factories alongside their app. Each factory mirrors its model and provides sensible defaults.

```python
# apps/products/factories.py
import factory
from factory.django import DjangoModelFactory
from apps.products.models import Product, Category

class CategoryFactory(DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"Category {n}")
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(" ", "-"))

class ProductFactory(DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Sequence(lambda n: f"Product {n}")
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(" ", "-"))
    description = factory.Faker("paragraph")
    price = factory.Faker("pydecimal", left_digits=3, right_digits=2, positive=True)
    stock_quantity = factory.Faker("random_int", min=0, max=500)
    is_active = True
    category = factory.SubFactory(CategoryFactory)
```

Use factories in tests instead of fixtures:

```python
def test_product_creation():
    product = ProductFactory(price=29.99, stock_quantity=100)
    assert product.price == 29.99
    assert product.stock_quantity == 100
    assert product.category is not None
```

### factory_boy Traits and Lazy Attributes

```python
class ProductFactory(DjangoModelFactory):
    class Meta:
        model = Product

    class Params:
        out_of_stock = factory.Trait(stock_quantity=0, is_active=False)
        premium = factory.Trait(
            price=factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True),
            name=factory.LazyAttribute(lambda obj: f"Premium {obj.name}"),
        )

# Usage
cheap_product = ProductFactory(price=5.00)
sold_out = ProductFactory(out_of_stock=True)
premium = ProductFactory(premium=True)
```

## Unit Testing Models

Test model methods, properties, constraints, and custom manager methods.

```python
import pytest
from django.core.exceptions import ValidationError
from apps.products.factories import ProductFactory

@pytest.mark.django_db
class TestProductModel:
    def test_str_representation(self):
        product = ProductFactory(name="Widget Pro")
        assert str(product) == "Widget Pro"

    def test_is_in_stock_when_quantity_positive(self):
        product = ProductFactory(stock_quantity=10)
        assert product.is_in_stock is True

    def test_is_not_in_stock_when_quantity_zero(self):
        product = ProductFactory(stock_quantity=0)
        assert product.is_in_stock is False

    def test_price_cannot_be_negative(self):
        product = ProductFactory.build(price=-10)
        with pytest.raises(ValidationError):
            product.full_clean()

    def test_active_manager_excludes_inactive(self):
        ProductFactory(is_active=True)
        ProductFactory(is_active=False)
        assert Product.objects.active().count() == 1
```

## Testing Views and API Endpoints

### DRF APIClient

```python
import pytest
from rest_framework.test import APIClient
from rest_framework import status
from apps.accounts.factories import UserFactory
from apps.products.factories import ProductFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client):
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client

@pytest.mark.django_db
class TestProductAPI:
    def test_list_products_unauthenticated(self, api_client):
        response = api_client.get("/api/v1/products/")
        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_list_products_authenticated(self, authenticated_client):
        ProductFactory.create_batch(5)
        response = authenticated_client.get("/api/v1/products/")
        assert response.status_code == status.HTTP_200_OK
        assert len(response.data["results"]) == 5

    def test_create_product(self, authenticated_client):
        category = CategoryFactory()
        payload = {
            "name": "New Widget",
            "price": "29.99",
            "category": category.pk,
        }
        response = authenticated_client.post("/api/v1/products/", payload)
        assert response.status_code == status.HTTP_201_CREATED
        assert response.data["name"] == "New Widget"

    def test_filter_by_category(self, authenticated_client):
        cat_a = CategoryFactory(name="Electronics")
        cat_b = CategoryFactory(name="Books")
        ProductFactory.create_batch(3, category=cat_a)
        ProductFactory.create_batch(2, category=cat_b)
        response = authenticated_client.get(f"/api/v1/products/?category={cat_a.pk}")
        assert len(response.data["results"]) == 3
```

### Django RequestFactory

Use `RequestFactory` for unit-testing views in isolation without URL routing overhead.

```python
from django.test import RequestFactory
from apps.products.views import ProductDetailView
from apps.accounts.factories import UserFactory

@pytest.mark.django_db
def test_product_detail_view():
    product = ProductFactory()
    user = UserFactory()
    factory = RequestFactory()
    request = factory.get(f"/products/{product.pk}/")
    request.user = user
    response = ProductDetailView.as_view()(request, pk=product.pk)
    assert response.status_code == 200
```

## Mocking

Use `unittest.mock.patch` to isolate units from external dependencies.

```python
from unittest.mock import patch, MagicMock

@pytest.mark.django_db
class TestOrderService:
    @patch("apps.orders.services.order_service.PaymentGateway")
    def test_create_order_charges_payment(self, MockGateway):
        mock_instance = MockGateway.return_value
        mock_instance.charge.return_value = {"id": "txn_123", "status": "succeeded"}

        service = OrderService()
        order = service.create_order(user=UserFactory(), items_data=[...])

        mock_instance.charge.assert_called_once()
        assert order.payment_status == "paid"

    @patch("apps.orders.services.order_service.send_mail")
    def test_create_order_sends_confirmation_email(self, mock_send_mail):
        service = OrderService()
        order = service.create_order(user=UserFactory(), items_data=[...])

        mock_send_mail.assert_called_once_with(
            "Order Confirmation",
            mock_send_mail.call_args[0][1],  # body
            None,
            [order.user.email],
        )
```

### Mocking ORM Queries

Avoid mocking the ORM in most tests. Use the test database with factories instead. Only mock ORM calls when testing service logic that should not depend on database behavior:

```python
@patch("apps.products.services.Product.objects")
def test_product_search_service_filters_correctly(self, mock_objects):
    mock_qs = MagicMock()
    mock_objects.filter.return_value = mock_qs
    mock_qs.order_by.return_value = mock_qs
    mock_qs.count.return_value = 5

    result = ProductSearchService().search(query="widget", sort="-price")

    mock_objects.filter.assert_called_once()
    mock_qs.order_by.assert_called_with("-price")
```

## Test Fixtures with pytest

```python
@pytest.fixture
def product_with_reviews(db):
    product = ProductFactory()
    ReviewFactory.create_batch(5, product=product, rating=4)
    ReviewFactory.create_batch(2, product=product, rating=2)
    return product

def test_average_rating(product_with_reviews):
    avg = product_with_reviews.reviews.aggregate(avg=Avg("rating"))["avg"]
    assert 3.0 < avg < 4.0
```

## Query Count Assertions

Prevent N+1 regressions by asserting query counts in tests:

```python
@pytest.mark.django_db
def test_product_list_query_count(authenticated_client, django_assert_num_queries):
    ProductFactory.create_batch(20)
    with django_assert_num_queries(2):  # 1 auth + 1 product list
        response = authenticated_client.get("/api/v1/products/")
        assert response.status_code == 200
```

---

## References

For detailed testing patterns covering signals, middleware, management commands, and integration test strategies, see [testing-patterns.md](references/testing-patterns.md).

For configuration guides on pytest-django, factory_boy recipes, coverage setup, VCR.py, and freezegun, see [testing-tools.md](references/testing-tools.md).
