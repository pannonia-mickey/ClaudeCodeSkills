# Advanced Authentication Patterns for FastAPI

This reference covers advanced authentication flows for FastAPI applications, including OAuth2 Authorization Code with PKCE, social authentication, multi-factor authentication (MFA/2FA), token rotation, and secure password reset flows.

---

## OAuth2 Authorization Code with PKCE

The Authorization Code flow with PKCE is the recommended approach for SPAs and mobile apps. PKCE prevents authorization code interception attacks.

### PKCE Utilities

```python
import hashlib
import base64
import secrets

def generate_code_verifier() -> str:
    """Generate a cryptographically random code verifier (43-128 characters)."""
    return secrets.token_urlsafe(64)

def generate_code_challenge(verifier: str) -> str:
    """Derive the code challenge from the verifier using S256."""
    digest = hashlib.sha256(verifier.encode("ascii")).digest()
    return base64.urlsafe_b64encode(digest).rstrip(b"=").decode("ascii")

def verify_code_challenge(verifier: str, challenge: str) -> bool:
    """Verify that the code verifier matches the stored code challenge."""
    return secrets.compare_digest(generate_code_challenge(verifier), challenge)
```

### Authorization Endpoint

```python
from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import RedirectResponse

auth_router = APIRouter(prefix="/oauth", tags=["OAuth2"])

class AuthorizationRequest(BaseModel):
    response_type: str = Field("code")
    client_id: str
    redirect_uri: str
    scope: str = ""
    state: str
    code_challenge: str
    code_challenge_method: str = Field("S256")

@auth_router.get("/authorize")
async def authorize(
    params: AuthorizationRequest = Depends(),
    db: AsyncSession = Depends(get_db),
):
    # Validate client_id and redirect_uri
    client = await db.execute(
        select(OAuthClient).where(OAuthClient.client_id == params.client_id)
    )
    client = client.scalar_one_or_none()
    if not client or params.redirect_uri not in client.redirect_uris:
        raise HTTPException(400, "Invalid client_id or redirect_uri")

    if params.code_challenge_method != "S256":
        raise HTTPException(400, "Only S256 code_challenge_method is supported")

    # Store authorization request in session/cache for the consent screen
    # After user consents, generate an authorization code:
    auth_code = secrets.token_urlsafe(32)

    await db.execute(
        insert(AuthorizationCode).values(
            code=auth_code,
            client_id=params.client_id,
            redirect_uri=params.redirect_uri,
            scope=params.scope,
            code_challenge=params.code_challenge,
            user_id=current_user.id,
            expires_at=datetime.now(timezone.utc) + timedelta(minutes=5),
        )
    )
    await db.commit()

    redirect_url = f"{params.redirect_uri}?code={auth_code}&state={params.state}"
    return RedirectResponse(url=redirect_url, status_code=302)
```

### Token Exchange Endpoint

```python
class TokenExchangeRequest(BaseModel):
    grant_type: str = Field("authorization_code")
    code: str
    redirect_uri: str
    client_id: str
    code_verifier: str

@auth_router.post("/token")
async def exchange_token(
    request: TokenExchangeRequest,
    db: AsyncSession = Depends(get_db),
):
    if request.grant_type != "authorization_code":
        raise HTTPException(400, "Unsupported grant_type")

    # Look up the authorization code
    result = await db.execute(
        select(AuthorizationCode).where(
            AuthorizationCode.code == request.code,
            AuthorizationCode.client_id == request.client_id,
            AuthorizationCode.redirect_uri == request.redirect_uri,
            AuthorizationCode.used == False,
            AuthorizationCode.expires_at > func.now(),
        )
    )
    auth_code = result.scalar_one_or_none()

    if not auth_code:
        raise HTTPException(400, "Invalid or expired authorization code")

    # Verify PKCE
    if not verify_code_challenge(request.code_verifier, auth_code.code_challenge):
        raise HTTPException(400, "Invalid code_verifier")

    # Mark code as used (codes are single-use)
    auth_code.used = True
    await db.commit()

    # Issue tokens
    access_token = create_access_token(
        data={"sub": str(auth_code.user_id), "scope": auth_code.scope}
    )
    refresh_token = create_refresh_token(
        data={"sub": str(auth_code.user_id)}
    )

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": settings.access_token_expire_minutes * 60,
        "scope": auth_code.scope,
    }
```

---

## Social Authentication (OAuth2 Providers)

### Provider Configuration

```python
from pydantic_settings import BaseSettings

class OAuthProviders(BaseSettings):
    google_client_id: str = ""
    google_client_secret: SecretStr = SecretStr("")
    google_redirect_uri: str = "http://localhost:8000/auth/google/callback"

    github_client_id: str = ""
    github_client_secret: SecretStr = SecretStr("")
    github_redirect_uri: str = "http://localhost:8000/auth/github/callback"
```

