---
tags:
  - BE101
  - backend-4
  - auth
  - security
date: 2026-06-27
status: complete
domain: "4 of 8"
track: backend
---

# B4 — Authentication & Security

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll implement the most common authentication patterns used in production APIs: API keys, JWT, and OAuth 2.0. You'll also handle the security concerns every backend developer must understand: CORS, HTTPS, and basic web security hygiene.

---

## 4.1 — Authentication vs Authorization

> [!NOTE]
> These two words are often used interchangeably, but they mean different things — and conflating them in code causes real security holes.

**Authentication (AuthN)** — *Who are you?*
Verifying that the caller is who they claim to be. A valid token or password proves identity.

**Authorization (AuthZ)** — *What are you allowed to do?*
Deciding whether an authenticated identity has permission to perform a specific action.

| | Authentication | Authorization |
|--|----------------|--------------|
| Question | Who are you? | What can you do? |
| Happens | First | After authentication |
| HTTP status on failure | `401 Unauthorized` | `403 Forbidden` |
| Example | "Your JWT is valid — you're Alice" | "Alice cannot delete users — she's not an admin" |

> [!IMPORTANT] 401 vs 403 — Get This Right
> - `401` → the request has no valid credentials. The client should log in.
> - `403` → the client IS authenticated but doesn't have permission. Logging in again won't help.
>
> Returning `403` when you mean `401` hides the real problem from the client.

**Implementing Authorization in FastAPI**

Authorization is a natural fit for FastAPI's `Depends()` system — write a dependency once, reuse it on any route that needs it. We'll implement `get_current_user` fully in section 4.3; here's how to build role-checking on top of it:

```python
# app/dependencies/auth.py
from fastapi import Depends, HTTPException, status
from app.core.security import get_current_user  # implemented in 4.3


def require_role(role: str):
    def checker(current_user: dict = Depends(get_current_user)) -> dict:
        if current_user.get("role") != role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{role}' required",
            )
        return current_user
    return checker
```

```python
# app/routers/admin.py
from fastapi import APIRouter, Depends
from app.dependencies.auth import require_role

router = APIRouter(prefix="/admin", tags=["admin"])


@router.delete("/users/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    _: dict = Depends(require_role("admin")),
) -> None:
    ...  # only admins reach this line
```

`require_role("admin")` returns a dependency — FastAPI resolves it before the handler runs. Non-admins get a `403` automatically.

---

## 4.2 — Token Auth & API Keys

> [!NOTE]
> API keys are the simplest form of token authentication. They're best suited for machine-to-machine communication — internal services, CI scripts, CLI tools — where a human login flow is unnecessary.

**What an API key is**

An API key is a long random string (e.g. `sk-abc123...`) that the client sends with every request, typically in a header. The server checks it against stored keys to grant or deny access.

```
Client ──── X-API-Key: sk-abc123 ──────▶ Server
                                          └─ Is this key valid?
                                          ├─ No  → 401
                                          └─ Yes → proceed
```

**Implementation in FastAPI**

```python
# app/dependencies/api_key.py
from fastapi import Depends, HTTPException, Security, status
from fastapi.security import APIKeyHeader
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.db.session import get_db
from app.models.api_key import ApiKey
from app.core.security import hash_api_key

API_KEY_HEADER = APIKeyHeader(name="X-API-Key", auto_error=True)


async def verify_api_key(
    raw_key: str = Security(API_KEY_HEADER),
    db: AsyncSession = Depends(get_db),
) -> str:
    hashed = hash_api_key(raw_key)
    key_row = await db.scalar(select(ApiKey).where(ApiKey.key_hash == hashed))
    if not key_row:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key",
        )
    return raw_key
```

```python
# app/routers/data.py
from fastapi import APIRouter, Depends
from app.dependencies.api_key import verify_api_key

router = APIRouter()


@router.get("/export", dependencies=[Depends(verify_api_key)])
async def export_data() -> dict:
    return {"data": []}
```

**Generating and hashing API keys at rest**

