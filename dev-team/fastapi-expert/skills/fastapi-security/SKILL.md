---
name: FastAPI Security
description: This skill should be used when the user asks about "FastAPI security", "CSRF protection", "brute force prevention", "file upload security", "SSRF", "IDOR", "audit logging", "secure cookies", "MFA FastAPI", "account lockout", or "security hardening". It covers authentication hardening, CSRF protection, secure file upload handling, SSRF and IDOR prevention, security audit logging, request size limiting, secure cookie and session management, and multi-factor authentication patterns. Use this skill for any task involving security hardening, vulnerability prevention, or security best practices in FastAPI applications.
---

### CSRF Protection

To protect session-based or cookie-based authentication against cross-site request forgery, implement CSRF token validation:

```python
import secrets
from fastapi import Request, HTTPException, Depends
from fastapi.responses import JSONResponse

CSRF_TOKEN_LENGTH = 64
CSRF_HEADER_NAME = "X-CSRF-Token"
CSRF_COOKIE_NAME = "csrf_token"

def generate_csrf_token() -> str:
    return secrets.token_urlsafe(CSRF_TOKEN_LENGTH)

async def csrf_protect(request: Request):
    """Dependency that validates CSRF token on state-changing requests."""
    if request.method in ("GET", "HEAD", "OPTIONS"):
        return

    cookie_token = request.cookies.get(CSRF_COOKIE_NAME)
    header_token = request.headers.get(CSRF_HEADER_NAME)

    if not cookie_token or not header_token:
        raise HTTPException(status_code=403, detail="Missing CSRF token")

    if not secrets.compare_digest(cookie_token, header_token):
        raise HTTPException(status_code=403, detail="CSRF token mismatch")

@router.get("/csrf-token")
async def get_csrf_token():
    """Issue a CSRF token to the client."""
    token = generate_csrf_token()
    response = JSONResponse({"csrf_token": token})
    response.set_cookie(
        key=CSRF_COOKIE_NAME,
        value=token,
        httponly=True,
        secure=True,
        samesite="strict",
        max_age=3600,
    )
    return response
```

Apply CSRF protection at the router or app level:

```python
router = APIRouter(dependencies=[Depends(csrf_protect)])
```

When using cookie-based sessions, always set `SameSite=Strict` or `SameSite=Lax` on all authentication cookies. This provides a strong first line of defense against CSRF even before token validation.

### Brute Force and Account Lockout Protection

To prevent credential stuffing and brute force attacks, implement per-account rate limiting with progressive lockout:

```python
from datetime import datetime, timedelta, timezone
from fastapi import HTTPException

class LoginRateLimiter:
    """Tracks failed login attempts and enforces lockout policy."""

    def __init__(self, redis):
        self.redis = redis
        self.max_attempts = 5
        self.lockout_duration = timedelta(minutes=15)
        self.attempt_window = timedelta(minutes=30)

    async def check_and_record(self, identifier: str, success: bool):
        key = f"login_attempts:{identifier}"
        lockout_key = f"login_lockout:{identifier}"

        # Check if account is locked
        if await self.redis.exists(lockout_key):
            ttl = await self.redis.ttl(lockout_key)
            raise HTTPException(
                status_code=429,
                detail=f"Account locked. Try again in {ttl} seconds.",
                headers={"Retry-After": str(ttl)},
            )

        if success:
            # Clear attempts on successful login
            await self.redis.delete(key)
            return

        # Increment failed attempts
        attempts = await self.redis.incr(key)
        if attempts == 1:
            await self.redis.expire(key, int(self.attempt_window.total_seconds()))

        if attempts >= self.max_attempts:
            # Lock the account
            await self.redis.setex(
                lockout_key,
                int(self.lockout_duration.total_seconds()),
                "1",
            )
            await self.redis.delete(key)
            raise HTTPException(
                status_code=429,
                detail="Too many failed attempts. Account temporarily locked.",
            )

@auth_router.post("/token")
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
    limiter: LoginRateLimiter = Depends(get_login_limiter),
):
    # Check lockout before authentication attempt
    await limiter.check_and_record(form_data.username, success=False)

    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Clear failed attempts on success
    await limiter.check_and_record(form_data.username, success=True)

    access_token = create_access_token(data={"sub": str(user.id)})
    return {"access_token": access_token, "token_type": "bearer"}
```