### Generic OAuth2 Provider

```python
import httpx
from dataclasses import dataclass

@dataclass
class OAuthProvider:
    name: str
    client_id: str
    client_secret: str
    authorize_url: str
    token_url: str
    userinfo_url: str
    scopes: list[str]

    def get_authorization_url(self, state: str, redirect_uri: str) -> str:
        params = {
            "client_id": self.client_id,
            "redirect_uri": redirect_uri,
            "response_type": "code",
            "scope": " ".join(self.scopes),
            "state": state,
            "access_type": "offline",
        }
        query = "&".join(f"{k}={v}" for k, v in params.items())
        return f"{self.authorize_url}?{query}"

    async def exchange_code(self, code: str, redirect_uri: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_url,
                data={
                    "grant_type": "authorization_code",
                    "code": code,
                    "redirect_uri": redirect_uri,
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                },
                headers={"Accept": "application/json"},
            )
            response.raise_for_status()
            return response.json()

    async def get_user_info(self, access_token: str) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                self.userinfo_url,
                headers={"Authorization": f"Bearer {access_token}"},
            )
            response.raise_for_status()
            return response.json()

# Provider instances
google_provider = OAuthProvider(
    name="google",
    client_id=settings.google_client_id,
    client_secret=settings.google_client_secret.get_secret_value(),
    authorize_url="https://accounts.google.com/o/oauth2/v2/auth",
    token_url="https://oauth2.googleapis.com/token",
    userinfo_url="https://www.googleapis.com/oauth2/v2/userinfo",
    scopes=["openid", "email", "profile"],
)

github_provider = OAuthProvider(
    name="github",
    client_id=settings.github_client_id,
    client_secret=settings.github_client_secret.get_secret_value(),
    authorize_url="https://github.com/login/oauth/authorize",
    token_url="https://github.com/login/oauth/access_token",
    userinfo_url="https://api.github.com/user",
    scopes=["user:email"],
)
```

### Social Auth Endpoints

```python
@auth_router.get("/login/{provider}")
async def social_login(provider: str):
    oauth_provider = get_provider(provider)
    state = secrets.token_urlsafe(32)
    # Store state in Redis/session for validation
    await redis.setex(f"oauth_state:{state}", 600, provider)

    url = oauth_provider.get_authorization_url(
        state=state,
        redirect_uri=oauth_provider.redirect_uri,
    )
    return RedirectResponse(url=url)

@auth_router.get("/callback/{provider}")
async def social_callback(
    provider: str,
    code: str,
    state: str,
    db: AsyncSession = Depends(get_db),
):
    # Validate state to prevent CSRF
    stored_provider = await redis.get(f"oauth_state:{state}")
    if not stored_provider or stored_provider.decode() != provider:
        raise HTTPException(400, "Invalid state parameter")
    await redis.delete(f"oauth_state:{state}")

    oauth_provider = get_provider(provider)

    # Exchange code for tokens
    tokens = await oauth_provider.exchange_code(
        code=code, redirect_uri=oauth_provider.redirect_uri
    )

    # Get user info from provider
    user_info = await oauth_provider.get_user_info(tokens["access_token"])

    # Find or create user
    user = await find_or_create_social_user(
        db, provider=provider, provider_user_id=str(user_info["id"]),
        email=user_info.get("email"), name=user_info.get("name"),
    )

    # Issue application tokens
    access_token = create_access_token(data={"sub": str(user.id)})
    refresh_token = create_refresh_token(data={"sub": str(user.id)})

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
    }
```

### Social Account Linking

```python
class SocialAccount(Base):
    __tablename__ = "social_accounts"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    provider: Mapped[str] = mapped_column(String(50))
    provider_user_id: Mapped[str] = mapped_column(String(255))
    access_token: Mapped[str | None] = mapped_column(String(500))
    refresh_token: Mapped[str | None] = mapped_column(String(500))

    __table_args__ = (
        UniqueConstraint("provider", "provider_user_id", name="uq_social_provider_user"),
    )

async def find_or_create_social_user(
    db: AsyncSession,
    provider: str,
    provider_user_id: str,
    email: str | None,
    name: str | None,
) -> User:
    # Check for existing social account link
    result = await db.execute(
        select(SocialAccount).where(
            SocialAccount.provider == provider,
            SocialAccount.provider_user_id == provider_user_id,
        )
    )
    social = result.scalar_one_or_none()

    if social:
        return await db.get(User, social.user_id)

    # Check if a user with this email already exists
    if email:
        result = await db.execute(select(User).where(User.email == email))
        user = result.scalar_one_or_none()
        if user:
            # Link the social account to existing user
            db.add(SocialAccount(
                user_id=user.id,
                provider=provider,
                provider_user_id=provider_user_id,
            ))
            await db.commit()
            return user

    # Create new user
    user = User(email=email, full_name=name or "", is_active=True)
    db.add(user)
    await db.flush()

    db.add(SocialAccount(
        user_id=user.id,
        provider=provider,
        provider_user_id=provider_user_id,
    ))
    await db.commit()
    return user
```

