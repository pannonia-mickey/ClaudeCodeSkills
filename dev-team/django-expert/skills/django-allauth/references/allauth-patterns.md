# django-allauth Advanced Patterns

This reference covers advanced patterns for django-allauth: headless API flows, DRF integration, social account linking, multi-factor authentication, custom user models, signals, and testing strategies.

---

## Headless API Flows

### Session-Based Authentication (Browser SPAs)

Session-based headless mode uses cookies for session management. Suitable for browser-based SPAs on the same domain.

```python
# urls.py
urlpatterns = [
    path("_allauth/browser/v1/", include("allauth.headless.urls")),
]
```

API endpoints exposed:
- `POST /_allauth/browser/v1/auth/signup` - Register
- `POST /_allauth/browser/v1/auth/login` - Login
- `DELETE /_allauth/browser/v1/auth/session` - Logout
- `GET /_allauth/browser/v1/auth/session` - Get current session
- `POST /_allauth/browser/v1/auth/password/request` - Request password reset
- `POST /_allauth/browser/v1/auth/password/reset` - Reset password
- `POST /_allauth/browser/v1/auth/email/verify` - Verify email
- `GET /_allauth/browser/v1/auth/providers` - List social providers

### Token-Based Authentication (Mobile/Cross-Domain)

Token-based headless mode returns an `X-Session-Token` header. Suitable for mobile apps or cross-domain clients.

```python
# urls.py
urlpatterns = [
    path("_allauth/app/v1/", include("allauth.headless.urls")),
]
```

The client must include the token in subsequent requests:
```
X-Session-Token: <token-value>
```

### Frontend URL Mapping

Map allauth internal URLs to frontend routes:

```python
HEADLESS_FRONTEND_URLS = {
    "account_confirm_email": "/verify-email/{key}",
    "account_reset_password_from_key": "/reset-password/{uidb36}/{key}",
    "socialaccount_login_cancelled": "/login/cancelled/",
    "socialaccount_login_error": "/login/error/",
}
```

These URLs are included in emails and redirects so the frontend can handle them.

---

## DRF Integration

### Authentication Class

Use `XSessionTokenAuthentication` to protect DRF views with allauth's headless token:

```python
from allauth.headless.contrib.rest_framework.authentication import (
    XSessionTokenAuthentication,
)
from rest_framework import permissions, viewsets
from rest_framework.views import APIView
from rest_framework.response import Response

class ProtectedAPIView(APIView):
    authentication_classes = [XSessionTokenAuthentication]
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        return Response({"email": request.user.email})
```

### Combining with Other Authentication

```python
from rest_framework_simplejwt.authentication import JWTAuthentication

class FlexibleAuthView(APIView):
    authentication_classes = [
        XSessionTokenAuthentication,  # allauth headless
        JWTAuthentication,            # simplejwt fallback
    ]
    permission_classes = [permissions.IsAuthenticated]
```

### Current User Endpoint

```python
from rest_framework import serializers

class UserSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    email = serializers.EmailField()
    username = serializers.CharField()
    has_mfa = serializers.SerializerMethodField()

    def get_has_mfa(self, obj):
        from allauth.mfa.utils import is_mfa_enabled
        return is_mfa_enabled(obj)

class CurrentUserView(APIView):
    authentication_classes = [XSessionTokenAuthentication]
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        serializer = UserSerializer(request.user)
        return Response(serializer.data)
```

---

## Social Account Linking

### Auto-Connect by Email

When a user logs in via a social provider and an account with the same email already exists, automatically link them:

```python
SOCIALACCOUNT_EMAIL_AUTHENTICATION = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True
```

### Manual Account Connection View

Allow authenticated users to connect/disconnect social accounts:

```python
# Template-based: allauth provides built-in views
# URL: /accounts/social/connections/

# For headless/DRF, use the headless API:
# GET /_allauth/browser/v1/auth/providers - list connected providers
# POST /_allauth/browser/v1/auth/provider/redirect - initiate connection
# DELETE /_allauth/browser/v1/auth/provider - disconnect provider
```

### Preventing Duplicate Accounts

```python
from allauth.socialaccount.adapter import DefaultSocialAccountAdapter

class PreventDuplicateAdapter(DefaultSocialAccountAdapter):
    def pre_social_login(self, request, sociallogin):
        """Connect social account to existing user if email matches."""
        if sociallogin.is_existing:
            return
        email = sociallogin.user.email
        if not email:
            return
        from django.contrib.auth import get_user_model
        User = get_user_model()
        try:
            existing_user = User.objects.get(email__iexact=email)
            sociallogin.connect(request, existing_user)
        except User.DoesNotExist:
            pass
```

---

## Multi-Factor Authentication (MFA)

### TOTP Setup

```python
INSTALLED_APPS += ["allauth.mfa"]

MFA_SUPPORTED_TYPES = ["totp", "recovery_codes"]
MFA_TOTP_ISSUER = "MyApp"
MFA_TOTP_PERIOD = 30  # Token validity period in seconds
MFA_TOTP_DIGITS = 6
MFA_RECOVERY_CODE_COUNT = 10
```