You generate the key and give it to the client once. If your database is breached, you don't want raw keys exposed — an attacker could impersonate any client. Hashing the key means a leaked database is useless without the original.

```python
# app/core/security.py (add these alongside password utilities)
import hashlib
import secrets


def generate_api_key() -> tuple[str, str]:
    """Returns (raw_key_for_client, hashed_key_for_db)."""
    raw = secrets.token_urlsafe(32)
    hashed = hash_api_key(raw)
    return raw, hashed


def hash_api_key(raw: str) -> str:
    return hashlib.sha256(raw.encode()).hexdigest()
```

> [!WARNING] Never Hardcode API Keys
> ❌ `API_KEY = "sk-abc123"` in source code — ends up in Git history forever
> ❌ Committing `.env` files with real keys
> ✅ Load from environment variables via `pydantic-settings` (covered in [[Backend/B3 - Databases & ORM|B3]])
> ✅ Use a secrets manager (Azure Key Vault, AWS Secrets Manager) in production

**API keys vs JWT**

| | API Keys | JWT |
|--|----------|-----|
| Best for | Machine-to-machine | User-facing login |
| Lifetime | Long-lived, low rotation | Short-lived, refreshed often |
| Granularity | Coarse (all-or-nothing) | Fine (roles and scopes in payload) |
| Complexity | Simple to implement | More moving parts, more flexibility |

---

## 4.3 — JWT (JSON Web Tokens)

> [!NOTE]
> JWT is the standard token format for user authentication in REST APIs. A token is issued on login and sent with every subsequent request — no server-side session needed.

**JWT Anatomy**

A JWT looks like this:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzE5NTYwMDAwfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Three Base64-encoded parts separated by dots:

```
HEADER . PAYLOAD . SIGNATURE
```

| Part | Contains | Decoded example |
|------|----------|-----------------|
| Header | Algorithm + token type | `{"alg": "HS256", "typ": "JWT"}` |
| Payload | Claims (user data) | `{"sub": "42", "role": "user", "exp": 1719560000}` |
| Signature | HMAC of header + payload | Proves the token wasn't tampered with |

> [!IMPORTANT] The Payload Is Not Secret
> The payload is Base64-encoded, not encrypted. Anyone can decode it with `base64.b64decode()`. The signature only proves the token wasn't tampered with — it does not hide the data.
> ❌ Never put passwords, credit card numbers, or sensitive PII in a JWT payload.
> ✅ Store only what you need for routing decisions: `user_id`, `role`, `exp`.

**The Full Login Flow**

```mermaid
sequenceDiagram
    participant C as Client
    participant API as FastAPI
    participant DB as Database

    C->>API: POST /auth/register {email, password}
    API->>API: hash password with bcrypt
    API->>DB: INSERT user (email, hashed_password, role)
    DB-->>API: user created
    API-->>C: 201 Created

    C->>API: POST /auth/login {email, password}
    API->>DB: SELECT user WHERE email = ?
    DB-->>API: user row
    API->>API: bcrypt.verify(password, stored_hash)
    alt Password valid
        API->>API: create JWT {sub: user_id, role: role, exp: now+30min}
        API-->>C: 200 OK {access_token: "eyJ..."}
    else Password invalid
        API-->>C: 401 Unauthorized
    end

    C->>API: GET /me  Authorization: Bearer eyJ...
    API->>API: verify JWT signature + expiry
    alt Token valid
        API-->>C: 200 OK {user data}
    else Token invalid or expired
        API-->>C: 401 Unauthorized
    end
```

**Install**

```bash
uv add PyJWT "passlib[bcrypt]"
```

| Package | Purpose |
|---------|---------|
| `PyJWT` | Issue and validate JWTs |
| `passlib[bcrypt]` | Hash and verify passwords |

**Config additions**

Add these to your `.env`:

```dotenv
SECRET_KEY=<output of: python -c "import secrets; print(secrets.token_hex(32))">
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
```

And extend `Settings` in `app/core/config.py`:

```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379"
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    model_config = ConfigDict(env_file=".env", env_file_encoding="utf-8")
```

**Pydantic schemas**

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr, Field


class RegisterRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, description="Minimum 8 characters")


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


class RegisterResponse(BaseModel):
    id: int
    email: str
```

**User model — required fields**

The `User` SQLAlchemy model needs a `role` column for authorization to work. Add it to your model and create an Alembic migration:

```python
# app/models/user.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String, unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String)
    role: Mapped[str] = mapped_column(String, default="user")  # "user" | "admin"
```

```bash
alembic revision --autogenerate -m "add role to users"
alembic upgrade head
```

**Security utilities**

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone

import jwt
from passlib.context import CryptContext

from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_access_token(data: dict) -> str:
    payload = data.copy()
    payload["exp"] = datetime.now(timezone.utc) + timedelta(
        minutes=settings.access_token_expire_minutes
    )
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)


def decode_access_token(token: str) -> dict:
    return jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
```

**Auth router**

```python
# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.security import create_access_token, hash_password, verify_password
from app.db.session import get_db
from app.models.user import User
from app.schemas.auth import LoginRequest, RegisterRequest, RegisterResponse, TokenResponse

router = APIRouter(prefix="/auth", tags=["auth"])


@router.post("/register", status_code=201, response_model=RegisterResponse)
async def register(
    body: RegisterRequest,
    db: AsyncSession = Depends(get_db),
) -> RegisterResponse:
    user = User(email=body.email, hashed_password=hash_password(body.password))
    db.add(user)
    try:
        await db.commit()
    except IntegrityError:
        await db.rollback()
        raise HTTPException(status_code=409, detail="Email already registered")
    await db.refresh(user)
    return RegisterResponse(id=user.id, email=user.email)


@router.post("/login", response_model=TokenResponse)
async def login(
    body: LoginRequest,
    db: AsyncSession = Depends(get_db),
) -> TokenResponse:
    user = await db.scalar(select(User).where(User.email == body.email))
    if not user or not verify_password(body.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
        )
    token = create_access_token({"sub": str(user.id), "role": user.role})
    return TokenResponse(access_token=token)
```

**Protected route dependency**

```python
# app/dependencies/auth.py (add get_current_user here)
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

from app.core.security import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")


async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    try:
        return decode_access_token(token)
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token expired",
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token",
        )
```

```python
# Usage on any protected route
@router.get("/me")
async def get_me(current_user: dict = Depends(get_current_user)) -> dict:
    return {"user_id": current_user["sub"], "role": current_user["role"]}
```

**Access token vs refresh token**

An access token is short-lived (15–60 minutes) so a stolen token expires quickly. A refresh token is long-lived (days/weeks), stored securely, and used only to issue new access tokens — not to access resources directly. Implementing refresh token rotation (storing in DB, rotating on use, revoking on logout) is a more advanced pattern. For now, understanding *why* two tokens exist is the goal.

> [!NOTE] Where to Store Tokens (Client-Side)
> This is a frontend decision, but backend devs get asked it in design reviews:
> - **`localStorage`** — easy to read from JS, but XSS attacks can steal it
> - **`httpOnly` cookie** — JS cannot read it (XSS-safe), but needs `SameSite` config to mitigate CSRF
>
> Most production apps use `httpOnly` cookies for refresh tokens and keep access tokens short-lived.

> [!TIP] Rate-Limit Your Login Endpoint
> Login endpoints are brute-force targets. In production, always apply rate limiting — `slowapi` is a FastAPI-compatible option. Even a simple "5 attempts per minute per IP" rule dramatically reduces credential stuffing attacks.

> [!WARNING] Common JWT Mistakes
> ❌ Sensitive data (passwords, PII) in the payload — it's Base64, not encrypted
> ❌ No `exp` claim — token is valid forever if stolen
> ❌ Weak or hardcoded secret key — brute-forceable offline
> ❌ Using `python-jose` — unmaintained since 2022; use `PyJWT` instead
> ✅ Short expiry (30 min), strong random secret, payload contains only `sub` + `role` + `exp`

**Refresh Tokens**

