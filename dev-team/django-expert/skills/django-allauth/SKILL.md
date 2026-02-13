---
name: Django Allauth
description: This skill should be used when the user asks about "django-allauth", "social login", "social authentication", "Google login Django", "GitHub login Django", "allauth configuration", "Django email verification", "headless authentication", "Django MFA", "Django TOTP", or "allauth adapter". It covers django-allauth setup, social provider integration, email-based authentication, headless API mode for SPAs, multi-factor authentication, custom adapters, and DRF integration. Use this skill for adding authentication with django-allauth, configuring social login providers, setting up email verification flows, implementing headless auth for single-page applications, enabling MFA/TOTP, or customizing allauth behavior with adapters.
---

## Overview

django-allauth is a comprehensive Django authentication solution covering local account signup/login, social account authentication (OAuth2), multi-factor authentication (MFA/TOTP), and a headless JSON API for SPAs and mobile apps. It integrates with Django's `contrib.auth` and provides a unified, extensible interface for all authentication flows.

## Installation and Core Configuration

Install the package and configure the required Django settings:

```bash
pip install django-allauth
```

```python
# settings.py
INSTALLED_APPS = [
    "django.contrib.auth",
    "django.contrib.messages",
    "django.contrib.sites",
    # Required
    "allauth",
    "allauth.account",
    # Optional: social authentication
    "allauth.socialaccount",
    # Optional: MFA
    "allauth.mfa",
    # Optional: headless API
    "allauth.headless",
]

MIDDLEWARE = [
    # ... existing middleware ...
    "allauth.account.middleware.AccountMiddleware",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

SITE_ID = 1
```

### Account Settings

```python
# Authentication method: email-based login (recommended)
ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
ACCOUNT_EMAIL_VERIFICATION = "mandatory"  # "mandatory", "optional", "none"
ACCOUNT_EMAIL_VERIFICATION_BY_CODE_ENABLED = True
ACCOUNT_UNIQUE_EMAIL = True

# Redirects
LOGIN_REDIRECT_URL = "/dashboard/"
ACCOUNT_LOGOUT_REDIRECT_URL = "/"
```

### URL Configuration

```python
# urls.py
from django.urls import path, include

urlpatterns = [
    path("accounts/", include("allauth.urls")),
]
```

This registers all allauth views: login, logout, signup, email verification, password reset, password change, and social account management.

## Social Authentication

Add provider apps to `INSTALLED_APPS` and configure credentials in `SOCIALACCOUNT_PROVIDERS`:

```python
INSTALLED_APPS += [
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
]

SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "SCOPE": ["profile", "email"],
        "AUTH_PARAMS": {"access_type": "online"},
        "APP": {
            "client_id": os.environ["GOOGLE_CLIENT_ID"],
            "secret": os.environ["GOOGLE_CLIENT_SECRET"],
            "key": "",
        },
    },
    "github": {
        "SCOPE": ["user:email"],
        "APP": {
            "client_id": os.environ["GITHUB_CLIENT_ID"],
            "secret": os.environ["GITHUB_CLIENT_SECRET"],
            "key": "",
        },
    },
}

SOCIALACCOUNT_AUTO_SIGNUP = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True
```

Always load client IDs and secrets from environment variables. Never hardcode credentials.

## Headless API Mode

For SPAs (React, Vue, Angular) and mobile apps, enable headless mode to expose JSON API endpoints instead of server-rendered templates:

```python
INSTALLED_APPS += ["allauth.headless"]

# Disable traditional HTML views if using headless only
HEADLESS_ONLY = True

HEADLESS_FRONTEND_URLS = {
    "account_confirm_email": "/verify-email/{key}",
    "account_reset_password_from_key": "/reset-password/{uidb36}/{key}",
    "socialaccount_login_cancelled": "/login/cancelled/",
    "socialaccount_login_error": "/login/error/",
}
```

```python
# urls.py
urlpatterns = [
    # Session-based API (browser clients)
    path("_allauth/browser/v1/", include("allauth.headless.urls")),
    # Token-based API (mobile/app clients)
    path("_allauth/app/v1/", include("allauth.headless.urls")),
]
```

### DRF Integration

Secure DRF views with allauth's session token authentication:

```python
from allauth.headless.contrib.rest_framework.authentication import (
    XSessionTokenAuthentication,
)
from rest_framework import permissions
from rest_framework.views import APIView

class ProtectedAPIView(APIView):
    authentication_classes = [XSessionTokenAuthentication]
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        return Response({"user": request.user.email})
```

## Multi-Factor Authentication

Enable MFA with TOTP and recovery codes:

```python
INSTALLED_APPS += ["allauth.mfa"]

MFA_SUPPORTED_TYPES = ["totp", "recovery_codes"]
MFA_TOTP_ISSUER = "MyApp"

# Optional: customize MFA forms
MFA_FORMS = {
    "activate_totp": "allauth.mfa.totp.forms.ActivateTOTPForm",
    "deactivate_totp": "allauth.mfa.totp.forms.DeactivateTOTPForm",
    "generate_recovery_codes": "allauth.mfa.recovery_codes.forms.GenerateRecoveryCodesForm",
}
```

## Custom Adapters

Override `DefaultAccountAdapter` to customize signup, login redirects, email sending, and validation. Override `DefaultSocialAccountAdapter` for social auth customization.

```python
# adapters.py
from allauth.account.adapter import DefaultAccountAdapter

class CustomAccountAdapter(DefaultAccountAdapter):
    def is_open_for_signup(self, request):
        return getattr(settings, "ACCOUNT_ALLOW_SIGNUPS", True)

    def get_login_redirect_url(self, request):
        if request.user.is_staff:
            return "/admin/dashboard/"
        return "/dashboard/"

    def save_user(self, request, user, form, commit=True):
        user = super().save_user(request, user, form, commit=False)
        # Add custom fields from signup
        if commit:
            user.save()
        return user

    def clean_email(self, email):
        email = super().clean_email(email)
        domain = email.split("@")[1].lower()
        blocked = ["tempmail.com", "throwaway.com"]
        if domain in blocked:
            raise self.validation_error("email_blacklisted")
        return email
```

```python
# settings.py
ACCOUNT_ADAPTER = "apps.accounts.adapters.CustomAccountAdapter"
SOCIALACCOUNT_ADAPTER = "apps.accounts.adapters.CustomSocialAccountAdapter"
```

## Template Customization

Override allauth templates by creating matching paths under the project template directory:

```
templates/
    account/
        login.html
        signup.html
        email_confirm.html
        password_reset.html
    socialaccount/
        login.html
        connections.html
```

Each template receives context variables from allauth views. Extend a base template and override only the content block.

## Common Patterns

### Email-Only Authentication (No Username)

```python
ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
```

### Auto-Link Social Accounts to Existing Users by Email

```python
SOCIALACCOUNT_EMAIL_AUTHENTICATION = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True
```

### Invite-Only Signup

```python
# adapters.py
class InviteOnlyAdapter(DefaultAccountAdapter):
    def is_open_for_signup(self, request):
        return "invitation_code" in request.session
```

---

## References

For detailed configuration of all allauth settings, social provider setup, and adapter customization patterns, see [allauth-configuration.md](references/allauth-configuration.md).

For advanced patterns including headless API flows, MFA setup, DRF integration, social account linking, and testing strategies, see [allauth-patterns.md](references/allauth-patterns.md).
