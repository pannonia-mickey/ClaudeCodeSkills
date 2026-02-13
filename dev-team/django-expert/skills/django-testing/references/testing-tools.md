# Django Testing Tools

This reference provides configuration and usage guides for the essential tools in a Django test suite: pytest-django, factory_boy, coverage, VCR.py/responses, and freezegun.

---

## pytest-django

### Installation and Configuration

```bash
pip install pytest pytest-django
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.testing"
python_files = ["tests.py", "test_*.py", "*_tests.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
    "--reuse-db",
    "-p", "no:warnings",
]
markers = [
    "slow: marks tests as slow running",
    "integration: marks integration tests",
    "e2e: marks end-to-end tests",
]
filterwarnings = [
    "ignore::DeprecationWarning",
]
```

### Key Fixtures

```python
# db - enables database access for a single test
@pytest.mark.django_db
def test_model_creation():
    obj = MyModel.objects.create(name="test")
    assert obj.pk is not None

# client - Django test client
def test_homepage(client):
    response = client.get("/")
    assert response.status_code == 200

# admin_client - authenticated admin test client
def test_admin_page(admin_client):
    response = admin_client.get("/admin/")
    assert response.status_code == 200

# rf - RequestFactory instance
def test_view_with_request_factory(rf):
    request = rf.get("/test/")
    request.user = AnonymousUser()
    response = my_view(request)
    assert response.status_code == 302

# django_assert_num_queries - query count assertion
@pytest.mark.django_db
def test_optimized_query(django_assert_num_queries):
    ProductFactory.create_batch(10)
    with django_assert_num_queries(1):
        list(Product.objects.select_related("category").all())

# settings - override Django settings
def test_with_custom_setting(settings):
    settings.MAX_UPLOAD_SIZE = 1024
    assert settings.MAX_UPLOAD_SIZE == 1024

# live_server - starts a live Django server for e2e tests
@pytest.mark.django_db
def test_with_live_server(live_server):
    response = requests.get(f"{live_server.url}/health/")
    assert response.status_code == 200
```

### Custom Fixtures

```python
# conftest.py
import pytest
from rest_framework.test import APIClient
from apps.accounts.factories import UserFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def user(db):
    return UserFactory()

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client

@pytest.fixture
def admin_user(db):
    return UserFactory(is_staff=True, is_superuser=True)

@pytest.fixture
def admin_api_client(api_client, admin_user):
    api_client.force_authenticate(admin_user)
    return api_client
```

### Parametrize

```python
@pytest.mark.django_db
@pytest.mark.parametrize("status,expected_count", [
    ("active", 3),
    ("inactive", 2),
    ("archived", 1),
])
def test_filter_by_status(authenticated_client, status, expected_count):
    ProductFactory.create_batch(3, status="active")
    ProductFactory.create_batch(2, status="inactive")
    ProductFactory.create_batch(1, status="archived")
    response = authenticated_client.get(f"/api/v1/products/?status={status}")
    assert len(response.data["results"]) == expected_count
```

---

## factory_boy

### Installation

```bash
pip install factory_boy
```

### Basic Factory

```python
import factory
from factory.django import DjangoModelFactory
from django.contrib.auth import get_user_model

User = get_user_model()

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
        skip_postgeneration_save = True

    username = factory.Sequence(lambda n: f"user_{n}")
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    is_active = True

    @factory.post_generation
    def password(self, create, extracted, **kwargs):
        password = extracted or "defaultpass123"
        self.set_password(password)
        if create:
            self.save()
```

### SubFactory and RelatedFactory

```python
class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    customer = factory.SubFactory(UserFactory)
    status = "pending"
    total = factory.Faker("pydecimal", left_digits=3, right_digits=2, positive=True)

class OrderItemFactory(DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = factory.Faker("random_int", min=1, max=10)
    unit_price = factory.LazyAttribute(lambda obj: obj.product.price)
```

### Traits

```python
class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    class Params:
        completed = factory.Trait(
            status="completed",
            completed_at=factory.LazyFunction(timezone.now),
        )
        cancelled = factory.Trait(
            status="cancelled",
            cancelled_at=factory.LazyFunction(timezone.now),
            cancellation_reason="Customer request",
        )
        with_items = factory.Trait(
            items=factory.RelatedFactoryList(
                OrderItemFactory, factory_related_name="order", size=3,
            )
        )

# Usage
completed_order = OrderFactory(completed=True)
cancelled_order = OrderFactory(cancelled=True)
order_with_items = OrderFactory(with_items=True)
```

