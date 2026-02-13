# Django Query Optimization

This reference covers techniques for writing efficient Django ORM queries. Each section provides the pattern, when to use it, and concrete code examples.

---

## select_related

`select_related` performs a SQL JOIN and includes related objects in the same query. Use it for `ForeignKey` and `OneToOneField` relationships.

```python
# WITHOUT select_related: 1 + N queries
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)      # Hits the database each time
    print(order.customer.company)    # Another query if Company is FK on Customer

# WITH select_related: 1 query with JOINs
orders = Order.objects.select_related("customer", "customer__company").all()
for order in orders:
    print(order.customer.name)      # No additional query
    print(order.customer.company)   # No additional query
```

Chain multiple related fields:

```python
OrderItem.objects.select_related(
    "order",
    "order__customer",
    "product",
    "product__category",
)
```

Avoid `select_related` on nullable foreign keys if the majority of rows have NULL values, as the JOIN still executes. In those cases, `prefetch_related` may be more efficient.

## prefetch_related

`prefetch_related` executes a separate query for each relationship and performs the join in Python. Use it for `ManyToManyField`, reverse `ForeignKey`, and `GenericRelation`.

```python
# WITHOUT prefetch_related: 1 + N queries
categories = Category.objects.all()
for category in categories:
    print(category.products.count())  # Query per category

# WITH prefetch_related: 2 queries total
categories = Category.objects.prefetch_related("products").all()
for category in categories:
    print(category.products.count())  # No additional query (counted in Python)
```

### Prefetch Object for Filtered Relations

Use the `Prefetch` object when the related queryset needs filtering, annotation, or a custom queryset.

```python
from django.db.models import Prefetch

# Prefetch only active products with their review stats
categories = Category.objects.prefetch_related(
    Prefetch(
        "products",
        queryset=Product.objects.filter(is_active=True)
            .select_related("brand")
            .annotate(avg_rating=Avg("reviews__rating")),
        to_attr="active_products",
    )
)

for category in categories:
    for product in category.active_products:  # Uses to_attr (a list, not a manager)
        print(product.name, product.avg_rating)
```

`to_attr` stores the result as a Python list attribute instead of a manager. This avoids accidental re-queries and makes the code more explicit.

### Nested Prefetch

```python
authors = Author.objects.prefetch_related(
    Prefetch(
        "books",
        queryset=Book.objects.filter(published=True).prefetch_related(
            Prefetch(
                "reviews",
                queryset=Review.objects.filter(rating__gte=4),
                to_attr="top_reviews",
            )
        ),
        to_attr="published_books",
    )
)
```

## Subquery and OuterRef

Use `Subquery` to embed one queryset inside another. This is useful for correlated lookups where a related value needs to be annotated onto each row.

```python
from django.db.models import Subquery, OuterRef

# Annotate each customer with their most recent order date
latest_order = Order.objects.filter(
    customer=OuterRef("pk")
).order_by("-created_at").values("created_at")[:1]

customers = Customer.objects.annotate(
    last_order_date=Subquery(latest_order)
)
```

### Exists Subquery

```python
from django.db.models import Exists, OuterRef

# Find customers who have at least one order this year
recent_orders = Order.objects.filter(
    customer=OuterRef("pk"),
    created_at__year=2025,
)

active_customers = Customer.objects.annotate(
    has_recent_order=Exists(recent_orders)
).filter(has_recent_order=True)
```

`Exists` generates an `EXISTS(SELECT 1 ...)` subquery, which is typically faster than `COUNT > 0`.

## F Expressions

`F` expressions reference model field values directly in the database, enabling field-to-field comparisons and atomic updates without loading the object into Python.

```python
from django.db.models import F

# Atomic increment: no race condition
Product.objects.filter(pk=42).update(stock_quantity=F("stock_quantity") - 1)

# Field-to-field comparison
Order.objects.filter(shipped_at__gt=F("expected_delivery_date"))

# Arithmetic between fields
Product.objects.annotate(
    profit_margin=F("price") - F("cost")
).filter(profit_margin__gt=10)

# Duration arithmetic
from datetime import timedelta
Order.objects.filter(
    delivered_at__lte=F("created_at") + timedelta(days=3)
)
```

## Q Objects

`Q` objects enable complex lookups with `OR`, `AND`, and `NOT` logic.

```python
from django.db.models import Q

# OR condition
Product.objects.filter(
    Q(name__icontains="widget") | Q(description__icontains="widget")
)

# NOT condition
Product.objects.filter(~Q(status="discontinued"))

# Complex combination
Product.objects.filter(
    (Q(category="electronics") & Q(price__lt=500))
    | (Q(category="books") & Q(price__lt=50))
)
```

### Dynamic Q Construction

Build queries dynamically from user input:

```python
def search_products(filters):
    query = Q()
    if filters.get("name"):
        query &= Q(name__icontains=filters["name"])
    if filters.get("min_price"):
        query &= Q(price__gte=filters["min_price"])
    if filters.get("max_price"):
        query &= Q(price__lte=filters["max_price"])
    if filters.get("categories"):
        query &= Q(category__in=filters["categories"])
    return Product.objects.filter(query)
```

## Aggregation and Annotation

### Aggregation (Whole QuerySet)

