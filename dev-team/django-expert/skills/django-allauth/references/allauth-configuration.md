# django-allauth Configuration Reference

This reference covers all major configuration settings, social provider setup, adapter customization, and email/template configuration for django-allauth.

---

## Account Settings Reference

### Authentication Method

```python
# Login by email only (recommended for most apps)
ACCOUNT_LOGIN_METHODS = {"email"}

# Login by username only
ACCOUNT_LOGIN_METHODS = {"username"}

# Login by either email or username
ACCOUNT_LOGIN_METHODS = {"email", "username"}
```

### Signup Fields

```python
# Email and password required, no username
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]

# Username, email, and password
ACCOUNT_SIGNUP_FIELDS = ["username*", "email*", "password1*", "password2*"]
```

Fields with `*` are required. Fields without `*` are optional.

### Email Verification

```python
# Require email verification before login
ACCOUNT_EMAIL_VERIFICATION = "mandatory"

# Send verification email but allow login without verifying
ACCOUNT_EMAIL_VERIFICATION = "optional"

# No email verification
ACCOUNT_EMAIL_VERIFICATION = "none"

# Use code-based verification (6-digit code) instead of link
ACCOUNT_EMAIL_VERIFICATION_BY_CODE_ENABLED = True
```

### Session and Security

```python
# Log out other sessions on login
ACCOUNT_SESSION_REMEMBER = True

# Require re-authentication for sensitive actions
ACCOUNT_REAUTHENTICATION_REQUIRED = True

# Password change logs out other sessions
ACCOUNT_LOGOUT_ON_PASSWORD_CHANGE = True

# Rate limiting for login attempts
ACCOUNT_RATE_LIMITS = {
    "login": "5/m/key",
    "login_failed": "3/m/key,5/5m/key",
    "signup": "5/m/ip",
    "confirm_email": "1/3m/key",
    "password_reset": "1/m/ip",
}
```

### Redirect URLs

```python
LOGIN_REDIRECT_URL = "/dashboard/"
ACCOUNT_LOGOUT_REDIRECT_URL = "/"
ACCOUNT_SIGNUP_REDIRECT_URL = "/welcome/"

# Or use a callable in the adapter for dynamic redirects
ACCOUNT_ADAPTER = "apps.accounts.adapters.CustomAccountAdapter"
```

---

## Social Provider Configuration

### Provider Setup Pattern

Each social provider requires:
1. Add the provider app to `INSTALLED_APPS`
2. Configure credentials in `SOCIALACCOUNT_PROVIDERS` or via Django admin `SocialApp` model
3. Register the OAuth application with the provider's developer console

### Google

```python
INSTALLED_APPS += ["allauth.socialaccount.providers.google"]

SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "SCOPE": ["profile", "email"],
        "AUTH_PARAMS": {"access_type": "online"},
        "EMAIL_AUTHENTICATION": True,
        "VERIFIED_EMAIL": True,
        "APP": {
            "client_id": os.environ["GOOGLE_CLIENT_ID"],
            "secret": os.environ["GOOGLE_CLIENT_SECRET"],
            "key": "",
        },
    },
}
```

Google OAuth2 setup:
1. Create a project at https://console.cloud.google.com
2. Enable the Google+ API or People API
3. Create OAuth 2.0 credentials (Web application)
4. Set authorized redirect URI: `https://yourdomain.com/accounts/google/login/callback/`

### GitHub

```python
INSTALLED_APPS += ["allauth.socialaccount.providers.github"]

SOCIALACCOUNT_PROVIDERS = {
    "github": {
        "SCOPE": ["user:email"],
        "VERIFIED_EMAIL": True,
        "APP": {
            "client_id": os.environ["GITHUB_CLIENT_ID"],
            "secret": os.environ["GITHUB_CLIENT_SECRET"],
            "key": "",
        },
    },
}
```

GitHub OAuth setup:
1. Go to GitHub Developer Settings > OAuth Apps > New OAuth App
2. Set Authorization callback URL: `https://yourdomain.com/accounts/github/login/callback/`

### Microsoft / Azure AD

```python
INSTALLED_APPS += ["allauth.socialaccount.providers.microsoft"]

SOCIALACCOUNT_PROVIDERS = {
    "microsoft": {
        "SCOPE": ["openid", "email", "profile", "User.Read"],
        "APP": {
            "client_id": os.environ["AZURE_CLIENT_ID"],
            "secret": os.environ["AZURE_CLIENT_SECRET"],
            "key": "",
        },
        "TENANT": "common",  # "common", "organizations", "consumers", or specific tenant ID
    },
}
```