Access tokens are short-lived (15–60 minutes) to limit damage from theft. A **refresh token** is a long-lived credential (7–30 days) stored securely by the client. When the access token expires, the client exchanges the refresh token for a new access token — without asking the user to log in again.

```python
# app/core/security.py — add refresh token support (alongside existing functions)
# ... existing imports ...
REFRESH_TOKEN_EXPIRE_DAYS = 7


def create_access_token(data: dict) -> str:
    payload = data.copy()
    payload["exp"] = datetime.now(timezone.utc) + timedelta(
        minutes=settings.access_token_expire_minutes
    )
    payload["type"] = "access"
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)


def create_refresh_token(data: dict) -> str:
    expires = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    return jwt.encode(
        {**data, "exp": expires, "type": "refresh"},
        settings.secret_key,
        algorithm=settings.algorithm,
    )
```

Update `TokenResponse` and `login` to return both tokens:

```python
# app/schemas/auth.py
class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
```

```python
# app/routers/auth.py — update login endpoint
@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, db: AsyncSession = Depends(get_db)) -> TokenResponse:
    user = await db.scalar(select(User).where(User.email == body.email))
    if not user or not verify_password(body.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    payload = {"sub": str(user.id), "role": user.role}
    return TokenResponse(
        access_token=create_access_token(payload),
        refresh_token=create_refresh_token(payload),
    )


class RefreshRequest(BaseModel):
    refresh_token: str


@router.post("/refresh", response_model=TokenResponse)
async def refresh(body: RefreshRequest) -> TokenResponse:
    try:
        payload = jwt.decode(body.refresh_token, settings.secret_key, algorithms=[settings.algorithm])
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="Invalid token type")
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Refresh token expired — please log in again")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid refresh token")
    data = {"sub": payload["sub"], "role": payload["role"]}
    return TokenResponse(
        access_token=create_access_token(data),
        refresh_token=create_refresh_token(data),   # rotate refresh token on each use
    )
```

> [!TIP] Rotate Refresh Tokens
> The example above issues a **new refresh token on every refresh**. This is called refresh token rotation — if a stolen refresh token is used, the legitimate user's next refresh will fail (the old token is already spent), alerting you to a potential breach. Store refresh tokens server-side (Redis or DB) for full revocation support.

> [!WARNING] Where to Store Refresh Tokens (Frontend)
> - ✅ `HttpOnly` cookie — not accessible to JavaScript, safe from XSS
> - ❌ `localStorage` — readable by any JavaScript on the page; vulnerable to XSS
> - ❌ In-memory JS variable — lost on page refresh; forces re-login on every tab close

---

## 4.4 — OAuth 2.0

> [!NOTE]
> OAuth 2.0 is a protocol for **delegated authorization** — it lets a user grant your app limited access to their account on another service, without sharing their password with you.