---

## Multi-Factor Authentication (MFA/2FA)

### TOTP (Time-Based One-Time Password)

```python
import pyotp
import qrcode
import io
import base64

class TOTPService:
    def __init__(self, issuer: str = "MyApp"):
        self.issuer = issuer

    def generate_secret(self) -> str:
        """Generate a new TOTP secret for a user."""
        return pyotp.random_base32()

    def get_provisioning_uri(self, secret: str, email: str) -> str:
        """Generate the otpauth:// URI for QR code scanning."""
        totp = pyotp.TOTP(secret)
        return totp.provisioning_uri(name=email, issuer_name=self.issuer)

    def generate_qr_code(self, uri: str) -> str:
        """Generate a base64-encoded QR code image."""
        qr = qrcode.make(uri)
        buffer = io.BytesIO()
        qr.save(buffer, format="PNG")
        return base64.b64encode(buffer.getvalue()).decode()

    def verify_token(self, secret: str, token: str) -> bool:
        """Verify a TOTP token with a +-1 window for clock skew."""
        totp = pyotp.TOTP(secret)
        return totp.verify(token, valid_window=1)

    def generate_backup_codes(self, count: int = 10) -> list[str]:
        """Generate single-use backup codes."""
        return [secrets.token_hex(4).upper() for _ in range(count)]
```

### MFA Setup Endpoints

```python
totp_service = TOTPService(issuer="MyApp")

@auth_router.post("/mfa/setup")
async def setup_mfa(
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    if user.mfa_enabled:
        raise HTTPException(400, "MFA is already enabled")

    secret = totp_service.generate_secret()
    uri = totp_service.get_provisioning_uri(secret, user.email)
    qr_code = totp_service.generate_qr_code(uri)

    # Store secret temporarily until user verifies
    user.mfa_secret_pending = secret
    await db.commit()

    return {
        "qr_code": qr_code,
        "secret": secret,  # For manual entry
        "message": "Scan the QR code and submit a verification token",
    }

@auth_router.post("/mfa/verify-setup")
async def verify_mfa_setup(
    token: str,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    if not user.mfa_secret_pending:
        raise HTTPException(400, "No MFA setup in progress")

    if not totp_service.verify_token(user.mfa_secret_pending, token):
        raise HTTPException(400, "Invalid verification token")

    # Activate MFA
    user.mfa_secret = user.mfa_secret_pending
    user.mfa_secret_pending = None
    user.mfa_enabled = True

    # Generate backup codes
    backup_codes = totp_service.generate_backup_codes()
    user.mfa_backup_codes = [
        pwd_context.hash(code) for code in backup_codes
    ]
    await db.commit()

    return {
        "message": "MFA enabled successfully",
        "backup_codes": backup_codes,  # Show once, never again
    }
```

### MFA-Aware Login Flow

```python
@auth_router.post("/token")
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(401, "Invalid credentials")

    if user.mfa_enabled:
        # Issue a short-lived MFA challenge token
        mfa_token = create_access_token(
            data={"sub": str(user.id), "type": "mfa_challenge"},
            expires_delta=timedelta(minutes=5),
        )
        return {
            "mfa_required": True,
            "mfa_token": mfa_token,
        }

    # No MFA — issue regular tokens
    return create_token_response(user)

@auth_router.post("/mfa/verify")
async def verify_mfa(
    mfa_token: str,
    totp_code: str,
    db: AsyncSession = Depends(get_db),
):
    payload = decode_jwt(mfa_token)
    if payload.get("type") != "mfa_challenge":
        raise HTTPException(400, "Invalid MFA token")

    user = await db.get(User, int(payload["sub"]))
    if not user or not user.mfa_enabled:
        raise HTTPException(400, "Invalid request")

    # Try TOTP first
    if totp_service.verify_token(user.mfa_secret, totp_code):
        return create_token_response(user)

    # Try backup codes
    for i, hashed_code in enumerate(user.mfa_backup_codes or []):
        if pwd_context.verify(totp_code, hashed_code):
            # Consume the backup code (single-use)
            user.mfa_backup_codes.pop(i)
            flag_modified(user, "mfa_backup_codes")
            await db.commit()
            return create_token_response(user)

    raise HTTPException(401, "Invalid MFA code")
```

---

## Token Rotation

### Refresh Token Rotation

Each time a refresh token is used, issue a new one and invalidate the old. This limits the window of compromise if a refresh token is stolen.