### Facebook

```python
INSTALLED_APPS += ["allauth.socialaccount.providers.facebook"]

SOCIALACCOUNT_PROVIDERS = {
    "facebook": {
        "METHOD": "oauth2",
        "SCOPE": ["email", "public_profile"],
        "FIELDS": [
            "id",
            "email",
            "name",
            "first_name",
            "last_name",
        ],
        "VERIFIED_EMAIL": False,
        "VERSION": "v18.0",
        "APP": {
            "client_id": os.environ["FACEBOOK_APP_ID"],
            "secret": os.environ["FACEBOOK_APP_SECRET"],
            "key": "",
        },
    },
}
```

### Apple

```python
INSTALLED_APPS += ["allauth.socialaccount.providers.apple"]

SOCIALACCOUNT_PROVIDERS = {
    "apple": {
        "APP": {
            "client_id": os.environ["APPLE_SERVICE_ID"],
            "secret": os.environ["APPLE_KEY_ID"],
            "key": os.environ["APPLE_TEAM_ID"],
            "certificate_key": os.environ["APPLE_PRIVATE_KEY"],
        },
    },
}
```

### Global Social Account Settings

```python
# Skip the signup form if social account provides enough info
SOCIALACCOUNT_AUTO_SIGNUP = True

# Automatically connect social accounts by matching email
SOCIALACCOUNT_EMAIL_AUTHENTICATION = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True

# Store OAuth tokens (needed if accessing provider APIs later)
SOCIALACCOUNT_STORE_TOKENS = True

# Social account email verification
SOCIALACCOUNT_EMAIL_REQUIRED = True
SOCIALACCOUNT_EMAIL_VERIFICATION = "mandatory"
```

### Database-Based Provider Configuration

Instead of storing credentials in settings, use the `SocialApp` model via Django admin:

1. Navigate to Django admin > Social Applications > Add
2. Select the provider
3. Enter the client ID and secret
4. Associate with the appropriate `Site`

This approach is useful for multi-tenant setups or when credentials need to be managed by non-developers.

---

## Custom Adapters

### Account Adapter

Override `DefaultAccountAdapter` and set `ACCOUNT_ADAPTER` in settings:

```python
# apps/accounts/adapters.py
from allauth.account.adapter import DefaultAccountAdapter
from django.conf import settings

class CustomAccountAdapter(DefaultAccountAdapter):

    def is_open_for_signup(self, request):
        """Control signup availability."""
        return getattr(settings, "ACCOUNT_ALLOW_SIGNUPS", True)

    def get_login_redirect_url(self, request):
        """Dynamic redirect based on user role."""
        if request.user.is_staff:
            return "/admin/dashboard/"
        if hasattr(request.user, "profile") and not request.user.profile.onboarded:
            return "/onboarding/"
        return "/dashboard/"

    def save_user(self, request, user, form, commit=True):
        """Customize user creation during signup."""
        user = super().save_user(request, user, form, commit=False)
        # Set custom fields
        if commit:
            user.save()
        return user

    def send_mail(self, template_prefix, email, context):
        """Override email sending to add custom context or use a different backend."""
        context["support_email"] = settings.SUPPORT_EMAIL
        context["company_name"] = settings.COMPANY_NAME
        super().send_mail(template_prefix, email, context)

    def clean_email(self, email):
        """Block disposable email domains."""
        email = super().clean_email(email)
        domain = email.split("@")[1].lower()
        blocked_domains = getattr(settings, "BLOCKED_EMAIL_DOMAINS", [])
        if domain in blocked_domains:
            raise self.validation_error("email_blacklisted")
        return email

    def clean_password(self, password, user=None):
        """Custom password validation."""
        password = super().clean_password(password, user)
        if user and user.username and user.username.lower() in password.lower():
            from django.core.exceptions import ValidationError
            raise ValidationError("Password cannot contain your username.")
        return password
```

```python
# settings.py
ACCOUNT_ADAPTER = "apps.accounts.adapters.CustomAccountAdapter"
```

### Social Account Adapter