To prevent username enumeration, always return the same error message and response time regardless of whether the username exists:

```python
import asyncio
import random

async def authenticate_user(db: AsyncSession, username: str, password: str) -> User | None:
    user = await db.execute(select(User).where(User.email == username))
    user = user.scalar_one_or_none()

    if user is None:
        # Perform a dummy hash check to maintain constant time
        pwd_context.hash("dummy_password")
        return None

    if not pwd_context.verify(password, user.hashed_password):
        return None

    return user
```

### Secure File Upload Handling

To validate file uploads beyond MIME type checking, verify actual file content using magic bytes:

```python
import magic
from fastapi import UploadFile, HTTPException

ALLOWED_TYPES = {
    "image/jpeg": [b"\xff\xd8\xff"],
    "image/png": [b"\x89PNG\r\n\x1a\n"],
    "image/webp": [b"RIFF"],
    "application/pdf": [b"%PDF"],
}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB
MAX_FILENAME_LENGTH = 255

async def validate_upload(
    file: UploadFile,
    allowed_types: dict = ALLOWED_TYPES,
    max_size: int = MAX_FILE_SIZE,
) -> bytes:
    """Validate file upload content, type, and size."""
    # Read content
    content = await file.read()

    # Check size
    if len(content) > max_size:
        raise HTTPException(
            status_code=413,
            detail=f"File exceeds {max_size // (1024 * 1024)} MB limit",
        )

    # Validate actual content type using magic bytes
    detected_type = magic.from_buffer(content, mime=True)
    if detected_type not in allowed_types:
        raise HTTPException(
            status_code=422,
            detail=f"File type '{detected_type}' not allowed",
        )

    # Verify magic bytes match
    header = content[:16]
    signatures = allowed_types[detected_type]
    if not any(header.startswith(sig) for sig in signatures):
        raise HTTPException(
            status_code=422,
            detail="File content does not match its type",
        )

    return content

def sanitize_filename(filename: str) -> str:
    """Strip path components and dangerous characters from filenames."""
    import os
    import re
    import uuid

    # Remove path components
    name = os.path.basename(filename)

    # Remove null bytes and control characters
    name = re.sub(r"[\x00-\x1f\x7f]", "", name)

    # Keep only safe characters
    name = re.sub(r"[^\w.\-]", "_", name)

    # Truncate
    if len(name) > MAX_FILENAME_LENGTH:
        ext = os.path.splitext(name)[1][:10]
        name = name[:MAX_FILENAME_LENGTH - len(ext)] + ext

    # Prefix with UUID to prevent overwrites
    return f"{uuid.uuid4().hex}_{name}"
```

To prevent path traversal in file storage:

```python
import os

def safe_join(base_dir: str, filename: str) -> str:
    """Join paths safely, preventing directory traversal."""
    full_path = os.path.realpath(os.path.join(base_dir, filename))
    if not full_path.startswith(os.path.realpath(base_dir)):
        raise HTTPException(status_code=400, detail="Invalid file path")
    return full_path
```

### IDOR Prevention (Insecure Direct Object References)

To prevent users from accessing resources they do not own, always verify ownership in service-layer queries:

