# DRF API Security

This reference covers authentication, authorization, and security patterns for Django REST Framework APIs: token authentication, JWT with simplejwt, OAuth2, object-level permissions, rate limiting, and CORS configuration.

---

## Token Authentication

### DRF Built-in Token Auth

DRF's `TokenAuthentication` assigns a single persistent token per user. Suitable for simple internal APIs but lacks expiration, refresh, and revocation capabilities.

```python
# settings.py
INSTALLED_APPS += ["rest_framework.authtoken"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
}
```

```bash
# Create token for a user
python manage.py drf_create_token username
```

```python
# views.py - custom login endpoint
from rest_framework.authtoken.models import Token
from django.contrib.auth import authenticate

class LoginView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        username = request.data.get("username")
        password = request.data.get("password")
        user = authenticate(username=username, password=password)
        if user is None:
            return Response({"error": "Invalid credentials"}, status=401)
        token, created = Token.objects.get_or_create(user=user)
        return Response({"token": token.key, "user_id": user.pk})
```

Client sends: `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`

### Limitations

- Tokens do not expire by default. Implement a custom model or use simplejwt instead.
- One token per user. Rotating a token invalidates all sessions.
- Token is stored in plain text in the database.

---

## JWT Authentication with simplejwt

`djangorestframework-simplejwt` provides stateless JWT tokens with access/refresh token pairs, expiration, token rotation, and blacklisting.

### Installation and Configuration

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "UPDATE_LAST_LOGIN": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": SECRET_KEY,
    "AUTH_HEADER_TYPES": ("Bearer",),
    "AUTH_HEADER_NAME": "HTTP_AUTHORIZATION",
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "user_id",
    "TOKEN_OBTAIN_SERIALIZER": "rest_framework_simplejwt.serializers.TokenObtainPairSerializer",
}

# Enable blacklisting
INSTALLED_APPS += ["rest_framework_simplejwt.token_blacklist"]
```

### URL Configuration

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain_pair"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
    path("api/token/verify/", TokenVerifyView.as_view(), name="token_verify"),
    path("api/token/blacklist/", TokenBlacklistView.as_view(), name="token_blacklist"),
]
```

### Custom Token Claims

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        # Add custom claims
        token["username"] = user.username
        token["email"] = user.email
        token["is_staff"] = user.is_staff
        token["roles"] = list(user.groups.values_list("name", flat=True))
        return token

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

### Token Flow

1. Client sends `POST /api/token/` with `username` and `password`.
2. Server returns `{"access": "<jwt>", "refresh": "<jwt>"}`.
3. Client sends `Authorization: Bearer <access_token>` on subsequent requests.
4. When access token expires, client sends `POST /api/token/refresh/` with the refresh token.
5. Server returns a new access token (and rotated refresh token if `ROTATE_REFRESH_TOKENS` is True).
6. On logout, client sends `POST /api/token/blacklist/` with the refresh token.

---

## OAuth2

### django-oauth-toolkit

For APIs that need third-party OAuth2 provider capabilities (authorization code, client credentials, PKCE).

```bash
pip install django-oauth-toolkit
```

```python
# settings.py
INSTALLED_APPS += ["oauth2_provider"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "oauth2_provider.contrib.rest_framework.OAuth2Authentication",
    ],
}

OAUTH2_PROVIDER = {
    "ACCESS_TOKEN_EXPIRE_SECONDS": 3600,
    "REFRESH_TOKEN_EXPIRE_SECONDS": 86400,
    "ROTATE_REFRESH_TOKEN": True,
    "SCOPES": {
        "read": "Read access",
        "write": "Write access",
        "products:read": "Read products",
        "products:write": "Write products",
        "orders:read": "Read orders",
        "orders:write": "Write orders",
    },
}
```

```python
# urls.py
urlpatterns = [
    path("o/", include("oauth2_provider.urls", namespace="oauth2_provider")),
]
```

### Scope-Based Permissions

```python
from oauth2_provider.contrib.rest_framework import TokenHasScope, TokenHasReadWriteScope

class ProductViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, TokenHasScope]
    required_scopes = ["products:read"]

    def get_permissions(self):
        if self.action in ("create", "update", "partial_update", "destroy"):
            self.required_scopes = ["products:write"]
        return super().get_permissions()
```

---

## Object-Level Permissions

### DRF Custom Object Permissions

```python
class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user

class IsOrganizationMember(BasePermission):
    def has_object_permission(self, request, view, obj):
        return request.user in obj.organization.members.all()
```

### django-guardian for Fine-Grained Permissions

`django-guardian` provides per-object permission assignment.

```bash
pip install django-guardian
```

```python
# settings.py
INSTALLED_APPS += ["guardian"]
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "guardian.backends.ObjectPermissionBackend",
]
```

```python
from guardian.shortcuts import assign_perm, get_objects_for_user, remove_perm

# Assign permission
assign_perm("change_project", user, project_instance)
assign_perm("view_project", team_group, project_instance)

# Check permission
user.has_perm("change_project", project_instance)  # True

# Get all objects user has permission on
projects = get_objects_for_user(user, "view_project", Project)

# Remove permission
remove_perm("change_project", user, project_instance)
```

### DRF Integration with django-guardian

