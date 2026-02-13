# API Security for FastAPI

This reference covers authentication and authorization patterns for FastAPI APIs, including OAuth2 with scopes, JWT implementation, API key authentication, CORS configuration, rate limiting, input validation hardening, and security headers.

---

## OAuth2 with Scopes

### OAuth2 Password Bearer with Scopes

```python
from fastapi import Depends, Security, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, SecurityScopes
from pydantic import BaseModel

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/auth/token",
    scopes={
        "users:read": "Read user information",
        "users:write": "Create and modify users",
        "products:read": "Read product catalog",
        "products:write": "Manage products",
        "orders:read": "View orders",
        "orders:write": "Create and manage orders",
        "admin": "Full administrative access",
    },
)

class TokenData(BaseModel):
    sub: str
    scopes: list[str] = []

async def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"

    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": authenticate_value},
    )

    try:
        payload = decode_jwt(token)
        user_id = payload.get("sub")
        if user_id is None:
            raise credentials_exception
        token_scopes = payload.get("scopes", [])
        token_data = TokenData(sub=user_id, scopes=token_scopes)
    except JWTError:
        raise credentials_exception

    user = await db.get(User, int(token_data.sub))
    if user is None:
        raise credentials_exception

    # Check required scopes
    for scope in security_scopes.scopes:
        if scope not in token_data.scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Not enough permissions. Required: {scope}",
                headers={"WWW-Authenticate": authenticate_value},
            )

    return user
```

### Using Scopes on Endpoints

```python
@router.get("/users/", response_model=list[UserResponse])
async def list_users(
    user: User = Security(get_current_user, scopes=["users:read"]),
    db: AsyncSession = Depends(get_db),
):
    return await user_service.get_all(db)

@router.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(
    user_in: UserCreate,
    user: User = Security(get_current_user, scopes=["users:write"]),
    db: AsyncSession = Depends(get_db),
):
    return await user_service.create(db, user_in)

@router.delete("/users/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    user: User = Security(get_current_user, scopes=["admin"]),
    db: AsyncSession = Depends(get_db),
):
    await user_service.delete(db, user_id)
```

### Token Endpoint

```python
from fastapi import APIRouter
from fastapi.security import OAuth2PasswordRequestForm
from datetime import timedelta

auth_router = APIRouter(prefix="/auth", tags=["Authentication"])

@auth_router.post("/token")
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Determine scopes based on user role
    scopes = get_scopes_for_role(user.role)

    # Only grant requested scopes that the user actually has
    granted_scopes = [s for s in form_data.scopes if s in scopes]

    access_token = create_access_token(
        data={"sub": str(user.id), "scopes": granted_scopes},
        expires_delta=timedelta(minutes=settings.access_token_expire_minutes),
    )
    refresh_token = create_refresh_token(data={"sub": str(user.id)})

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": settings.access_token_expire_minutes * 60,
        "scope": " ".join(granted_scopes),
    }
```

---

## JWT Implementation

### Token Creation and Validation

```python
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError

ALGORITHM = "HS256"

def create_access_token(
    data: dict,
    expires_delta: timedelta = timedelta(minutes=30),
) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + expires_delta
    to_encode.update({
        "exp": expire,
        "iat": datetime.now(timezone.utc),
        "type": "access",
    })
    return jwt.encode(to_encode, settings.secret_key, algorithm=ALGORITHM)

def create_refresh_token(
    data: dict,
    expires_delta: timedelta = timedelta(days=7),
) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + expires_delta
    to_encode.update({
        "exp": expire,
        "iat": datetime.now(timezone.utc),
        "type": "refresh",
    })
    return jwt.encode(to_encode, settings.secret_key, algorithm=ALGORITHM)

def decode_jwt(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            settings.secret_key,
            algorithms=[ALGORITHM],
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=401,
            detail="Token has expired",
            headers={"WWW-Authenticate": "Bearer"},
        )
    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Invalid token",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

### Token Refresh Endpoint

```python
@auth_router.post("/refresh")
async def refresh_access_token(
    refresh_token: str,
    db: AsyncSession = Depends(get_db),
):
    payload = decode_jwt(refresh_token)

    if payload.get("type") != "refresh":
        raise HTTPException(400, "Invalid token type")

    user = await db.get(User, int(payload["sub"]))
    if not user or not user.is_active:
        raise HTTPException(401, "User not found or inactive")

    scopes = get_scopes_for_role(user.role)
    new_access_token = create_access_token(
        data={"sub": str(user.id), "scopes": scopes}
    )

    return {
        "access_token": new_access_token,
        "token_type": "bearer",
        "expires_in": settings.access_token_expire_minutes * 60,
    }