```python
class RefreshTokenStore:
    """Redis-backed refresh token store with rotation and family tracking."""

    def __init__(self, redis):
        self.redis = redis

    async def store(
        self, user_id: int, token_id: str, family_id: str, ttl: int = 604800
    ):
        key = f"refresh:{token_id}"
        await self.redis.hset(key, mapping={
            "user_id": str(user_id),
            "family_id": family_id,
            "used": "0",
        })
        await self.redis.expire(key, ttl)

        # Track all tokens in the family
        family_key = f"token_family:{family_id}"
        await self.redis.sadd(family_key, token_id)
        await self.redis.expire(family_key, ttl)

    async def use(self, token_id: str) -> dict | None:
        key = f"refresh:{token_id}"
        data = await self.redis.hgetall(key)
        if not data:
            return None

        if data.get(b"used") == b"1":
            # Token reuse detected — possible theft
            # Invalidate the entire token family
            family_id = data[b"family_id"].decode()
            await self.invalidate_family(family_id)
            return None

        # Mark as used
        await self.redis.hset(key, "used", "1")
        return {
            "user_id": int(data[b"user_id"]),
            "family_id": data[b"family_id"].decode(),
        }

    async def invalidate_family(self, family_id: str):
        """Invalidate all tokens in a family (security response to reuse)."""
        family_key = f"token_family:{family_id}"
        token_ids = await self.redis.smembers(family_key)
        pipeline = self.redis.pipeline()
        for tid in token_ids:
            pipeline.delete(f"refresh:{tid.decode()}")
        pipeline.delete(family_key)
        await pipeline.execute()

@auth_router.post("/refresh")
async def refresh_token(
    refresh_token: str,
    db: AsyncSession = Depends(get_db),
    store: RefreshTokenStore = Depends(get_refresh_store),
):
    payload = decode_jwt(refresh_token)
    token_id = payload.get("jti")

    data = await store.use(token_id)
    if not data:
        raise HTTPException(401, "Invalid or reused refresh token")

    user = await db.get(User, data["user_id"])
    if not user or not user.is_active:
        raise HTTPException(401, "User not found or inactive")

    # Issue new token pair with same family
    new_token_id = secrets.token_urlsafe(32)
    new_access = create_access_token(data={"sub": str(user.id)})
    new_refresh = create_refresh_token(
        data={"sub": str(user.id), "jti": new_token_id}
    )

    await store.store(user.id, new_token_id, data["family_id"])

    return {
        "access_token": new_access,
        "refresh_token": new_refresh,
        "token_type": "bearer",
    }
```

---

## Secure Password Reset

### Token-Based Password Reset Flow

```python
from itsdangerous import URLSafeTimedSerializer, BadSignature, SignatureExpired

reset_serializer = URLSafeTimedSerializer(settings.secret_key.get_secret_value())

@auth_router.post("/forgot-password")
async def forgot_password(
    email: str,
    request: Request,
    db: AsyncSession = Depends(get_db),
    background_tasks: BackgroundTasks = BackgroundTasks(),
):
    """Always return 200 to prevent email enumeration."""
    user = await get_user_by_email(db, email)

    if user:
        token = reset_serializer.dumps(
            {"user_id": user.id, "email": user.email},
            salt="password-reset",
        )
        reset_url = f"{settings.frontend_url}/reset-password?token={token}"

        background_tasks.add_task(
            send_password_reset_email,
            to=user.email,
            reset_url=reset_url,
        )

        await log_security_event(
            SecurityEvent.PASSWORD_RESET_REQUESTED, request, user_id=user.id
        )

    # Always return success to prevent email enumeration
    return {"message": "If the email exists, a reset link has been sent"}

@auth_router.post("/reset-password")
async def reset_password(
    token: str,
    new_password: str = Field(..., min_length=10),
    request: Request = None,
    db: AsyncSession = Depends(get_db),
):
    try:
        data = reset_serializer.loads(
            token,
            salt="password-reset",
            max_age=3600,  # 1 hour expiration
        )
    except SignatureExpired:
        raise HTTPException(400, "Reset link has expired")
    except BadSignature:
        raise HTTPException(400, "Invalid reset link")

    user = await db.get(User, data["user_id"])
    if not user or user.email != data["email"]:
        raise HTTPException(400, "Invalid reset link")

    # Update password
    user.hashed_password = hash_password(new_password)

    # Invalidate all existing sessions/tokens
    await invalidate_all_user_tokens(user.id)

    await db.commit()

    await log_security_event(
        SecurityEvent.PASSWORD_CHANGED, request, user_id=user.id
    )

    return {"message": "Password has been reset successfully"}
```

Key security considerations for password reset:
- Always return the same response regardless of whether the email exists (prevent enumeration).
- Use signed, time-limited tokens (not random database-stored codes that could be brute-forced).
- Invalidate all existing sessions after a password change.
- Rate-limit the forgot-password endpoint.
- Log all password reset events for audit.