**What OAuth 2.0 is (and isn't)**

OAuth 2.0 answers: *"Can this app access my GitHub repos on my behalf?"*
It does **not** answer: *"Who is this user?"* — that's OpenID Connect (OIDC), which builds on OAuth 2.0 and adds an identity layer.

> [!TIP] OAuth 2.0 vs OpenID Connect
> **OAuth 2.0** = authorization (can this app access a resource?). **OIDC** = authentication (who is this user?) — it extends OAuth 2.0 with an `id_token` that contains user identity. Most "Login with Google/GitHub" buttons use OIDC under the hood.

**Authorization Code Flow**

The most secure OAuth flow for web apps. This diagram shows what a "Login with GitHub" integration looks like conceptually — no hands-on setup required at this stage:

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant App as Your App
    participant GH as GitHub (OAuth Provider)

    U->>App: Click "Login with GitHub"
    App->>U: Redirect → github.com/login/oauth/authorize?client_id=...&state=xyz
    U->>GH: Log in + approve permissions
    GH->>App: Redirect → /callback?code=abc&state=xyz
    App->>App: Validate state matches xyz (CSRF check)
    App->>GH: POST /login/oauth/access_token {code, client_secret}
    GH-->>App: {access_token, scope}
    App->>GH: GET /user  Authorization: Bearer access_token
    GH-->>App: {id, login, email}
    App->>U: Logged in as alice@github.com
```

> [!IMPORTANT] The `state` Parameter
> The `state` value your app sends in the initial redirect must match what the provider returns in the callback. This prevents CSRF attacks on the OAuth handshake — always validate it.

**Hands-on: OAuth2PasswordBearer (first-party flow)**

FastAPI ships `OAuth2PasswordBearer` — a simplified flow designed for **your own** login form, not third-party delegation. This is what powers the `/docs` "Authorize" button and what you already wired up in section 4.3.

```python
from fastapi.security import OAuth2PasswordBearer

# Points to your login endpoint — enables the /docs Authorize button
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")
```

**Scopes**

Scopes are strings that define what an access token is allowed to do. A token with `read:user` cannot write even if the user has that permission — scopes narrow the blast radius of a compromised token.

| Scope | Meaning |
|-------|---------|
| `read:user` | Read user profile |
| `write:repos` | Create and modify repositories |
| `admin:org` | Full organization admin access |

---

## 4.5 — CORS

> [!NOTE]
> CORS (Cross-Origin Resource Sharing) controls which external origins browsers allow to make requests to your API.

**The Same-Origin Policy**

Browsers enforce a rule: JavaScript on `https://app.com` cannot make requests to `https://api.other.com` by default. This prevents malicious websites from silently calling APIs using a visitor's credentials.

An **origin** is the combination of scheme + host + port:
- `https://app.com` and `https://api.app.com` → different origins (different subdomain)
- `http://localhost:3000` and `http://localhost:8000` → different origins (different port)

> [!IMPORTANT] CORS Is Enforced by Browsers, Not Servers
> Your server doesn't block the request — the **browser** does, after inspecting the server's response headers. This means:
> - `curl`, Postman, and server-to-server calls are **never affected by CORS**
> - Only browser-based JavaScript is subject to CORS restrictions
>
> If your Postman test passes but your frontend gets a CORS error, this is why.

**How CORS Works: The Preflight**

For non-simple requests (e.g. POST with JSON body, custom headers), the browser sends an `OPTIONS` preflight before the real request:

```http
OPTIONS /api/users HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

Your server must respond with what's allowed:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

If the preflight fails, the browser blocks the real request entirely.

**CORSMiddleware in FastAPI**

```python
from fastapi.middleware.cors import CORSMiddleware

# Development
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Production
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.yourcompany.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

| Parameter | Controls |
|-----------|---------|
| `allow_origins` | Which domains can make requests |
| `allow_methods` | Which HTTP methods are permitted |
| `allow_headers` | Which request headers are permitted |
| `allow_credentials` | Whether cookies and auth headers are forwarded |

> [!WARNING] Wildcard + Credentials = Security Misconfiguration
> ❌ `allow_origins=["*"]` + `allow_credentials=True` — browsers reject this combination outright
> ❌ `allow_origins=["*"]` in production — any website can call your API on behalf of your users
> ✅ Always enumerate exact production origins

---

## 4.6 — HTTPS & Web Security Basics

> [!NOTE]
> Authentication only works if the channel is secure. HTTPS prevents tokens and passwords from being intercepted in transit. Security headers and awareness of common vulnerabilities round out a production-ready backend.

**HTTPS and TLS**

HTTP sends data as plain text. Anyone on the same network — café Wi-Fi, ISP, compromised router — can read it. TLS (Transport Layer Security) encrypts the connection so only the two endpoints can read the data.

What happens during a TLS handshake (simplified):
1. Client says: "I want a secure connection, here are the cipher suites I support"
2. Server sends its **certificate** (signed by a Certificate Authority) + public key
3. Client verifies the certificate is trusted
4. Both sides derive a shared symmetric encryption key
5. All data from this point is encrypted

> [!IMPORTANT] Never Run Production APIs on Plain HTTP
> A login endpoint over HTTP means passwords and tokens travel as plain text. One packet capture and your users' credentials are exposed.

**Force HTTPS with FastAPI middleware**

For development and simple deployments, FastAPI/Starlette ships a one-liner redirect:

```python
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

app.add_middleware(HTTPSRedirectMiddleware)
```

Any request arriving over HTTP is automatically redirected to HTTPS. In production behind nginx this is typically handled at the proxy level instead.

**TLS Termination Pattern**

In production, TLS is handled by a reverse proxy in front of your app — not by FastAPI itself:

```
Browser ──HTTPS──▶ nginx (TLS termination) ──HTTP──▶ FastAPI (port 8000)
                   └── holds TLS cert
                   └── handles handshake
```

nginx manages the certificate and encryption. FastAPI only sees plain HTTP internally. For hands-on setup, see the [nginx docs](https://nginx.org/en/docs/) and [Let's Encrypt getting started guide](https://letsencrypt.org/getting-started/).

**Security Headers**

Headers your server sends that tell browsers how to handle your content:

| Header | What it does |
|--------|-------------|
| `Strict-Transport-Security` (HSTS) | Forces browsers to always use HTTPS for your domain, even if the user types `http://` |
| `X-Content-Type-Options: nosniff` | Prevents browsers from guessing content types — blocks MIME sniffing attacks |
| `X-Frame-Options: DENY` | Prevents your page from being embedded in an iframe — blocks clickjacking |
| `Content-Security-Policy` | Restricts which scripts, styles, and resources the browser will load |

Add them via middleware:

```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        return response


app.add_middleware(SecurityHeadersMiddleware)
```

**OWASP Top 10 — Most Relevant for Backend Devs**

The OWASP Top 10 is the industry-standard list of critical web security risks. Four are directly relevant to what you've built in B1–B4:

| Risk | What it means | Your mitigation |
|------|--------------|----------------|
| **A01 — Broken Access Control** | Users access resources they shouldn't — missing authZ checks | `require_role()` dependency; always verify ownership before returning data |
| **A02 — Cryptographic Failures** | Sensitive data exposed — plain-text passwords, weak algorithms | `passlib[bcrypt]` for passwords; HTTPS everywhere; no secrets in JWT payload |
| **A07 — Identification & Auth Failures** | Weak auth — brute-forceable, no expiry, stolen tokens reused indefinitely | Short JWT expiry; rate-limit login; `httpOnly` cookies for tokens |
| **A05 — Security Misconfiguration** | Default configs, debug mode in prod, wildcard CORS | Lock down CORS origins; disable debug mode; strip stack traces from error responses |

> [!WARNING] Anti-Pattern Hall of Shame
> ❌ Storing passwords as MD5 or SHA-1 — cracked in seconds with precomputed tables
> ❌ `DEBUG=True` in production — exposes full stack traces and internal paths
> ❌ JWT secret key committed to Git — anyone with repo access can forge any token
> ❌ `allow_origins=["*"]` in production — any site can call your API as your users
> ❌ No HTTPS in staging — security bugs only surface in production, where they cost the most
> ✅ bcrypt for passwords, strong random secrets, explicit CORS origins, HTTPS from day one

---

## 🎯 What You Learned

You can now:

- **Implement API key and JWT authentication** — hash API keys with SHA-256, issue JWTs with `PyJWT`, decode and validate tokens in a `Depends()` guard, and rotate refresh tokens to limit exposure
- **Hash and verify passwords securely** — `bcrypt` via `passlib` with automatic salt and work factor; never store or compare plaintext passwords
- **Guard routes by role** — `get_current_user` dependency chain that decodes the JWT and enforces `role == "admin"` before the handler runs
- **Understand OAuth 2.0** — the Authorization Code Flow, what each step exchanges, and what the `state` parameter prevents (CSRF on the redirect)
- **Apply HTTPS and security headers** — TLS termination at the reverse proxy, HSTS, `X-Content-Type-Options`, `X-Frame-Options`, and the OWASP Top 10 threats to design against

---

## ✅ Practice Checklist

- [ ] Implement an API key guard on at least one endpoint using `APIKeyHeader`
- [ ] Write a `/auth/register` endpoint that hashes passwords with `passlib[bcrypt]`
- [ ] Write a `/auth/login` endpoint that verifies a password and returns a JWT using `PyJWT`
- [ ] Write a `get_current_user` dependency that validates a JWT from the `Authorization` header
- [ ] Protect at least one route with `Depends(get_current_user)`
- [ ] Implement a `require_role()` dependency and protect one admin route with it
- [ ] Decode a JWT payload manually with `base64.b64decode()` and confirm the payload is readable plain text
- [ ] Configure `CORSMiddleware` to allow only `http://localhost:3000` and verify a cross-origin request works
- [ ] Add `HTTPSRedirectMiddleware`, HSTS, `X-Content-Type-Options`, and `X-Frame-Options` to the app
- [ ] Draw the OAuth 2.0 Authorization Code Flow from memory, labeling where the `state` parameter is sent and validated
- [ ] Explain the difference between `401` and `403` to a colleague without referring to notes

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://fastapi.tiangolo.com/tutorial/security/ | FastAPI security and OAuth2 docs |
| https://pyjwt.readthedocs.io/en/stable/ | PyJWT — issue and validate JWTs |
| https://passlib.readthedocs.io/en/stable/ | passlib — password hashing with bcrypt |
| https://oauth.net/2/ | OAuth 2.0 specification |
| https://openid.net/connect/ | OpenID Connect overview |
| https://jwt.io | JWT decoder and anatomy reference |
| https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS | CORS — MDN reference |
| https://owasp.org/www-project-top-ten/ | OWASP Top 10 web security risks |
| https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security | HSTS header reference |
| https://slowapi.readthedocs.io/en/latest/ | slowapi — rate limiting for FastAPI |

## 🃏 Quick-Reference Flash Cards

**Q:** What is the difference between authentication and authorization?
**A:** Authentication (AuthN) verifies *who you are* — identity. Authorization (AuthZ) decides *what you can do* — permissions. AuthN always happens first.

**Q:** When do you return `401` vs `403`?
**A:** `401` — no valid credentials; the client should log in. `403` — authenticated but not permitted; logging in again won't help.

**Q:** What are the three parts of a JWT?
**A:** Header (algorithm + token type), Payload (claims: `sub`, `role`, `exp`), Signature (HMAC proving the token wasn't tampered with).

**Q:** Is the JWT payload encrypted?
**A:** No — it is Base64-encoded, not encrypted. Anyone can decode it. Never put passwords or sensitive PII in the payload.

**Q:** Why use bcrypt for passwords instead of SHA-256?
**A:** bcrypt is deliberately slow and salted, making brute-force and rainbow table attacks impractical. SHA-256 is fast and deterministic — trivially crackable with precomputed tables.

**Q:** Why is CORS enforced by the browser, not the server?
**A:** CORS is a browser security policy protecting users from malicious websites silently calling APIs on their behalf. `curl` and Postman don't enforce it because they're not browsers acting on behalf of a logged-in user.

**Q:** What does the OAuth 2.0 `state` parameter do?
**A:** It's a random value your app sends in the initial redirect. When the provider returns the user to your callback, you validate that `state` matches — preventing CSRF attacks on the OAuth handshake.

**Q:** What is the difference between OAuth 2.0 and OpenID Connect?
**A:** OAuth 2.0 handles *authorization* (what can this app access?). OIDC builds on OAuth 2.0 and adds *authentication* (who is the user?) via an `id_token`.

**Q:** What does `allow_origins=["*"]` + `allow_credentials=True` cause in CORS?
**A:** Browsers reject the response entirely. Credentialed CORS requests require explicit origin enumeration — wildcards are not allowed with credentials.

**Q:** What is TLS termination and why is it done at the proxy rather than the app?
**A:** TLS termination is where the encrypted HTTPS connection is unwrapped. Doing it at a reverse proxy (nginx) keeps the app simple, centralizes certificate management, and lets multiple app instances share one TLS config.

*Checkpoint: [[Backend/Checkpoints/CB4 - Auth & Security Verified|CB4]]*

*Previous: [[Backend/B3 - Databases & ORM|B3]] | Next: [[Backend/B5 - Testing & Code Quality|B5]]*