```

### Token Revocation with Blocklist

```python
class TokenBlocklist:
    """Redis-backed token blocklist for logout and revocation."""

    def __init__(self, redis):
        self.redis = redis

    async def revoke(self, token: str, expires_in: int):
        """Add token to blocklist until its natural expiration."""
        jti = decode_jwt(token).get("jti", token[:32])
        await self.redis.setex(f"blocked:{jti}", expires_in, "1")

    async def is_revoked(self, token: str) -> bool:
        jti = decode_jwt(token).get("jti", token[:32])
        return await self.redis.exists(f"blocked:{jti}")

# Integrate into auth dependency
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    blocklist: TokenBlocklist = Depends(get_blocklist),
    db: AsyncSession = Depends(get_db),
) -> User:
    if await blocklist.is_revoked(token):
        raise HTTPException(401, "Token has been revoked")
    # ... rest of validation
```

---

## API Key Authentication

### Header-Based API Keys

```python
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=True)

async def get_api_key_user(
    api_key: str = Depends(api_key_header),
    db: AsyncSession = Depends(get_db),
) -> APIKeyRecord:
    # Hash the key for lookup (store hashed keys in DB)
    key_hash = hashlib.sha256(api_key.encode()).hexdigest()

    result = await db.execute(
        select(APIKeyRecord)
        .where(APIKeyRecord.key_hash == key_hash)
        .where(APIKeyRecord.is_active == True)
        .where(
            or_(
                APIKeyRecord.expires_at.is_(None),
                APIKeyRecord.expires_at > func.now(),
            )
        )
    )
    record = result.scalar_one_or_none()

    if not record:
        raise HTTPException(
            status_code=401,
            detail="Invalid or expired API key",
        )

    # Update last_used timestamp
    record.last_used_at = func.now()
    record.request_count += 1
    await db.commit()

    return record
```

### Combined Authentication (Bearer OR API Key)

```python
from fastapi.security import HTTPBearer, APIKeyHeader

http_bearer = HTTPBearer(auto_error=False)
api_key_header_scheme = APIKeyHeader(name="X-API-Key", auto_error=False)

async def get_authenticated_entity(
    bearer: HTTPAuthorizationCredentials | None = Depends(http_bearer),
    api_key: str | None = Depends(api_key_header_scheme),
    db: AsyncSession = Depends(get_db),
) -> User | APIKeyRecord:
    """Accept either Bearer token or API key authentication."""
    if bearer:
        return await validate_bearer_token(bearer.credentials, db)
    elif api_key:
        return await validate_api_key(api_key, db)
    else:
        raise HTTPException(
            status_code=401,
            detail="Provide either a Bearer token or X-API-Key header",
        )
```

---

## CORS Configuration

### Production CORS Setup

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,   # Explicit list, never ["*"] in production
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=[
        "Authorization",
        "Content-Type",
        "X-API-Key",
        "X-Request-ID",
    ],
    expose_headers=[
        "X-Request-ID",
        "X-Process-Time",
        "X-Total-Count",
    ],
    max_age=600,  # Cache preflight for 10 minutes
)
```

### Environment-Specific CORS

```python
if settings.environment == "development":
    origins = ["http://localhost:3000", "http://localhost:5173"]
elif settings.environment == "staging":
    origins = ["https://staging.example.com"]
elif settings.environment == "production":
    origins = ["https://app.example.com", "https://admin.example.com"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

---

## Rate Limiting

### Using slowapi

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["100/minute"],
    storage_uri="redis://localhost:6379/1",
)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.get("/products/")
@limiter.limit("60/minute")
async def list_products(request: Request):
    ...

@router.post("/auth/token")
@limiter.limit("5/minute")
async def login(request: Request):
    ...
```

