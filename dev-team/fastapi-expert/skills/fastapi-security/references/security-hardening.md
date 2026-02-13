# Security Hardening for FastAPI

This reference covers production security hardening for FastAPI applications, including dependency vulnerability scanning, secrets management, data encryption, content-type enforcement, HTTP security headers, security testing, and deployment hardening.

---

## Dependency Vulnerability Scanning

### pip-audit

Scan Python dependencies for known vulnerabilities:

```bash
# Install
pip install pip-audit

# Scan current environment
pip-audit

# Scan a requirements file
pip-audit -r requirements.txt

# Output in JSON for CI integration
pip-audit --format json --output audit-results.json

# Fail CI on findings (exit code 1 if vulnerabilities found)
pip-audit --strict
```

### safety

```bash
pip install safety

# Scan installed packages
safety check

# Scan a requirements file
safety check -r requirements.txt --output json
```

### GitHub Actions Integration

```yaml
- name: Audit Python dependencies
  run: |
    pip install pip-audit
    pip-audit --strict --desc
```

### Pre-Commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/trailofbits/pip-audit
    rev: v2.7.0
    hooks:
      - id: pip-audit
        args: ["--strict"]
```

Run `pip-audit` regularly in CI and before deployments. Pin dependency versions in `requirements.txt` or `pyproject.toml` with exact versions to ensure reproducible builds.

---

## Secrets Management

### Environment Variables with pydantic-settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, SecretStr

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="APP_",
        case_sensitive=False,
    )

    # Use SecretStr for sensitive values — prevents accidental logging
    secret_key: SecretStr
    database_url: SecretStr
    api_secret: SecretStr

    # Non-sensitive settings use regular str
    app_name: str = "MyService"
    debug: bool = False
```

`SecretStr` redacts the value in `repr()`, `str()`, and serialization. To access the raw value:

```python
settings = Settings()

# This prints "**********"
print(settings.secret_key)

# This returns the actual value
actual_key = settings.secret_key.get_secret_value()
```

### Never Hardcode Secrets

```python
# BAD: Secret in source code
SECRET_KEY = "my-super-secret-key-2024"

# BAD: Secret in default parameter
def create_app(secret: str = "default-secret"):
    ...

# GOOD: Load from environment, fail if missing
class Settings(BaseSettings):
    secret_key: SecretStr  # No default — app won't start without it
```

### .env File Management

```gitignore
# .gitignore - ALWAYS exclude env files
.env
.env.local
.env.production
*.pem
*.key
credentials.json
```

Provide a `.env.example` with placeholder values:

```bash
# .env.example
APP_SECRET_KEY=change-me-to-a-random-string
APP_DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/mydb
APP_API_SECRET=change-me
```

### Generating Secure Keys

```python
import secrets

# Generate a secure random key
print(secrets.token_urlsafe(64))

# For hex-based keys
print(secrets.token_hex(32))
```

---

## Data Encryption

### Field-Level Encryption

To encrypt sensitive database fields at rest using Fernet symmetric encryption:

```python
from cryptography.fernet import Fernet
from sqlalchemy import TypeDecorator, String

class EncryptedString(TypeDecorator):
    """SQLAlchemy type that transparently encrypts/decrypts string values."""
    impl = String
    cache_ok = True

    def __init__(self, key: bytes, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fernet = Fernet(key)

    def process_bind_param(self, value, dialect):
        if value is not None:
            return self.fernet.encrypt(value.encode()).decode()
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            return self.fernet.decrypt(value.encode()).decode()
        return value

# Usage in a model
ENCRYPTION_KEY = Fernet.generate_key()

class UserProfile(Base):
    __tablename__ = "user_profiles"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    ssn: Mapped[str | None] = mapped_column(EncryptedString(ENCRYPTION_KEY, 500))
    tax_id: Mapped[str | None] = mapped_column(EncryptedString(ENCRYPTION_KEY, 500))
```

### Key Rotation

```python
class KeyRotator:
    """Supports decrypting with old keys while encrypting with the current key."""

    def __init__(self, current_key: bytes, old_keys: list[bytes]):
        self.current = Fernet(current_key)
        self.all_fernets = [Fernet(current_key)] + [Fernet(k) for k in old_keys]
        self.multi = MultiFernet(self.all_fernets)

    def encrypt(self, data: str) -> str:
        return self.current.encrypt(data.encode()).decode()

    def decrypt(self, token: str) -> str:
        return self.multi.decrypt(token.encode()).decode()
```

---

## Content-Type Enforcement