### Enforcing MFA for Specific Users

```python
from allauth.account.adapter import DefaultAccountAdapter
from allauth.mfa.utils import is_mfa_enabled

class MFAEnforcementAdapter(DefaultAccountAdapter):
    def get_login_redirect_url(self, request):
        """Redirect to MFA setup if not enabled for staff users."""
        if request.user.is_staff and not is_mfa_enabled(request.user):
            return "/accounts/2fa/totp/activate/"
        return super().get_login_redirect_url(request)
```

### MFA with Headless API

MFA challenges are returned as part of the headless authentication flow. When a user with MFA enabled logs in, the response includes an MFA challenge that the frontend must handle:

```json
{
    "status": 401,
    "data": {
        "flows": [
            {"id": "mfa_authenticate", "types": ["totp", "recovery_codes"]}
        ]
    }
}
```

The frontend then submits the TOTP code:
```
POST /_allauth/browser/v1/auth/2fa/authenticate
Content-Type: application/json

{"code": "123456"}
```

### Custom MFA Forms

```python
from allauth.mfa.totp.forms import ActivateTOTPForm

class CustomActivateTOTPForm(ActivateTOTPForm):
    def clean_code(self):
        code = super().clean_code()
        # Additional validation if needed
        return code

# settings.py
MFA_FORMS = {
    "activate_totp": "apps.accounts.forms.CustomActivateTOTPForm",
}
```

---

## Custom User Models

django-allauth works with custom user models. Ensure the custom model is set before running migrations.

### AbstractUser Extension

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    phone_number = models.CharField(max_length=20, blank=True)
    organization = models.ForeignKey(
        "organizations.Organization",
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
    )
    onboarded = models.BooleanField(default=False)

    class Meta:
        ordering = ["-date_joined"]

    def __str__(self):
        return self.email or self.username
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.CustomUser"
```

### Populating Custom Fields from Social Data

```python
from allauth.socialaccount.adapter import DefaultSocialAccountAdapter

class CustomSocialAdapter(DefaultSocialAccountAdapter):
    def populate_user(self, request, sociallogin, data):
        user = super().populate_user(request, sociallogin, data)
        # Map provider-specific fields to custom user fields
        extra_data = sociallogin.account.extra_data
        if sociallogin.account.provider == "google":
            user.phone_number = extra_data.get("phoneNumber", "")
        return user
```

---

## Signals

django-allauth emits signals for key authentication events:

```python
from allauth.account.signals import (
    user_signed_up,
    user_logged_in,
    user_logged_out,
    email_confirmed,
    password_changed,
    password_reset,
)
from allauth.socialaccount.signals import (
    social_account_added,
    social_account_removed,
    pre_social_login,
)

from django.dispatch import receiver

@receiver(user_signed_up)
def handle_user_signup(request, user, **kwargs):
    """Run post-signup tasks."""
    from apps.accounts.tasks import send_welcome_email, create_default_workspace
    send_welcome_email.delay(user.pk)
    create_default_workspace.delay(user.pk)

@receiver(user_logged_in)
def handle_user_login(request, user, **kwargs):
    """Log login events for security audit."""
    import logging
    logger = logging.getLogger("auth")
    logger.info(
        "user_login email=%s ip=%s",
        user.email,
        request.META.get("REMOTE_ADDR"),
    )

@receiver(email_confirmed)
def handle_email_confirmed(request, email_address, **kwargs):
    """Activate features that require verified email."""
    user = email_address.user
    user.email_verified = True
    user.save(update_fields=["email_verified"])

@receiver(social_account_added)
def handle_social_account_added(request, sociallogin, **kwargs):
    """Track social account connections."""
    import logging
    logger = logging.getLogger("auth")
    logger.info(
        "social_connected user=%s provider=%s",
        sociallogin.user.email,
        sociallogin.account.provider,
    )
```

---

## Testing

### Test Configuration

```python
# config/settings/testing.py
ACCOUNT_EMAIL_VERIFICATION = "none"  # Speed up tests
ACCOUNT_RATE_LIMITS = {}  # Disable rate limiting in tests
SOCIALACCOUNT_AUTO_SIGNUP = True
```

### Testing Signup and Login

```python
import pytest
from django.test import Client
from django.urls import reverse

@pytest.mark.django_db
class TestAccountSignup:
    def test_signup_creates_user(self, client):
        response = client.post(reverse("account_signup"), {
            "email": "test@example.com",
            "password1": "SecureP@ss123!",
            "password2": "SecureP@ss123!",
        })
        assert response.status_code in (200, 302)
        from django.contrib.auth import get_user_model
        assert get_user_model().objects.filter(email="test@example.com").exists()

    def test_login_with_email(self, client):
        from apps.accounts.factories import UserFactory
        user = UserFactory(email="user@example.com")
        user.set_password("TestP@ss123!")
        user.save()
        response = client.post(reverse("account_login"), {
            "login": "user@example.com",
            "password": "TestP@ss123!",
        })
        assert response.status_code == 302
