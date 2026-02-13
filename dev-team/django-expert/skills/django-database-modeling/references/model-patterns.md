# Django Model Patterns

This reference documents reusable model patterns for Django projects, complete with code examples. Each pattern addresses a specific architectural need.

---

## TimeStampedModel

The foundational abstract model that provides automatic `created_at` and `updated_at` timestamps. Nearly every concrete model should inherit from this.

```python
from django.db import models

class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]
```

## UUIDModel

Use UUID primary keys for models exposed to external systems. UUIDs prevent enumeration attacks and simplify multi-database synchronization.

```python
import uuid
from django.db import models

class UUIDModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    class Meta:
        abstract = True
```

Combine with `TimeStampedModel` using multiple inheritance:

```python
class BaseModel(UUIDModel, TimeStampedModel):
    class Meta(UUIDModel.Meta, TimeStampedModel.Meta):
        abstract = True
```

## Soft Delete Pattern

Soft delete marks records as deleted without removing them from the database. This preserves referential integrity and enables recovery.

```python
from django.db import models
from django.utils import timezone

class SoftDeleteQuerySet(models.QuerySet):
    def delete(self):
        return self.update(deleted_at=timezone.now())

    def hard_delete(self):
        return super().delete()

    def alive(self):
        return self.filter(deleted_at__isnull=True)

    def dead(self):
        return self.filter(deleted_at__isnull=False)

class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return SoftDeleteQuerySet(self.model, using=self._db).alive()

class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True, db_index=True)

    objects = SoftDeleteManager()
    all_objects = SoftDeleteQuerySet.as_manager()

    class Meta:
        abstract = True

    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])

    def hard_delete(self, using=None, keep_parents=False):
        super().delete(using=using, keep_parents=keep_parents)

    def restore(self):
        self.deleted_at = None
        self.save(update_fields=["deleted_at"])
```

Usage:

```python
class Article(SoftDeleteModel, TimeStampedModel):
    title = models.CharField(max_length=300)
    body = models.TextField()

# Default manager excludes soft-deleted records
Article.objects.all()  # Only alive articles

# Access all records including soft-deleted
Article.all_objects.all()
Article.all_objects.dead()  # Only soft-deleted

# Soft delete
article.delete()

# Restore
article.restore()

# Permanent delete
article.hard_delete()
```

## Audit Trail Pattern

Track who created and last modified each record. Pair with middleware or signals to auto-populate the fields.

```python
from django.conf import settings
from django.db import models

class AuditModel(models.Model):
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        related_name="%(class)s_created",
    )
    modified_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        related_name="%(class)s_modified",
    )

    class Meta:
        abstract = True
```

Middleware to auto-populate audit fields:

```python
import threading

_thread_local = threading.local()

class CurrentUserMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        _thread_local.current_user = getattr(request, "user", None)
        return self.get_response(request)

def get_current_user():
    return getattr(_thread_local, "current_user", None)
```

Override `save()` on the audit model:

```python
class AuditModel(models.Model):
    # ... fields as above ...

    class Meta:
        abstract = True

    def save(self, *args, **kwargs):
        user = get_current_user()
        if user and user.is_authenticated:
            if not self.pk:
                self.created_by = user
            self.modified_by = user
        super().save(*args, **kwargs)
```

## Abstract Model Inheritance

Abstract models define shared fields and behavior. They do not create database tables. Concrete subclasses each get their own table with the inherited fields.

```python
class Publishable(models.Model):
    STATUS_CHOICES = [
        ("draft", "Draft"),
        ("review", "In Review"),
        ("published", "Published"),
        ("archived", "Archived"),
    ]

    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="draft", db_index=True)
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def publish(self):
        self.status = "published"
        self.published_at = timezone.now()
        self.save(update_fields=["status", "published_at"])

    def archive(self):
        self.status = "archived"
        self.save(update_fields=["status"])

    @property
    def is_published(self):
        return self.status == "published"

class BlogPost(Publishable, TimeStampedModel):
    title = models.CharField(max_length=300)
    body = models.TextField()

class NewsArticle(Publishable, TimeStampedModel):
    headline = models.CharField(max_length=300)
    content = models.TextField()
    source_url = models.URLField()
```

## Proxy Model Pattern

Proxy models share the same database table as their parent but can define different default managers, methods, and Meta options.

```python
class Order(TimeStampedModel):
    STATUS_CHOICES = [
        ("pending", "Pending"),
        ("processing", "Processing"),
        ("shipped", "Shipped"),
        ("delivered", "Delivered"),
        ("cancelled", "Cancelled"),
    ]

    customer = models.ForeignKey("Customer", on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    total = models.DecimalField(max_digits=10, decimal_places=2)

class PendingOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="pending")

class PendingOrder(Order):
    objects = PendingOrderManager()

    class Meta:
        proxy = True
        verbose_name = "Pending Order"

    def approve(self):
        self.status = "processing"
        self.save(update_fields=["status"])

class CancelledOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="cancelled")

class CancelledOrder(Order):
    objects = CancelledOrderManager()

    class Meta:
        proxy = True

    def refund(self):
        # Refund logic
        pass
```