### Strict Content-Type Validation

To reject requests with unexpected content types:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class ContentTypeValidationMiddleware(BaseHTTPMiddleware):
    """Reject requests with unexpected Content-Type headers."""

    METHODS_REQUIRING_BODY = {"POST", "PUT", "PATCH"}
    ALLOWED_CONTENT_TYPES = {
        "application/json",
        "multipart/form-data",
        "application/x-www-form-urlencoded",
    }

    async def dispatch(self, request, call_next):
        if request.method in self.METHODS_REQUIRING_BODY:
            content_type = request.headers.get("content-type", "")
            base_type = content_type.split(";")[0].strip().lower()

            if base_type and base_type not in self.ALLOWED_CONTENT_TYPES:
                return JSONResponse(
                    status_code=415,
                    content={
                        "error": "UNSUPPORTED_MEDIA_TYPE",
                        "detail": f"Content-Type '{base_type}' is not supported",
                    },
                )

        return await call_next(request)

app.add_middleware(ContentTypeValidationMiddleware)
```

### JSON Deserialization Safety

Pydantic provides safe deserialization by default. Additional protections:

```python
from pydantic import BaseModel, ConfigDict

class StrictInput(BaseModel):
    """Base model with strict validation to prevent type coercion attacks."""
    model_config = ConfigDict(
        strict=True,            # Disallow type coercion (e.g., "1" -> 1)
        extra="forbid",         # Reject unknown fields
        str_strip_whitespace=True,
        str_max_length=10000,   # Global string length limit
    )
```

---

## HTTP Security Headers (Comprehensive)

### Full Security Headers Middleware

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # Prevent MIME type sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Prevent clickjacking
        response.headers["X-Frame-Options"] = "DENY"

        # XSS filter (legacy browsers)
        response.headers["X-XSS-Protection"] = "0"

        # Force HTTPS
        response.headers["Strict-Transport-Security"] = (
            "max-age=63072000; includeSubDomains; preload"
        )

        # Control referrer information
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # Restrict browser features
        response.headers["Permissions-Policy"] = (
            "camera=(), microphone=(), geolocation=(), "
            "payment=(), usb=(), magnetometer=()"
        )

        # Content Security Policy (adjust per application)
        response.headers["Content-Security-Policy"] = (
            "default-src 'none'; "
            "script-src 'self'; "
            "style-src 'self'; "
            "img-src 'self' data:; "
            "font-src 'self'; "
            "connect-src 'self'; "
            "frame-ancestors 'none'; "
            "base-uri 'self'; "
            "form-action 'self'"
        )

        # Prevent caching of sensitive responses
        if request.url.path.startswith("/api/"):
            response.headers["Cache-Control"] = "no-store"
            response.headers["Pragma"] = "no-cache"

        # Remove server identification
        response.headers.pop("server", None)
        response.headers.pop("x-powered-by", None)

        return response
```

### Notes on X-XSS-Protection

Set `X-XSS-Protection: 0` (disabled) rather than `1; mode=block`. The browser XSS auditor is deprecated in modern browsers and can itself introduce vulnerabilities. Rely on Content-Security-Policy instead.

---

## Secure Password Handling

### Argon2 (Preferred over bcrypt)

```python
from passlib.context import CryptContext

pwd_context = CryptContext(
    schemes=["argon2", "bcrypt"],
    default="argon2",
    deprecated=["bcrypt"],
    argon2__memory_cost=65536,   # 64 MB
    argon2__time_cost=3,
    argon2__parallelism=4,
)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def needs_rehash(hashed: str) -> bool:
    """Check if hash uses a deprecated scheme and should be re-hashed."""
    return pwd_context.needs_update(hashed)
```

Automatically rehash on login when the hash algorithm or parameters change:

```python
async def authenticate_and_rehash(
    db: AsyncSession, email: str, password: str
) -> User | None:
    user = await get_user_by_email(db, email)
    if not user or not verify_password(password, user.hashed_password):
        return None

    if needs_rehash(user.hashed_password):
        user.hashed_password = hash_password(password)
        await db.commit()

    return user
```

### Password Policy Validation

```python
import re
from pydantic import field_validator

class PasswordMixin(BaseModel):
    password: str = Field(..., min_length=10, max_length=128)

    @field_validator("password")
    @classmethod
    def validate_password(cls, v: str) -> str:
        errors = []
        if not re.search(r"[A-Z]", v):
            errors.append("at least one uppercase letter")
        if not re.search(r"[a-z]", v):
            errors.append("at least one lowercase letter")
        if not re.search(r"\d", v):
            errors.append("at least one digit")
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", v):
            errors.append("at least one special character")
        if errors:
            raise ValueError(f"Password must contain {', '.join(errors)}")
        return v
```