```python
from allauth.socialaccount.adapter import DefaultSocialAccountAdapter

class CustomSocialAccountAdapter(DefaultSocialAccountAdapter):

    def can_authenticate_by_email(self, login, email):
        """Control whether social login can match by email."""
        return super().can_authenticate_by_email(login, email)

    def is_email_verified(self, provider, email):
        """Trust email verification from specific providers."""
        trusted_providers = ["google", "github", "microsoft"]
        if provider.id in trusted_providers:
            return True
        return super().is_email_verified(provider, email)

    def pre_social_login(self, request, sociallogin):
        """Hook before social login completes."""
        # Example: block specific email domains from social login
        email = sociallogin.user.email
        if email and email.endswith("@blocked-domain.com"):
            from django.core.exceptions import PermissionDenied
            raise PermissionDenied("This email domain is not allowed.")
        super().pre_social_login(request, sociallogin)

    def populate_user(self, request, sociallogin, data):
        """Customize user fields from social account data."""
        user = super().populate_user(request, sociallogin, data)
        # Set additional fields from social data
        return user
```

```python
# settings.py
SOCIALACCOUNT_ADAPTER = "apps.accounts.adapters.CustomSocialAccountAdapter"
```

---

## Email Configuration

### Email Templates

allauth uses Django's template system for emails. Override templates by creating files at:

```
templates/
    account/
        email/
            email_confirmation_message.txt
            email_confirmation_subject.txt
            password_reset_key_message.txt
            password_reset_key_subject.txt
```

### Custom Email Backend

```python
# settings.py (production)
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = os.environ["EMAIL_HOST"]
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ["EMAIL_HOST_USER"]
EMAIL_HOST_PASSWORD = os.environ["EMAIL_HOST_PASSWORD"]
DEFAULT_FROM_EMAIL = "noreply@example.com"
```

```python
# settings.py (development)
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

---

## Template Customization

Override allauth HTML templates by placing files at matching paths:

```
templates/
    account/
        login.html
        signup.html
        logout.html
        email_confirm.html
        email.html                    # Email management
        password_change.html
        password_reset.html
        password_reset_done.html
        password_reset_from_key.html
        verification_sent.html
    socialaccount/
        login.html
        connections.html              # Social account management
        signup.html
    mfa/
        authenticate.html             # MFA challenge
        totp/
            activate_form.html
```

Each template receives allauth-specific context. Extend a base template and customize the content block:

```html
<!-- templates/account/login.html -->
{% extends "base.html" %}
{% load allauth %}

{% block content %}
<h1>Sign In</h1>
{% element form form=form method="POST" action=action_url %}
    {% slot body %}
        {% csrf_token %}
        {% element fields form=form %}{% endelement %}
    {% endslot %}
    {% slot actions %}
        {% element button type="submit" %}Sign In{% endelement %}
        <a href="{% url 'account_signup' %}">Create Account</a>
        <a href="{% url 'account_reset_password' %}">Forgot Password?</a>
    {% endslot %}
{% endelement %}

{% for provider in socialaccount_providers %}
    <a href="{% provider_login_url provider.id %}">Sign in with {{ provider.name }}</a>
{% endfor %}
{% endblock %}
```

---

## Complete Settings Reference

```python
# Full recommended configuration
INSTALLED_APPS = [
    "django.contrib.auth",
    "django.contrib.messages",
    "django.contrib.sites",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
    "allauth.mfa",
    "allauth.headless",
]

MIDDLEWARE = [
    # ... other middleware ...
    "allauth.account.middleware.AccountMiddleware",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

SITE_ID = 1

# Account
ACCOUNT_ADAPTER = "apps.accounts.adapters.CustomAccountAdapter"
ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
ACCOUNT_EMAIL_VERIFICATION = "mandatory"
ACCOUNT_EMAIL_VERIFICATION_BY_CODE_ENABLED = True
ACCOUNT_UNIQUE_EMAIL = True
ACCOUNT_SESSION_REMEMBER = True
ACCOUNT_REAUTHENTICATION_REQUIRED = True
ACCOUNT_LOGOUT_ON_PASSWORD_CHANGE = True

# Social
SOCIALACCOUNT_ADAPTER = "apps.accounts.adapters.CustomSocialAccountAdapter"
SOCIALACCOUNT_AUTO_SIGNUP = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True
SOCIALACCOUNT_STORE_TOKENS = True

# MFA
MFA_SUPPORTED_TYPES = ["totp", "recovery_codes"]
MFA_TOTP_ISSUER = "MyApp"

# Headless
HEADLESS_ONLY = True
HEADLESS_FRONTEND_URLS = {
    "account_confirm_email": "/verify-email/{key}",
    "account_reset_password_from_key": "/reset-password/{uidb36}/{key}",
}

# Redirects
LOGIN_REDIRECT_URL = "/dashboard/"
ACCOUNT_LOGOUT_REDIRECT_URL = "/"
```