Register proxy models in the admin to provide separate admin views for different order states.

## Multi-Table Inheritance

Multi-table inheritance creates a separate table for each model in the hierarchy with an implicit `OneToOneField` link. Use it when child models have substantially different fields.

```python
class Vehicle(TimeStampedModel):
    make = models.CharField(max_length=100)
    model_name = models.CharField(max_length=100)
    year = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=12, decimal_places=2)

class Car(Vehicle):
    num_doors = models.PositiveSmallIntegerField()
    trunk_volume_liters = models.FloatField()

class Truck(Vehicle):
    payload_capacity_kg = models.FloatField()
    num_axles = models.PositiveSmallIntegerField()
```

Querying `Vehicle.objects.all()` returns all vehicles. Accessing `vehicle.car` raises `Car.DoesNotExist` if the vehicle is actually a truck. Use `select_related("car", "truck")` to avoid N+1 queries when accessing child attributes.

Prefer abstract models when there is no need to query across the entire hierarchy.

## Polymorphic Pattern

For true polymorphic queries, use `django-polymorphic` or a manual type-discriminator column.

```python
class Content(TimeStampedModel):
    CONTENT_TYPES = [
        ("article", "Article"),
        ("video", "Video"),
        ("podcast", "Podcast"),
    ]

    content_type = models.CharField(max_length=20, choices=CONTENT_TYPES, db_index=True)
    title = models.CharField(max_length=300)
    author = models.ForeignKey("auth.User", on_delete=models.CASCADE)

    # Shared fields
    view_count = models.PositiveIntegerField(default=0)
    is_featured = models.BooleanField(default=False)

class ArticleContent(Content):
    body = models.TextField()
    reading_time_minutes = models.PositiveIntegerField()

    def save(self, *args, **kwargs):
        self.content_type = "article"
        super().save(*args, **kwargs)

class VideoContent(Content):
    video_url = models.URLField()
    duration_seconds = models.PositiveIntegerField()

    def save(self, *args, **kwargs):
        self.content_type = "video"
        super().save(*args, **kwargs)
```

## Tree Structures

Model hierarchical data using `django-mptt` or `django-treebeard` for efficient tree queries (ancestors, descendants, siblings).

### Self-Referential ForeignKey (Simple)

```python
class Category(TimeStampedModel):
    name = models.CharField(max_length=200)
    parent = models.ForeignKey(
        "self",
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name="children",
    )
    depth = models.PositiveSmallIntegerField(default=0)
    path = models.CharField(max_length=1000, blank=True, db_index=True)

    def save(self, *args, **kwargs):
        if self.parent:
            self.depth = self.parent.depth + 1
            self.path = f"{self.parent.path}/{self.pk or 'new'}"
        else:
            self.depth = 0
            self.path = str(self.pk or "new")
        super().save(*args, **kwargs)

    def get_ancestors(self):
        """Return all ancestors from root to parent."""
        if not self.parent:
            return Category.objects.none()
        ancestor_ids = []
        current = self.parent
        while current:
            ancestor_ids.append(current.pk)
            current = current.parent
        return Category.objects.filter(pk__in=ancestor_ids)

    def get_descendants(self):
        """Return all descendants using path prefix matching."""
        return Category.objects.filter(path__startswith=f"{self.path}/")
```

### django-mptt (Recommended for Complex Trees)

```python
from mptt.models import MPTTModel, TreeForeignKey

class Category(MPTTModel):
    name = models.CharField(max_length=200)
    parent = TreeForeignKey(
        "self",
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name="children",
    )

    class MPTTMeta:
        order_insertion_by = ["name"]

# Efficient tree queries
category.get_ancestors()
category.get_descendants(include_self=True)
category.get_siblings()
category.is_leaf_node()
Category.objects.all()  # Returns in tree order by default
```

## Status Machine Pattern

Model state transitions explicitly to prevent invalid status changes.

```python
class OrderStateMachine:
    TRANSITIONS = {
        "pending": ["processing", "cancelled"],
        "processing": ["shipped", "cancelled"],
        "shipped": ["delivered", "returned"],
        "delivered": ["returned"],
        "cancelled": [],
        "returned": [],
    }

    @classmethod
    def can_transition(cls, from_status, to_status):
        return to_status in cls.TRANSITIONS.get(from_status, [])

class Order(TimeStampedModel):
    status = models.CharField(max_length=20, default="pending")

    def transition_to(self, new_status):
        if not OrderStateMachine.can_transition(self.status, new_status):
            raise ValidationError(
                f"Cannot transition from '{self.status}' to '{new_status}'"
            )
        self.status = new_status
        self.save(update_fields=["status", "updated_at"])
```