---

## Security Testing Checklist

### Automated Security Tests

```python
import pytest
from httpx import AsyncClient

class TestSecurityHeaders:
    """Verify security headers are present on all responses."""

    @pytest.mark.anyio
    async def test_security_headers_present(self, client: AsyncClient):
        response = await client.get("/api/v1/health")

        assert response.headers["X-Content-Type-Options"] == "nosniff"
        assert response.headers["X-Frame-Options"] == "DENY"
        assert "max-age=" in response.headers["Strict-Transport-Security"]
        assert "Content-Security-Policy" in response.headers

    @pytest.mark.anyio
    async def test_no_server_header(self, client: AsyncClient):
        response = await client.get("/api/v1/health")
        assert "server" not in response.headers

class TestAuthSecurity:
    """Verify authentication security behavior."""

    @pytest.mark.anyio
    async def test_expired_token_rejected(self, client: AsyncClient):
        expired_token = create_access_token(
            data={"sub": "1"}, expires_delta=timedelta(seconds=-1)
        )
        response = await client.get(
            "/api/v1/me",
            headers={"Authorization": f"Bearer {expired_token}"},
        )
        assert response.status_code == 401

    @pytest.mark.anyio
    async def test_malformed_token_rejected(self, client: AsyncClient):
        response = await client.get(
            "/api/v1/me",
            headers={"Authorization": "Bearer not.a.valid.token"},
        )
        assert response.status_code == 401

    @pytest.mark.anyio
    async def test_no_auth_header_rejected(self, client: AsyncClient):
        response = await client.get("/api/v1/me")
        assert response.status_code in (401, 403)

class TestIDOR:
    """Verify users cannot access other users' resources."""

    @pytest.mark.anyio
    async def test_cannot_access_other_users_order(
        self, client: AsyncClient, user_a_token: str, user_b_order_id: int
    ):
        response = await client.get(
            f"/api/v1/orders/{user_b_order_id}",
            headers={"Authorization": f"Bearer {user_a_token}"},
        )
        assert response.status_code == 404  # Not 403

class TestInputValidation:
    """Verify input validation and injection prevention."""

    @pytest.mark.anyio
    async def test_sql_injection_in_query_param(self, client: AsyncClient):
        response = await client.get(
            "/api/v1/users/",
            params={"search": "'; DROP TABLE users; --"},
        )
        assert response.status_code in (200, 422)

    @pytest.mark.anyio
    async def test_xss_in_input_sanitized(self, client: AsyncClient, auth_headers):
        response = await client.post(
            "/api/v1/comments/",
            json={"body": '<script>alert("xss")</script>Hello'},
            headers=auth_headers,
        )
        if response.status_code == 201:
            assert "<script>" not in response.json()["body"]

    @pytest.mark.anyio
    async def test_oversized_body_rejected(self, client: AsyncClient):
        large_payload = {"data": "x" * (10 * 1024 * 1024)}
        response = await client.post("/api/v1/data/", json=large_payload)
        assert response.status_code == 413
```

---

## Production Deployment Hardening

### Uvicorn Configuration

```bash
uvicorn app.main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4 \
    --timeout-keep-alive 5 \
    --limit-concurrency 100 \
    --limit-max-requests 10000 \
    --access-log \
    --proxy-headers \
    --forwarded-allow-ips="10.0.0.0/8"
```

### Reverse Proxy Headers

When behind a reverse proxy (nginx, Traefik), configure trusted proxies to prevent IP spoofing:

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware

# Only accept requests for known hostnames
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.example.com", "*.example.com"],
)

# Trust proxy headers only from known proxy IPs
# Configure in uvicorn: --forwarded-allow-ips="10.0.0.0/8"
```

### Disable Debug in Production

```python
from app.config import get_settings

settings = get_settings()

app = FastAPI(
    debug=False,                              # Never True in production
    docs_url="/docs" if settings.debug else None,     # Disable Swagger in production
    redoc_url="/redoc" if settings.debug else None,   # Disable ReDoc in production
    openapi_url="/openapi.json" if settings.debug else None,
)
```

### HTTPS Enforcement

Redirect all HTTP traffic to HTTPS via middleware:

```python
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

if settings.environment == "production":
    app.add_middleware(HTTPSRedirectMiddleware)
```