```python
from rest_framework.permissions import BasePermission

class GuardianPermission(BasePermission):
    perms_map = {
        "GET": ["%(app_label)s.view_%(model_name)s"],
        "POST": ["%(app_label)s.add_%(model_name)s"],
        "PUT": ["%(app_label)s.change_%(model_name)s"],
        "PATCH": ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }

    def has_object_permission(self, request, view, obj):
        model_cls = obj.__class__
        kwargs = {
            "app_label": model_cls._meta.app_label,
            "model_name": model_cls._meta.model_name,
        }
        perms = [perm % kwargs for perm in self.perms_map.get(request.method, [])]
        return all(request.user.has_perm(perm, obj) for perm in perms)
```

---

## Rate Limiting

### Scoped Rate Limiting

Apply different rate limits to different endpoints:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [],
    "DEFAULT_THROTTLE_RATES": {
        "login": "5/minute",
        "signup": "3/hour",
        "api_default": "100/hour",
        "api_premium": "1000/hour",
        "password_reset": "3/hour",
    },
}
```

```python
from rest_framework.throttling import SimpleRateThrottle

class LoginRateThrottle(SimpleRateThrottle):
    scope = "login"

    def get_cache_key(self, request, view):
        ident = request.data.get("username", self.get_ident(request))
        return self.cache_format % {"scope": self.scope, "ident": ident}

class SignupRateThrottle(SimpleRateThrottle):
    scope = "signup"

    def get_cache_key(self, request, view):
        return self.cache_format % {
            "scope": self.scope,
            "ident": self.get_ident(request),
        }

class PremiumUserThrottle(SimpleRateThrottle):
    scope = "api_premium"

    def get_cache_key(self, request, view):
        if request.user.is_authenticated and request.user.is_premium:
            return self.cache_format % {
                "scope": self.scope,
                "ident": request.user.pk,
            }
        return None  # Do not throttle; fall through to next throttle class

class DefaultUserThrottle(SimpleRateThrottle):
    scope = "api_default"

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            return self.cache_format % {
                "scope": self.scope,
                "ident": request.user.pk,
            }
        return self.cache_format % {
            "scope": self.scope,
            "ident": self.get_ident(request),
        }
```

```python
class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
    permission_classes = [AllowAny]

    def post(self, request):
        ...

class ProductViewSet(viewsets.ModelViewSet):
    throttle_classes = [PremiumUserThrottle, DefaultUserThrottle]
```

### Custom Rate Limit Headers

```python
class ThrottleHeaderMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        if hasattr(request, "throttle_info"):
            info = request.throttle_info
            response["X-RateLimit-Limit"] = info["limit"]
            response["X-RateLimit-Remaining"] = info["remaining"]
            response["X-RateLimit-Reset"] = info["reset"]
        return response
```

---

## CORS Configuration

### django-cors-headers

```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS += ["corsheaders"]

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",  # Must be before CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    # ...
]
```

### Development Configuration

```python
# config/settings/development.py
CORS_ALLOW_ALL_ORIGINS = True  # Only in development
```

### Production Configuration

```python
# config/settings/production.py
CORS_ALLOW_ALL_ORIGINS = False
CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://admin.example.com",
]
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.example\.com$",
]
CORS_ALLOW_METHODS = [
    "DELETE",
    "GET",
    "OPTIONS",
    "PATCH",
    "POST",
    "PUT",
]
CORS_ALLOW_HEADERS = [
    "accept",
    "authorization",
    "content-type",
    "origin",
    "x-csrftoken",
    "x-requested-with",
    "x-tenant-id",
]
CORS_EXPOSE_HEADERS = [
    "x-request-id",
    "x-ratelimit-limit",
    "x-ratelimit-remaining",
]
CORS_ALLOW_CREDENTIALS = True
CORS_PREFLIGHT_MAX_AGE = 86400  # Cache preflight for 24 hours
```

### Per-View CORS

For endpoints that need different CORS rules than the global configuration:

```python
from django.views.decorators.csrf import csrf_exempt
from corsheaders.signals import check_request_enabled

def cors_allow_webhook(sender, request, **kwargs):
    """Allow CORS for webhook endpoints from any origin."""
    return request.path.startswith("/api/webhooks/")

check_request_enabled.connect(cors_allow_webhook)
```

---

## Security Checklist for APIs

1. **Authentication**: Use JWT (simplejwt) for stateless authentication. Set short access token lifetimes (15 minutes) and longer refresh token lifetimes (7 days).
2. **Authorization**: Apply permissions at both the view level (`permission_classes`) and the object level (`has_object_permission`).
3. **Input validation**: Validate all input through serializers. Never trust client data. Use `validate_<field>` and `validate()` methods.
4. **Rate limiting**: Apply rate limits to authentication endpoints (login, signup, password reset) and to data mutation endpoints.
5. **CORS**: Whitelist specific origins in production. Never use `CORS_ALLOW_ALL_ORIGINS = True` in production.
6. **HTTPS**: Enforce HTTPS with `SECURE_SSL_REDIRECT = True`. Set `SECURE_HSTS_SECONDS`.
7. **Sensitive data**: Never include passwords, tokens, or secrets in API responses. Use `write_only=True` on serializer fields for sensitive input.
8. **Enumeration prevention**: Use UUIDs instead of sequential integers for public-facing resource identifiers.
9. **SQL injection**: Use the ORM exclusively. Parameterize any raw SQL queries.
10. **Mass assignment**: Explicitly define `fields` on every serializer `Meta` class. Never use `fields = "__all__"` on write serializers.
11. **Error messages**: Do not leak internal details (stack traces, SQL, file paths) in production error responses. Use a custom exception handler.
12. **Logging**: Log authentication failures, permission denials, and rate limit hits. Feed logs to a SIEM or monitoring system.