```python
class OrderService:
    def __init__(
        self,
        repo: OrderRepository = Depends(),
        current_user: User = Depends(get_current_user),
    ):
        self.repo = repo
        self.current_user = current_user

    async def get_order(self, order_id: int) -> Order:
        order = await self.repo.get_by_id(order_id)
        if not order:
            raise NotFoundError("Order", order_id)

        # IDOR check: verify the current user owns this resource
        if order.user_id != self.current_user.id and not self.current_user.is_admin:
            # Return 404, not 403, to avoid revealing resource existence
            raise NotFoundError("Order", order_id)

        return order

    async def list_orders(self, page: int, page_size: int) -> PaginatedResponse:
        """Always scope queries to the current user."""
        return await self.repo.get_by_user(
            user_id=self.current_user.id,
            skip=(page - 1) * page_size,
            limit=page_size,
        )
```

For non-sequential resource identifiers, use UUIDs instead of auto-incrementing integers to make identifiers non-guessable:

```python
import uuid
from sqlalchemy import Uuid

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[uuid.UUID] = mapped_column(
        Uuid, primary_key=True, default=uuid.uuid4
    )
```

### SSRF Prevention

To prevent server-side request forgery when the application makes outbound HTTP requests based on user input:

```python
import ipaddress
import socket
from urllib.parse import urlparse
from fastapi import HTTPException

BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),  # Link-local
    ipaddress.ip_network("::1/128"),          # IPv6 loopback
    ipaddress.ip_network("fc00::/7"),         # IPv6 private
    ipaddress.ip_network("fe80::/10"),        # IPv6 link-local
]

ALLOWED_SCHEMES = {"http", "https"}

def validate_url(url: str) -> str:
    """Validate that a user-supplied URL is safe to fetch."""
    parsed = urlparse(url)

    # Validate scheme
    if parsed.scheme not in ALLOWED_SCHEMES:
        raise HTTPException(400, f"URL scheme '{parsed.scheme}' not allowed")

    # Validate hostname exists
    if not parsed.hostname:
        raise HTTPException(400, "Invalid URL: no hostname")

    # Resolve hostname and check against blocked networks
    try:
        resolved_ips = socket.getaddrinfo(parsed.hostname, parsed.port or 443)
    except socket.gaierror:
        raise HTTPException(400, "Cannot resolve hostname")

    for _, _, _, _, sockaddr in resolved_ips:
        ip = ipaddress.ip_address(sockaddr[0])
        for network in BLOCKED_NETWORKS:
            if ip in network:
                raise HTTPException(
                    400, "URL resolves to a blocked network address"
                )

    return url

@router.post("/webhooks/test")
async def test_webhook(
    url: str,
    client: httpx.AsyncClient = Depends(get_http_client),
):
    safe_url = validate_url(url)
    response = await client.post(
        safe_url,
        json={"event": "test"},
        timeout=10.0,
        follow_redirects=False,  # Prevent redirect-based SSRF
    )
    return {"status_code": response.status_code}
```

### Security Audit Logging

To log security-relevant events for monitoring and compliance:

```python
import structlog
from enum import Enum

class SecurityEvent(str, Enum):
    LOGIN_SUCCESS = "login_success"
    LOGIN_FAILURE = "login_failure"
    ACCOUNT_LOCKED = "account_locked"
    PASSWORD_CHANGED = "password_changed"
    PERMISSION_DENIED = "permission_denied"
    TOKEN_REVOKED = "token_revoked"
    SUSPICIOUS_ACTIVITY = "suspicious_activity"
    DATA_EXPORT = "data_export"
    ADMIN_ACTION = "admin_action"

security_logger = structlog.get_logger("security")

async def log_security_event(
    event: SecurityEvent,
    request: Request,
    user_id: int | None = None,
    details: dict | None = None,
):
    """Log a structured security event."""
    security_logger.info(
        event.value,
        user_id=user_id,
        ip_address=request.client.host if request.client else None,
        user_agent=request.headers.get("User-Agent"),
        path=str(request.url),
        method=request.method,
        **(details or {}),
    )
```

Integrate into authentication flows:

```python
@auth_router.post("/token")
async def login(
    request: Request,
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        await log_security_event(
            SecurityEvent.LOGIN_FAILURE,
            request,
            details={"username": form_data.username},
        )
        raise HTTPException(status_code=401, detail="Invalid credentials")

    await log_security_event(
        SecurityEvent.LOGIN_SUCCESS,
        request,
        user_id=user.id,
    )
    return create_token_response(user)
```

