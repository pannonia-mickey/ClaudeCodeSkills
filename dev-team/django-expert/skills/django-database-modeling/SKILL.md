---
name: Django Database Modeling
description: This skill should be used when the user asks about "Django model", "Django ORM", "database design", "Django migration", or "Django queryset". It covers Django model design, field types, relationships, custom managers, queryset composition, migration strategies, and database indexing. Use this skill for designing Django models, optimizing database queries, writing data migrations, choosing between inheritance strategies, or implementing advanced ORM patterns like soft delete, audit trails, and polymorphic models.
---

## Model Design Fundamentals

Design models to represent a single domain entity each. Follow normalization rules unless a documented performance requirement justifies denormalization.

### Field Type Selection

Choose the most specific field type for each column. Specific types enable database-level constraints and improve query planning.

```python
from django.db import models
import uuid

class Product(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock_quantity = models.PositiveIntegerField(default=0)
    weight_kg = models.FloatField(null=True, blank=True)
    is_active = models.BooleanField(default=True, db_index=True)
    metadata = models.JSONField(default=dict, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]
        constraints = [
            models.CheckConstraint(check=models.Q(price__gte=0), name="positive_price"),
            models.UniqueConstraint(fields=["name", "category"], name="unique_product_per_category"),
        ]

    def __str__(self):
        return self.name
```

Use `DecimalField` for money (never `FloatField`). Use `PositiveIntegerField` for quantities. Use `JSONField` for semi-structured data that does not need relational queries. Always provide `__str__`, `Meta.ordering`, and `db_index` on frequently filtered fields.

### Relationship Design

#### ForeignKey (Many-to-One)

```python
class OrderItem(models.Model):
    order = models.ForeignKey("Order", on_delete=models.CASCADE, related_name="items")
    product = models.ForeignKey("Product", on_delete=models.PROTECT, related_name="order_items")
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
```

Choose `on_delete` deliberately:
- `CASCADE`: Delete children when parent is deleted (order items when order is deleted).
- `PROTECT`: Prevent deletion of parent if children exist (cannot delete product with orders).
- `SET_NULL`: Set FK to NULL on parent deletion (requires `null=True`).
- `SET_DEFAULT`: Set FK to a default value on parent deletion.
- `RESTRICT`: Like PROTECT but allows deletion if the referencing object is also being deleted in the same transaction.

#### ManyToManyField

```python
class Article(models.Model):
    title = models.CharField(max_length=300)
    tags = models.ManyToManyField("Tag", blank=True, related_name="articles")
    authors = models.ManyToManyField(
        "Author",
        through="ArticleAuthor",
        related_name="articles",
    )

class ArticleAuthor(models.Model):
    article = models.ForeignKey(Article, on_delete=models.CASCADE)
    author = models.ForeignKey("Author", on_delete=models.CASCADE)
    role = models.CharField(max_length=50, choices=[("primary", "Primary"), ("contributor", "Contributor")])
    order = models.PositiveSmallIntegerField(default=0)

    class Meta:
        ordering = ["order"]
        unique_together = [("article", "author")]
```

Use a `through` model when the relationship carries additional data (role, order, date joined).

#### OneToOneField

```python
class UserProfile(models.Model):
    user = models.OneToOneField("auth.User", on_delete=models.CASCADE, related_name="profile")
    bio = models.TextField(blank=True)
    avatar_url = models.URLField(blank=True)
```

### Custom Managers and QuerySets

Define a custom `QuerySet` subclass for chainable domain-specific filters. Attach it as the default manager using `as_manager()`.

```python
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def in_stock(self):
        return self.filter(stock_quantity__gt=0)

    def by_category(self, category):
        return self.filter(category=category)

    def price_range(self, min_price=None, max_price=None):
        qs = self
        if min_price is not None:
            qs = qs.filter(price__gte=min_price)
        if max_price is not None:
            qs = qs.filter(price__lte=max_price)
        return qs

    def with_review_stats(self):
        return self.annotate(
            avg_rating=models.Avg("reviews__rating"),
            review_count=models.Count("reviews"),
        )

class Product(models.Model):
    objects = ProductQuerySet.as_manager()
```

Compose queryset methods in views and serializers:

```python
Product.objects.active().in_stock().price_range(min_price=10, max_price=100).with_review_stats()
```

## Migration Strategies

### Standard Migrations

Generate migrations after model changes:

```bash
python manage.py makemigrations app_name
python manage.py migrate
```

Always review generated migration files before applying. Check for unintended changes, correct dependencies, and proper ordering.

### Data Migrations

Create data migrations for populating or transforming data:

```python
from django.db import migrations

def populate_slugs(apps, schema_editor):
    Product = apps.get_model("products", "Product")
    from django.utils.text import slugify
    for product in Product.objects.filter(slug=""):
        product.slug = slugify(product.name)
        product.save(update_fields=["slug"])

def reverse_populate_slugs(apps, schema_editor):
    pass  # No reverse needed

class Migration(migrations.Migration):
    dependencies = [("products", "0003_add_slug_field")]
    operations = [
        migrations.RunPython(populate_slugs, reverse_populate_slugs),
    ]
```

Always provide a reverse function. Use `apps.get_model()` inside `RunPython` to get the historical model state, not direct imports.

### Zero-Downtime Migrations

For columns that require a NOT NULL constraint on an existing table with data:

1. Add the field as nullable (`null=True`).
2. Deploy and backfill data in a data migration.
3. Add the NOT NULL constraint in a subsequent migration with a default.

For renaming columns:

1. Add the new column.
2. Deploy code that writes to both old and new columns.
3. Backfill existing data from old to new column.
4. Deploy code that reads from new column only.
5. Remove the old column.

### Squashing Migrations

Reduce migration count with `squashmigrations` after features stabilize:

```bash
python manage.py squashmigrations app_name 0001 0010
```

Review the squashed migration, remove any `RunPython` operations that are no longer relevant, and test both forward and backward application.

## Indexing Strategy

Add indexes for fields that appear in `filter()`, `order_by()`, `exclude()`, and `JOIN` conditions. Use composite indexes for queries that filter on multiple fields together.

```python
class Meta:
    indexes = [
        models.Index(fields=["status", "created_at"]),
        models.Index(fields=["category", "-price"]),
        models.Index(
            fields=["status"],
            condition=models.Q(status="pending"),
            name="pending_items_idx",
        ),
    ]
```

Partial indexes (using `condition`) reduce index size and improve write performance when only a subset of rows match the filter condition.

Use `EXPLAIN ANALYZE` via `queryset.explain(analyze=True)` to verify that indexes are being used. Remove unused indexes to avoid unnecessary write overhead.

---

## References

For advanced model patterns including abstract models, proxy models, soft delete, audit trails, and tree structures, see [model-patterns.md](references/model-patterns.md).

For query optimization techniques including select_related, prefetch_related, subqueries, F/Q objects, and aggregation, see [query-optimization.md](references/query-optimization.md).