```python
from django.db.models import Avg, Count, Max, Min, Sum

stats = Order.objects.aggregate(
    total_revenue=Sum("total"),
    avg_order_value=Avg("total"),
    order_count=Count("id"),
    largest_order=Max("total"),
    smallest_order=Min("total"),
)
# stats = {"total_revenue": Decimal("50000"), "avg_order_value": Decimal("125"), ...}
```

### Annotation (Per Object)

```python
# Annotate each category with product count and average price
categories = Category.objects.annotate(
    product_count=Count("products"),
    avg_product_price=Avg("products__price"),
).filter(product_count__gt=0).order_by("-product_count")
```

### Conditional Aggregation

```python
from django.db.models import Case, When, IntegerField

products = Product.objects.annotate(
    positive_reviews=Count(
        "reviews",
        filter=Q(reviews__rating__gte=4),
    ),
    negative_reviews=Count(
        "reviews",
        filter=Q(reviews__rating__lte=2),
    ),
)
```

### Group By with values()

```python
# Revenue by month
from django.db.models.functions import TruncMonth

monthly_revenue = (
    Order.objects
    .annotate(month=TruncMonth("created_at"))
    .values("month")
    .annotate(
        revenue=Sum("total"),
        order_count=Count("id"),
    )
    .order_by("month")
)
```

## .only() and .defer()

Limit which columns are fetched from the database. Use `only()` to specify the exact fields needed. Use `defer()` to exclude specific heavy fields.

```python
# Fetch only the fields needed for a list view
products = Product.objects.only("id", "name", "price", "thumbnail_url")

# Defer the heavy description field
products = Product.objects.defer("description", "full_specification")
```

Accessing a deferred field triggers a separate database query for that object. Only use these methods when profiling confirms the excluded fields are causing performance issues (e.g., large `TextField` or `JSONField` values).

## .values() and .values_list()

Return dictionaries or tuples instead of model instances. This avoids model instantiation overhead for read-only data.

```python
# Returns list of dicts
Product.objects.filter(is_active=True).values("id", "name", "price")

# Returns list of tuples
Product.objects.filter(is_active=True).values_list("id", "name", flat=False)

# Returns flat list of single field
product_names = Product.objects.values_list("name", flat=True)
```

## .iterator()

Process large querysets without loading all objects into memory at once.

```python
# Default: loads entire queryset into memory
for product in Product.objects.all():
    process(product)

# With iterator: fetches in server-side cursor batches
for product in Product.objects.all().iterator(chunk_size=2000):
    process(product)
```

Use `iterator()` when processing tens of thousands or more records in a loop. Note that `iterator()` disables the queryset cache, so subsequent access to the same queryset re-queries the database.

## Raw SQL

When the ORM cannot express the query efficiently, use `raw()` with parameterized queries or `connection.cursor()`.

```python
# Model.objects.raw() - returns model instances
products = Product.objects.raw(
    """
    SELECT p.*, COUNT(r.id) as review_count
    FROM products_product p
    LEFT JOIN reviews_review r ON r.product_id = p.id
    WHERE p.is_active = %s
    GROUP BY p.id
    HAVING COUNT(r.id) > %s
    ORDER BY review_count DESC
    """,
    [True, 5],
)

# connection.cursor() - returns raw rows
from django.db import connection

def get_dashboard_stats():
    with connection.cursor() as cursor:
        cursor.execute(
            """
            SELECT
                DATE_TRUNC('day', created_at) as day,
                COUNT(*) as orders,
                SUM(total) as revenue
            FROM orders_order
            WHERE created_at >= %s
            GROUP BY day
            ORDER BY day
            """,
            [timezone.now() - timedelta(days=30)],
        )
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]
```

Always use `%s` parameter placeholders. Never use f-strings or `.format()` for SQL construction.

## Database Functions

Django provides database functions that map to SQL functions for use in annotations, filters, and ordering.

```python
from django.db.models.functions import (
    Coalesce, Concat, Length, Lower, Upper,
    Now, ExtractYear, TruncDate,
    Cast, Greatest, Least,
)
from django.db.models import Value, CharField, IntegerField

# String functions
Product.objects.annotate(
    display_name=Concat("brand__name", Value(" - "), "name"),
    name_length=Length("name"),
    lower_sku=Lower("sku"),
)

# Null handling
Product.objects.annotate(
    effective_price=Coalesce("sale_price", "price"),
)

# Date functions
Order.objects.annotate(
    order_date=TruncDate("created_at"),
    order_year=ExtractYear("created_at"),
)

# Type casting
Product.objects.annotate(
    price_int=Cast("price", IntegerField()),
)

# Greatest / Least
Product.objects.annotate(
    max_dimension=Greatest("width", "height", "depth"),
)
```

## Query Debugging

### Counting Queries

```python
from django.test.utils import override_settings
from django.db import connection, reset_queries

reset_queries()
# ... execute code ...
print(f"Query count: {len(connection.queries)}")
for query in connection.queries:
    print(f"  {query['time']}s: {query['sql'][:200]}")
```

### assertNumQueries in Tests

```python
class ProductViewTest(TestCase):
    def test_product_list_query_count(self):
        create_test_products(50)
        with self.assertNumQueries(2):  # 1 for products, 1 for categories
            response = self.client.get("/api/products/")
            self.assertEqual(response.status_code, 200)
```

### EXPLAIN ANALYZE

```python
qs = Product.objects.filter(category="electronics", is_active=True).order_by("-price")
print(qs.explain(analyze=True))
# Output shows execution plan with actual timing
```