### Batch Creation

```python
# Create 10 products
products = ProductFactory.create_batch(10)

# Create 5 products with specific attributes
premium_products = ProductFactory.create_batch(5, price=999.99, is_active=True)

# Build without saving (for unit tests that do not need the database)
products = ProductFactory.build_batch(10)
```

---

## Coverage

### Installation and Configuration

```bash
pip install pytest-cov
```

```toml
# pyproject.toml
[tool.coverage.run]
source = ["apps"]
omit = [
    "*/migrations/*",
    "*/tests/*",
    "*/factories.py",
    "*/admin.py",
    "manage.py",
    "config/*",
]
branch = true

[tool.coverage.report]
show_missing = true
skip_covered = false
fail_under = 80
precision = 2
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "pass",
]

[tool.coverage.html]
directory = "htmlcov"
```

### Running with Coverage

```bash
# Run tests with coverage
pytest --cov=apps --cov-report=html --cov-report=term-missing

# Run coverage for specific app
pytest --cov=apps.products apps/products/tests/

# Fail if coverage drops below threshold
pytest --cov=apps --cov-fail-under=80
```

---

## VCR.py and responses

### VCR.py for Recording HTTP Interactions

VCR.py records HTTP interactions to cassette files and replays them in subsequent test runs. This eliminates flaky tests caused by external API dependencies.

```bash
pip install vcrpy pytest-recording
```

```python
@pytest.mark.vcr()
def test_fetch_weather_data():
    """First run records the HTTP interaction; subsequent runs replay it."""
    service = WeatherService()
    result = service.get_current_weather("New York")
    assert result["temperature"] is not None
    assert result["city"] == "New York"
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
vcr_cassette_dir = "tests/cassettes"
```

### responses Library for Mocking HTTP

`responses` provides programmatic HTTP mocking without cassette files.

```bash
pip install responses
```

```python
import responses

@responses.activate
def test_external_api_integration():
    responses.add(
        responses.GET,
        "https://api.example.com/products",
        json={"products": [{"id": 1, "name": "Widget"}]},
        status=200,
    )

    service = ExternalProductService()
    products = service.fetch_products()
    assert len(products) == 1
    assert products[0]["name"] == "Widget"

@responses.activate
def test_external_api_timeout():
    responses.add(
        responses.GET,
        "https://api.example.com/products",
        body=ConnectionError("Timeout"),
    )

    service = ExternalProductService()
    with pytest.raises(ServiceUnavailableError):
        service.fetch_products()
```

---

## freezegun

### Installation

```bash
pip install freezegun
```

### Freezing Time

```python
from freezegun import freeze_time

@freeze_time("2025-01-15 10:00:00")
@pytest.mark.django_db
def test_order_is_overdue():
    order = OrderFactory(
        due_date=timezone.datetime(2025, 1, 14, tzinfo=timezone.utc),
        status="pending",
    )
    assert order.is_overdue is True

@freeze_time("2025-01-15 10:00:00")
@pytest.mark.django_db
def test_order_is_not_overdue():
    order = OrderFactory(
        due_date=timezone.datetime(2025, 1, 16, tzinfo=timezone.utc),
        status="pending",
    )
    assert order.is_overdue is False
```

### Moving Time Forward

```python
from freezegun import freeze_time

@pytest.mark.django_db
def test_subscription_expires():
    with freeze_time("2025-01-01") as frozen:
        subscription = SubscriptionFactory(duration_days=30)
        assert subscription.is_active is True

        frozen.move_to("2025-02-01")
        subscription.refresh_from_db()
        assert subscription.is_active is False
```

### freezegun as pytest Fixture

```python
# conftest.py
@pytest.fixture
def frozen_time():
    with freeze_time("2025-06-15 12:00:00") as frozen:
        yield frozen

@pytest.mark.django_db
def test_created_at_timestamp(frozen_time):
    product = ProductFactory()
    assert product.created_at.date() == date(2025, 6, 15)
```

---

## time_machine (Alternative to freezegun)

`time_machine` is a faster alternative to `freezegun` that uses C extensions.

```bash
pip install time-machine
```

```python
import time_machine

@time_machine.travel("2025-01-15 10:00:00", tick=False)
@pytest.mark.django_db
def test_order_deadline():
    order = OrderFactory(due_date=timezone.datetime(2025, 1, 14, tzinfo=timezone.utc))
    assert order.is_overdue is True
```

Use `time_machine` for large test suites where `freezegun`'s Python-level patching creates measurable overhead.