Create a middleware for logging permission denials automatically:

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityAuditMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        if response.status_code == 403:
            await log_security_event(
                SecurityEvent.PERMISSION_DENIED,
                request,
                details={"status_code": 403},
            )
        elif response.status_code == 401:
            await log_security_event(
                SecurityEvent.SUSPICIOUS_ACTIVITY,
                request,
                details={"reason": "unauthenticated_access_attempt"},
            )

        return response
```

### Request Size Limiting and DoS Mitigation

To protect against oversized request bodies and slow-client attacks:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

class RequestSizeLimitMiddleware(BaseHTTPMiddleware):
    """Reject requests that exceed a configurable body size limit."""

    def __init__(self, app, max_body_size: int = 1 * 1024 * 1024):
        super().__init__(app)
        self.max_body_size = max_body_size  # Default 1 MB

    async def dispatch(self, request: Request, call_next):
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > self.max_body_size:
            return JSONResponse(
                status_code=413,
                content={"error": "PAYLOAD_TOO_LARGE", "detail": "Request body too large"},
            )
        return await call_next(request)

app.add_middleware(RequestSizeLimitMiddleware, max_body_size=5 * 1024 * 1024)
```

Configure Uvicorn-level timeouts to prevent slow-client attacks:

```python
# uvicorn main:app --timeout-keep-alive 5 --limit-concurrency 100
# Or programmatically:
import uvicorn

uvicorn.run(
    "app.main:app",
    host="0.0.0.0",
    port=8000,
    timeout_keep_alive=5,
    limit_concurrency=100,
    limit_max_requests=10000,  # Restart worker after N requests (leak prevention)
)
```

### Secure Cookie and Session Management

To implement secure cookie-based session authentication:

```python
from fastapi import Response, Request
from itsdangerous import URLSafeTimedSerializer, BadSignature, SignatureExpired

class SessionManager:
    def __init__(self, secret_key: str, max_age: int = 3600):
        self.serializer = URLSafeTimedSerializer(secret_key)
        self.max_age = max_age
        self.cookie_name = "session_id"

    def create_session(self, response: Response, data: dict):
        """Create a signed session cookie."""
        token = self.serializer.dumps(data)
        response.set_cookie(
            key=self.cookie_name,
            value=token,
            httponly=True,       # Not accessible via JavaScript
            secure=True,         # Only sent over HTTPS
            samesite="lax",      # CSRF protection
            max_age=self.max_age,
            path="/",
            domain=None,         # Current domain only
        )

    def get_session(self, request: Request) -> dict | None:
        """Retrieve and validate a session from the cookie."""
        token = request.cookies.get(self.cookie_name)
        if not token:
            return None
        try:
            return self.serializer.loads(token, max_age=self.max_age)
        except (BadSignature, SignatureExpired):
            return None

    def destroy_session(self, response: Response):
        """Clear the session cookie."""
        response.delete_cookie(
            key=self.cookie_name,
            httponly=True,
            secure=True,
            samesite="lax",
            path="/",
        )
```

Key cookie security flags:
- `httponly=True`: Prevents JavaScript access (mitigates XSS token theft).
- `secure=True`: Cookie only sent over HTTPS.
- `samesite="lax"` or `"strict"`: Blocks cross-site cookie sending (CSRF mitigation).
- `max_age`: Set a reasonable expiration; do not use permanent sessions.
- Never store sensitive data directly in cookies. Store a session ID and keep data server-side.

## References

- [references/security-hardening.md](references/security-hardening.md) - Dependency scanning, secrets management, data encryption, content-type enforcement, security testing checklist, and production deployment hardening
- [references/auth-advanced.md](references/auth-advanced.md) - OAuth2 Authorization Code with PKCE, social authentication, multi-factor authentication, token rotation, and secure password reset flows