```

### Testing Social Login

```python
from allauth.socialaccount.models import SocialApp, SocialAccount, SocialToken
from django.contrib.sites.models import Site

@pytest.fixture
def google_social_app(db):
    site = Site.objects.get_current()
    app = SocialApp.objects.create(
        provider="google",
        name="Google",
        client_id="test-client-id",
        secret="test-secret",
    )
    app.sites.add(site)
    return app

@pytest.fixture
def user_with_google(google_social_app):
    from apps.accounts.factories import UserFactory
    user = UserFactory()
    SocialAccount.objects.create(
        user=user,
        provider="google",
        uid="google-uid-123",
        extra_data={"email": user.email, "name": "Test User"},
    )
    return user

@pytest.mark.django_db
def test_user_has_social_account(user_with_google):
    assert user_with_google.socialaccount_set.filter(provider="google").exists()
```

### Testing Headless API

```python
import pytest
from rest_framework.test import APIClient

@pytest.fixture
def headless_client():
    return APIClient()

@pytest.mark.django_db
class TestHeadlessAuth:
    def test_signup_via_api(self, headless_client):
        response = headless_client.post(
            "/_allauth/browser/v1/auth/signup",
            {
                "email": "api@example.com",
                "password": "SecureP@ss123!",
            },
            format="json",
        )
        assert response.status_code in (200, 401)  # 401 if email verification required

    def test_login_via_api(self, headless_client):
        from apps.accounts.factories import UserFactory
        user = UserFactory(email="api@example.com")
        user.set_password("TestP@ss123!")
        user.save()
        response = headless_client.post(
            "/_allauth/browser/v1/auth/login",
            {
                "email": "api@example.com",
                "password": "TestP@ss123!",
            },
            format="json",
        )
        assert response.status_code == 200

    def test_session_info(self, headless_client):
        from apps.accounts.factories import UserFactory
        user = UserFactory()
        headless_client.force_login(user)
        response = headless_client.get("/_allauth/browser/v1/auth/session")
        assert response.status_code == 200
        assert response.json()["data"]["user"]["email"] == user.email
```

### Testing Custom Adapters

```python
from unittest.mock import patch
from allauth.account.adapter import DefaultAccountAdapter

@pytest.mark.django_db
class TestCustomAdapter:
    def test_blocked_email_domain(self):
        from apps.accounts.adapters import CustomAccountAdapter
        adapter = CustomAccountAdapter()
        with pytest.raises(Exception):
            adapter.clean_email("user@tempmail.com")

    def test_allowed_email_domain(self):
        from apps.accounts.adapters import CustomAccountAdapter
        adapter = CustomAccountAdapter()
        email = adapter.clean_email("user@company.com")
        assert email == "user@company.com"

    def test_signup_closed(self, client, settings):
        settings.ACCOUNT_ALLOW_SIGNUPS = False
        response = client.get(reverse("account_signup"))
        assert response.status_code == 302  # Redirected away
```

---

## Migration from django.contrib.auth Views

When migrating from Django's built-in auth views to allauth:

1. Replace `django.contrib.auth.urls` with `allauth.urls` in URL configuration.
2. Update templates from `registration/` to `account/` template directory.
3. Replace `LoginView`, `LogoutView`, `PasswordChangeView` references with allauth equivalents.
4. Move from `LOGOUT_REDIRECT_URL` to `ACCOUNT_LOGOUT_REDIRECT_URL`.
5. Replace `@login_required` usage with allauth's `LoginRequiredMiddleware` or keep `@login_required` (both work).
6. Run `python manage.py migrate` to create allauth database tables.
7. Existing users continue to work; allauth creates `EmailAddress` records on first login.

---

## Production Checklist

1. **Email verification**: Set `ACCOUNT_EMAIL_VERIFICATION = "mandatory"` for production.
2. **HTTPS**: Ensure all OAuth callback URLs use HTTPS.
3. **Credentials**: Store all OAuth client IDs and secrets in environment variables.
4. **Rate limiting**: Configure `ACCOUNT_RATE_LIMITS` to prevent brute-force attacks.
5. **CSRF**: Ensure `CsrfViewMiddleware` is enabled (allauth uses CSRF protection).
6. **Sites framework**: Verify `SITE_ID` matches the correct `Site` record for the deployment domain.
7. **Social providers**: Test OAuth flows in production with real credentials (not test/sandbox).
8. **MFA**: Consider enforcing MFA for admin and staff users.
9. **Token storage**: Only set `SOCIALACCOUNT_STORE_TOKENS = True` if accessing provider APIs post-login.
10. **Email backend**: Configure a production email backend (SMTP, SES, SendGrid) for verification emails.