### Custom Rate Limiting by API Key

```python
from slowapi.util import get_remote_address

def get_rate_limit_key(request: Request) -> str:
    """Use API key if present, otherwise fall back to IP address."""
    api_key = request.headers.get("X-API-Key")
    if api_key:
        return f"api_key:{hashlib.md5(api_key.encode()).hexdigest()}"
    return f"ip:{get_remote_address(request)}"

limiter = Limiter(key_func=get_rate_limit_key)
```

### Rate Limit Headers

```python
from starlette.middleware.base import BaseHTTPMiddleware

class RateLimitHeaderMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        # Add standard rate limit headers
        response.headers["X-RateLimit-Limit"] = "100"
        response.headers["X-RateLimit-Remaining"] = str(
            max(0, 100 - get_current_usage(request))
        )
        response.headers["X-RateLimit-Reset"] = str(get_window_reset_time())
        return response
```

---

## Input Validation and Sanitization

### Strict Pydantic Validation

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict
import re
import bleach

class CommentCreate(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    body: str = Field(..., min_length=1, max_length=10000)
    parent_id: int | None = Field(None, gt=0)

    @field_validator("body")
    @classmethod
    def sanitize_html(cls, v: str) -> str:
        """Strip dangerous HTML, allow only safe tags."""
        return bleach.clean(
            v,
            tags=["p", "br", "b", "i", "em", "strong", "a", "ul", "ol", "li"],
            attributes={"a": ["href", "title"]},
            protocols=["https"],
            strip=True,
        )

class UserCreate(BaseModel):
    email: str = Field(..., max_length=255)
    username: str = Field(..., min_length=3, max_length=30)

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_-]+$", v):
            raise ValueError("Username may only contain letters, digits, hyphens, and underscores")
        return v

    @field_validator("email")
    @classmethod
    def validate_email_format(cls, v: str) -> str:
        # Additional check beyond Pydantic's EmailStr
        if ".." in v or v.startswith(".") or v.endswith("."):
            raise ValueError("Invalid email format")
        return v.lower()
```

### Path Traversal Prevention

```python
import os

@router.get("/files/{file_path:path}")
async def serve_file(file_path: str):
    # Prevent path traversal attacks
    safe_base = "/app/uploads"
    requested_path = os.path.realpath(os.path.join(safe_base, file_path))

    if not requested_path.startswith(safe_base):
        raise HTTPException(400, "Invalid file path")

    if not os.path.isfile(requested_path):
        raise HTTPException(404, "File not found")

    return FileResponse(requested_path)
```

### SQL Injection Prevention

Always use parameterized queries. Never construct SQL strings from user input:

```python
# BAD - vulnerable to SQL injection
stmt = text(f"SELECT * FROM users WHERE email = '{email}'")

# GOOD - parameterized query
stmt = text("SELECT * FROM users WHERE email = :email")
result = await session.execute(stmt, {"email": email})

# BEST - use ORM which parameterizes automatically
stmt = select(User).where(User.email == email)
result = await session.execute(stmt)
```

---

## Security Headers

### Security Header Middleware

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains"
        )
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = (
            "camera=(), microphone=(), geolocation=()"
        )
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self'; "
            "frame-ancestors 'none'"
        )

        # Remove server identification headers
        response.headers.pop("server", None)

        return response

app.add_middleware(SecurityHeadersMiddleware)
```

### Trusted Host Middleware

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=settings.allowed_hosts,  # ["api.example.com", "*.example.com"]
)
```

---

## Password Hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__rounds=12,
)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

---

## Request ID Tracing

```python
from uuid import uuid4

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid4()))
        # Make available to handlers
        request.state.request_id = request_id

        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

This enables tracing requests across services in a microservices architecture. Log the request ID in every log entry for correlation.
