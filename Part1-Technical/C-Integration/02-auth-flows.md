# Authentication Flows — OAuth2, API Keys, mTLS, JWT

**FDE Course | Part 1: Technical | Section C: Integration**

> **FDE Principle**: In a single customer engagement you might encounter four different auth mechanisms across four different vendor APIs. You cannot Google your way through every implementation under pressure. This file is designed so you can implement any of these auth patterns from memory.

---

## Table of Contents

1. [Auth Landscape for FDEs](#auth-landscape)
2. [OAuth2: Authorization Code Flow](#oauth2-auth-code)
3. [OAuth2: Client Credentials Flow](#oauth2-client-credentials)
4. [Token Caching and Refresh Token Rotation](#token-caching)
5. [API Keys: Header, Query Param, Bearer](#api-keys)
6. [HMAC-Signed Requests](#hmac-signed-requests)
7. [Mutual TLS (mTLS)](#mtls)
8. [JWT Service-to-Service Auth](#jwt-service-to-service)
9. [Combining Auth Patterns](#combining-auth)
10. [Secrets Management](#secrets-management)
11. [Common Failure Modes](#common-failure-modes)
12. [Interview Angles](#interview-angles)
13. [Practice Exercise](#practice-exercise)

---

## Auth Landscape for FDEs {#auth-landscape}

Before writing a line of code, classify the integration:

| Scenario | Typical Auth | Your Implementation |
|----------|-------------|---------------------|
| User-facing OAuth app (e.g., connect your Shopify store) | OAuth2 Authorization Code | Browser redirect flow, token storage per-user |
| Service-to-service API (e.g., your backend to carrier API) | OAuth2 Client Credentials | Background token management, cached tokens |
| Legacy carrier API | API Key (X-API-Key) | Simple header injection |
| Shopify/Stripe webhook requests you receive | HMAC signature verification | Verify incoming signatures |
| Shopify/Stripe APIs you call | Bearer token | Bearer in Authorization header |
| AWS Signature APIs (SNS, S3, Bedrock) | HMAC-SHA256 AWS SigV4 | Sign each request |
| Financial/government APIs | mTLS (Mutual TLS) | Client certificate loaded in HTTP client |
| Internal microservices | JWT signed assertions | Create and verify JWTs |

The #1 FDE mistake: treating all auth as "just add a header." Each mechanism has failure modes that are completely different from each other.

---

## OAuth2: Authorization Code Flow {#oauth2-auth-code}

### When you use this

When a **user** (human) needs to authorize your application to act on their behalf. Examples:
- "Connect your Shopify store" — user logs in to Shopify and grants permissions
- "Authorize access to your Google Workspace" — user logs in to Google
- Any time you see a "Login with X" button

### The full flow

```
1. Your app redirects user → authorization server with:
   - client_id: who you are
   - redirect_uri: where to send the user after authorization
   - scope: what permissions you're requesting
   - state: random value you generate to prevent CSRF
   - response_type=code

2. User authenticates at authorization server, grants consent

3. Authorization server redirects user back to your redirect_uri with:
   - code: short-lived (usually 10 min) authorization code
   - state: the value you sent (verify it matches!)

4. Your backend (server-side) exchanges code for tokens:
   POST /oauth/token
   {
     grant_type: "authorization_code",
     code: <the code>,
     redirect_uri: <same URI as step 1>,
     client_id: <your client id>,
     client_secret: <your client secret>
   }
   
5. Authorization server returns:
   {
     "access_token": "...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "...",
     "scope": "read:orders write:shipments"
   }

6. Use access_token in Authorization: Bearer <token> header

7. When access_token expires: POST /oauth/token with grant_type=refresh_token
```

### Full implementation with FastAPI

```python
import secrets
import hashlib
import base64
import logging
from urllib.parse import urlencode, urlparse, parse_qs
from datetime import datetime, UTC, timedelta
from typing import Optional

import httpx
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import RedirectResponse

logger = logging.getLogger(__name__)

app = FastAPI()


class OAuthConfig:
    def __init__(
        self,
        client_id: str,
        client_secret: str,
        authorization_url: str,
        token_url: str,
        redirect_uri: str,
        scopes: list[str],
    ):
        self.client_id = client_id
        self.client_secret = client_secret
        self.authorization_url = authorization_url
        self.token_url = token_url
        self.redirect_uri = redirect_uri
        self.scopes = scopes


# In-memory state store — use Redis or DB in production
_oauth_states: dict[str, dict] = {}
# In-memory token store — use DB in production
_user_tokens: dict[str, dict] = {}


def generate_pkce_challenge() -> tuple[str, str]:
    """
    PKCE (Proof Key for Code Exchange) — required for public clients.
    Increasingly required by auth servers even for confidential clients.
    
    code_verifier: random 43-128 character string
    code_challenge: BASE64URL(SHA256(code_verifier))
    """
    code_verifier = secrets.token_urlsafe(96)  # 128 chars
    digest = hashlib.sha256(code_verifier.encode()).digest()
    code_challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
    return code_verifier, code_challenge


@app.get("/auth/connect")
async def start_oauth_flow(user_id: str) -> RedirectResponse:
    """
    Step 1: Redirect user to authorization server.
    """
    config = OAuthConfig(
        client_id="your_client_id",
        client_secret="your_client_secret",
        authorization_url="https://auth.example.com/oauth/authorize",
        token_url="https://auth.example.com/oauth/token",
        redirect_uri="https://yourapp.com/auth/callback",
        scopes=["read:orders", "write:shipments"],
    )
    
    # CSRF protection: random state bound to this user's request
    state = secrets.token_urlsafe(32)
    
    # PKCE: optional but recommended
    code_verifier, code_challenge = generate_pkce_challenge()
    
    # Store state + verifier server-side (validate in callback)
    _oauth_states[state] = {
        "user_id": user_id,
        "code_verifier": code_verifier,
        "created_at": datetime.now(UTC).isoformat(),
    }
    
    params = {
        "response_type": "code",
        "client_id": config.client_id,
        "redirect_uri": config.redirect_uri,
        "scope": " ".join(config.scopes),
        "state": state,
        "code_challenge": code_challenge,
        "code_challenge_method": "S256",
    }
    
    authorization_url = f"{config.authorization_url}?{urlencode(params)}"
    
    logger.info(
        "Starting OAuth flow",
        extra={"user_id": user_id, "state": state[:8] + "..."},
    )
    
    return RedirectResponse(url=authorization_url)


@app.get("/auth/callback")
async def oauth_callback(
    request: Request,
    code: Optional[str] = None,
    state: Optional[str] = None,
    error: Optional[str] = None,
    error_description: Optional[str] = None,
) -> dict:
    """
    Step 3-5: Handle authorization callback.
    """
    # Handle user-denied authorization
    if error:
        logger.warning(
            "OAuth authorization denied",
            extra={"error": error, "error_description": error_description}
        )
        raise HTTPException(
            status_code=400,
            detail=f"Authorization failed: {error_description or error}"
        )
    
    if not code or not state:
        raise HTTPException(status_code=400, detail="Missing code or state")
    
    # Validate state — prevents CSRF
    state_data = _oauth_states.pop(state, None)
    if not state_data:
        logger.warning("OAuth callback with unknown state — possible CSRF")
        raise HTTPException(status_code=400, detail="Invalid state parameter")
    
    # Check state isn't too old (10 minute window)
    created_at = datetime.fromisoformat(state_data["created_at"])
    if (datetime.now(UTC) - created_at).total_seconds() > 600:
        raise HTTPException(status_code=400, detail="State expired — restart authorization")
    
    user_id = state_data["user_id"]
    code_verifier = state_data["code_verifier"]
    
    # Exchange code for tokens
    config = OAuthConfig(
        client_id="your_client_id",
        client_secret="your_client_secret",
        authorization_url="https://auth.example.com/oauth/authorize",
        token_url="https://auth.example.com/oauth/token",
        redirect_uri="https://yourapp.com/auth/callback",
        scopes=["read:orders", "write:shipments"],
    )
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.post(
            config.token_url,
            data={
                "grant_type": "authorization_code",
                "code": code,
                "redirect_uri": config.redirect_uri,
                "client_id": config.client_id,
                "client_secret": config.client_secret,
                "code_verifier": code_verifier,  # PKCE
            },
            headers={"Accept": "application/json"},
        )
        
        if response.status_code != 200:
            logger.error(
                "Token exchange failed",
                extra={
                    "status_code": response.status_code,
                    "body": response.text[:500],
                }
            )
            raise HTTPException(
                status_code=502,
                detail=f"Token exchange failed: {response.status_code}"
            )
        
        token_data = response.json()
    
    # Store tokens for this user
    _user_tokens[user_id] = {
        "access_token": token_data["access_token"],
        "refresh_token": token_data.get("refresh_token"),
        "token_type": token_data.get("token_type", "Bearer"),
        "expires_at": (
            datetime.now(UTC) + timedelta(seconds=token_data.get("expires_in", 3600))
        ).isoformat(),
        "scope": token_data.get("scope"),
    }
    
    logger.info(
        "OAuth tokens obtained",
        extra={"user_id": user_id, "scope": token_data.get("scope")}
    )
    
    return {"status": "connected", "user_id": user_id}
```

---

## OAuth2: Client Credentials Flow {#oauth2-client-credentials}

### When you use this

Service-to-service. No human in the loop. Your backend authenticates as itself (not as a user). This is the most common OAuth2 flow in B2B integrations.

Examples:
- Your logistics platform → carrier's API
- Your warehouse system → freight broker's API
- Your backend → Azure AD-protected ERP API

### The flow

```
1. Your service → POST /oauth/token with:
   {
     "grant_type": "client_credentials",
     "client_id": "your_service_id",
     "client_secret": "your_secret",
     "scope": "read:rates write:shipments"
   }

2. Auth server returns:
   {
     "access_token": "...",
     "token_type": "Bearer",
     "expires_in": 3600
   }
   (No refresh_token — just get a new token when this one expires)

3. Use token in Authorization: Bearer <token>

4. Token expires → repeat step 1
```

### Production implementation with token caching

```python
import asyncio
import logging
import time
from dataclasses import dataclass
from typing import Optional

import httpx

logger = logging.getLogger(__name__)


@dataclass
class OAuthToken:
    """Represents a cached OAuth2 access token."""
    access_token: str
    token_type: str
    expires_at: float  # Unix timestamp
    scope: Optional[str] = None
    
    def is_expired(self, buffer_seconds: int = 60) -> bool:
        """
        Returns True if token is expired or will expire within buffer_seconds.
        
        Buffer prevents using a token that expires mid-request.
        60 seconds is a safe buffer for most APIs.
        """
        return time.time() >= (self.expires_at - buffer_seconds)
    
    @property
    def authorization_header(self) -> str:
        return f"{self.token_type} {self.access_token}"


class OAuth2ClientCredentials:
    """
    OAuth2 Client Credentials token manager.
    
    Features:
    - Caches token in memory — single HTTP call per token lifetime
    - Thread-safe token refresh (asyncio.Lock prevents thundering herd on expiry)
    - Configurable expiry buffer
    - Works with any OAuth2-compliant server (Azure AD, Okta, Auth0, etc.)
    
    Usage:
        auth = OAuth2ClientCredentials(
            token_url="https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token",
            client_id="your-client-id",
            client_secret="your-client-secret",
            scope="https://your-api.com/.default",
        )
        
        # In your request handler:
        token = await auth.get_token()
        headers = {"Authorization": token.authorization_header}
    """
    
    def __init__(
        self,
        token_url: str,
        client_id: str,
        client_secret: str,
        scope: Optional[str] = None,
        expiry_buffer_seconds: int = 60,
        extra_params: Optional[dict] = None,
    ):
        self.token_url = token_url
        self.client_id = client_id
        self.client_secret = client_secret
        self.scope = scope
        self.expiry_buffer = expiry_buffer_seconds
        self.extra_params = extra_params or {}
        
        self._token: Optional[OAuthToken] = None
        self._lock = asyncio.Lock()
        self._http_client = httpx.AsyncClient(timeout=30.0)
    
    async def get_token(self) -> OAuthToken:
        """
        Get a valid access token.
        Fetches a new one if expired or not yet obtained.
        Uses a lock to prevent multiple simultaneous refresh calls.
        """
        # Fast path: token is valid, no lock needed
        if self._token and not self._token.is_expired(self.expiry_buffer):
            return self._token
        
        # Slow path: need to refresh — acquire lock
        async with self._lock:
            # Double-check after acquiring lock (another coroutine may have refreshed)
            if self._token and not self._token.is_expired(self.expiry_buffer):
                return self._token
            
            self._token = await self._fetch_token()
            return self._token
    
    async def _fetch_token(self) -> OAuthToken:
        """Fetch a new token from the authorization server."""
        data = {
            "grant_type": "client_credentials",
            "client_id": self.client_id,
            "client_secret": self.client_secret,
        }
        
        if self.scope:
            data["scope"] = self.scope
        
        data.update(self.extra_params)
        
        logger.info(
            "Fetching OAuth2 client credentials token",
            extra={
                "token_url": self.token_url,
                "client_id": self.client_id,
                "scope": self.scope,
            }
        )
        
        try:
            response = await self._http_client.post(
                self.token_url,
                data=data,
                headers={"Accept": "application/json"},
            )
            response.raise_for_status()
        except httpx.HTTPStatusError as e:
            logger.error(
                "OAuth2 token fetch failed",
                extra={
                    "status_code": e.response.status_code,
                    "body": e.response.text[:500],
                }
            )
            raise RuntimeError(
                f"Failed to obtain OAuth2 token: HTTP {e.response.status_code}"
            ) from e
        
        token_data = response.json()
        
        if "error" in token_data:
            raise RuntimeError(
                f"OAuth2 error: {token_data['error']}: {token_data.get('error_description')}"
            )
        
        expires_in = token_data.get("expires_in", 3600)
        
        token = OAuthToken(
            access_token=token_data["access_token"],
            token_type=token_data.get("token_type", "Bearer"),
            expires_at=time.time() + expires_in,
            scope=token_data.get("scope"),
        )
        
        logger.info(
            "OAuth2 token obtained",
            extra={
                "expires_in_seconds": expires_in,
                "scope": token.scope,
            }
        )
        
        return token
    
    async def close(self):
        await self._http_client.aclose()


class OAuth2HTTPClient:
    """
    httpx AsyncClient that automatically injects OAuth2 Bearer tokens.
    
    Wrap any httpx client to get automatic token management:
    - Token fetched on first request
    - Token refreshed automatically when expired
    - Token refresh is transparent to callers
    """
    
    def __init__(
        self,
        auth: OAuth2ClientCredentials,
        base_url: str,
        timeout: float = 30.0,
    ):
        self.auth = auth
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout,
        )
    
    async def _get_headers(self) -> dict:
        token = await self.auth.get_token()
        return {"Authorization": token.authorization_header}
    
    async def get(self, url: str, **kwargs) -> httpx.Response:
        headers = await self._get_headers()
        headers.update(kwargs.pop("headers", {}))
        response = await self._client.get(url, headers=headers, **kwargs)
        
        # Handle token expiry mid-session (clock skew, early revocation)
        if response.status_code == 401:
            # Force token refresh
            self.auth._token = None
            headers = await self._get_headers()
            response = await self._client.get(url, headers=headers, **kwargs)
        
        return response
    
    async def post(self, url: str, **kwargs) -> httpx.Response:
        headers = await self._get_headers()
        headers.update(kwargs.pop("headers", {}))
        response = await self._client.post(url, headers=headers, **kwargs)
        
        if response.status_code == 401:
            self.auth._token = None
            headers = await self._get_headers()
            response = await self._client.post(url, headers=headers, **kwargs)
        
        return response
    
    async def close(self):
        await self._client.aclose()
        await self.auth.close()


# Azure AD example — very common in enterprise B2B
def make_azure_ad_client(
    tenant_id: str,
    client_id: str,
    client_secret: str,
    api_scope: str,  # e.g. "https://your-api.azurewebsites.net/.default"
    api_base_url: str,
) -> OAuth2HTTPClient:
    """
    Build an OAuth2 client pre-configured for Azure AD (Microsoft Entra ID).
    
    Azure AD uses the v2.0 endpoint and requires scope to end in '/.default'
    for client credentials flow when accessing Azure-registered APIs.
    """
    auth = OAuth2ClientCredentials(
        token_url=f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token",
        client_id=client_id,
        client_secret=client_secret,
        scope=api_scope,
    )
    return OAuth2HTTPClient(auth=auth, base_url=api_base_url)
```

---

## Token Caching and Refresh Token Rotation {#token-caching}

### Why token caching matters

Without caching: every API call → fetch new token → API call = 2x HTTP round trips, 2x latency.

With caching: first call → fetch token → cache → subsequent calls use cache = 1 round trip until token expires.

### Refresh token rotation (Auth Code flow only)

Some auth servers issue a new refresh token each time you use one. The old refresh token becomes invalid. This requires:
1. Storing the new refresh token atomically
2. Handling concurrent refresh attempts (two requests both try to use the same refresh token)

```python
import asyncio
import logging
from datetime import datetime, UTC, timedelta
from typing import Optional
import time

logger = logging.getLogger(__name__)


class RefreshTokenStore:
    """
    Persistent token storage with refresh token rotation support.
    In production: use a database or secrets manager (AWS Secrets Manager, Vault).
    This example uses a dict — replace with your storage backend.
    """
    
    def __init__(self):
        self._store: dict[str, dict] = {}
        self._refresh_lock: dict[str, asyncio.Lock] = {}
    
    async def get_valid_token(
        self,
        user_id: str,
        token_url: str,
        client_id: str,
        client_secret: str,
        redirect_uri: str,
        expiry_buffer: int = 60,
    ) -> Optional[str]:
        """
        Get a valid access token for user_id.
        Refreshes automatically if expired.
        Thread-safe: only one refresh per user at a time.
        """
        token_data = self._store.get(user_id)
        if not token_data:
            return None
        
        expires_at = datetime.fromisoformat(token_data["expires_at"])
        is_expired = datetime.now(UTC) >= (expires_at - timedelta(seconds=expiry_buffer))
        
        if not is_expired:
            return token_data["access_token"]
        
        if not token_data.get("refresh_token"):
            # No refresh token — user must re-authorize
            logger.warning(
                "Access token expired and no refresh token available",
                extra={"user_id": user_id}
            )
            return None
        
        # Acquire per-user lock to prevent concurrent refresh
        if user_id not in self._refresh_lock:
            self._refresh_lock[user_id] = asyncio.Lock()
        
        async with self._refresh_lock[user_id]:
            # Re-check after acquiring lock — another coroutine may have refreshed
            token_data = self._store.get(user_id)
            if token_data:
                expires_at = datetime.fromisoformat(token_data["expires_at"])
                if datetime.now(UTC) < (expires_at - timedelta(seconds=expiry_buffer)):
                    return token_data["access_token"]
            
            # Do the actual refresh
            new_token_data = await self._refresh_token(
                refresh_token=token_data["refresh_token"],
                token_url=token_url,
                client_id=client_id,
                client_secret=client_secret,
                redirect_uri=redirect_uri,
            )
            
            if new_token_data:
                self._store[user_id] = new_token_data
                return new_token_data["access_token"]
            
            return None
    
    async def _refresh_token(
        self,
        refresh_token: str,
        token_url: str,
        client_id: str,
        client_secret: str,
        redirect_uri: str,
    ) -> Optional[dict]:
        """Perform the token refresh request."""
        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                token_url,
                data={
                    "grant_type": "refresh_token",
                    "refresh_token": refresh_token,
                    "client_id": client_id,
                    "client_secret": client_secret,
                    "redirect_uri": redirect_uri,
                },
                headers={"Accept": "application/json"},
            )
        
        if response.status_code == 400:
            # Refresh token expired or revoked — user must re-authorize
            error = response.json().get("error", "")
            if error in ("invalid_grant", "refresh_token_not_found"):
                logger.warning("Refresh token invalid — user must re-authorize")
                return None
        
        response.raise_for_status()
        data = response.json()
        
        expires_in = data.get("expires_in", 3600)
        
        result = {
            "access_token": data["access_token"],
            "refresh_token": data.get("refresh_token", refresh_token),  # rotation
            "token_type": data.get("token_type", "Bearer"),
            "expires_at": (
                datetime.now(UTC) + timedelta(seconds=expires_in)
            ).isoformat(),
            "scope": data.get("scope"),
        }
        
        logger.info("Token refreshed successfully")
        return result
    
    def store_tokens(self, user_id: str, token_data: dict, expires_in: int) -> None:
        """Store tokens after initial authorization."""
        self._store[user_id] = {
            "access_token": token_data["access_token"],
            "refresh_token": token_data.get("refresh_token"),
            "token_type": token_data.get("token_type", "Bearer"),
            "expires_at": (
                datetime.now(UTC) + timedelta(seconds=expires_in)
            ).isoformat(),
            "scope": token_data.get("scope"),
        }
```

---

## API Keys: Header, Query Param, Bearer {#api-keys}

### Header-based API Keys (preferred)

```python
import httpx
from typing import Optional


class APIKeyClient:
    """
    Client for APIs that use header-based API key authentication.
    
    Common header names:
    - X-API-Key (most common)
    - X-Auth-Token
    - Api-Key
    - X-Access-Token
    
    Always use a header, never a query parameter in production —
    query params appear in server logs, access logs, browser history.
    """
    
    def __init__(
        self,
        base_url: str,
        api_key: str,
        header_name: str = "X-API-Key",
        key_prefix: Optional[str] = None,  # e.g. "Bearer " or "Token "
    ):
        header_value = f"{key_prefix}{api_key}" if key_prefix else api_key
        
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers={header_name: header_value},
            timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0),
        )
    
    async def get(self, path: str, **kwargs) -> dict:
        response = await self._client.get(path, **kwargs)
        response.raise_for_status()
        return response.json()
    
    async def post(self, path: str, json: dict, **kwargs) -> dict:
        response = await self._client.post(path, json=json, **kwargs)
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        await self._client.aclose()


# Examples for common APIs:

# FedEx API
fedex_client = APIKeyClient(
    base_url="https://apis.fedex.com",
    api_key="your_fedex_token",
    header_name="Authorization",
    key_prefix="Bearer ",
)

# Easypost
easypost_client = APIKeyClient(
    base_url="https://api.easypost.com/v2",
    api_key="EZT_your_key",
    header_name="Authorization",
    key_prefix="Bearer ",
)

# Generic API key in X-API-Key header
generic_client = APIKeyClient(
    base_url="https://api.yourvendor.com/v1",
    api_key="sk_live_abc123",
)
```

### Bearer Token (OAuth2 result or static token)

```python
# When you already have a token (from OAuth or static issuance):

def make_bearer_client(base_url: str, token: str) -> httpx.AsyncClient:
    """Bearer token in Authorization header — standard for OAuth2 tokens."""
    return httpx.AsyncClient(
        base_url=base_url,
        headers={"Authorization": f"Bearer {token}"},
        timeout=httpx.Timeout(connect=5.0, read=30.0),
    )


# Query param API key — avoid in production, but some legacy APIs require it
def make_query_key_client(base_url: str, api_key: str, param_name: str = "api_key") -> httpx.AsyncClient:
    """
    Query param API key — INSECURE, avoid when possible.
    The key appears in:
    - URL (visible in logs, proxy logs, browser history)
    - Referrer header (when user clicks a link)
    - Server access logs
    
    Some old APIs (here maps, openweathermap) still use this.
    """
    return httpx.AsyncClient(
        base_url=base_url,
        params={param_name: api_key},
        timeout=httpx.Timeout(connect=5.0, read=30.0),
    )
```

---

## HMAC-Signed Requests {#hmac-signed-requests}

### What HMAC signing is

Instead of just sending an API key, you use the key to sign the entire request (method + URL + headers + body). This provides:
1. **Authentication**: Only you know the key
2. **Integrity**: Tampering with the request invalidates the signature
3. **Replay prevention**: Include timestamp in signature

Used by: AWS (SigV4), Shopify (webhooks), Stripe (webhooks), some financial APIs, internal APIs that need integrity guarantees.

### Generic HMAC request signer

```python
import hashlib
import hmac
import json
import time
import uuid
from datetime import datetime, UTC
from typing import Optional
import httpx


class HMACRequestSigner:
    """
    Signs HTTP requests with HMAC-SHA256.
    
    Signature covers:
    - HTTP method (uppercase)
    - Request path (normalized)
    - Query string (sorted, normalized)
    - Timestamp (to prevent replay)
    - Request body hash
    
    This is a simplified version of AWS SigV4 — conceptually identical.
    """
    
    def __init__(self, api_key: str, api_secret: str):
        self.api_key = api_key
        self.api_secret = api_secret
    
    def sign(
        self,
        method: str,
        url: str,
        body: Optional[bytes] = None,
        extra_headers: Optional[dict] = None,
    ) -> dict:
        """
        Compute signature and return headers dict to add to request.
        
        Returns dict of headers that must be included in the signed request.
        """
        timestamp = str(int(time.time()))
        request_id = str(uuid.uuid4())
        body_bytes = body or b""
        
        # Hash the body
        body_hash = hashlib.sha256(body_bytes).hexdigest()
        
        # Parse URL components
        from urllib.parse import urlparse, urlencode, parse_qsl
        parsed = urlparse(url)
        
        # Normalize query string (sort parameters for canonical form)
        query_params = sorted(parse_qsl(parsed.query))
        canonical_query = urlencode(query_params)
        
        # Build the string to sign
        # Format: METHOD\nPATH\nQUERY\nTIMESTAMP\nBODY_HASH
        string_to_sign = "\n".join([
            method.upper(),
            parsed.path or "/",
            canonical_query,
            timestamp,
            body_hash,
        ])
        
        # Compute HMAC
        signature = hmac.new(
            self.api_secret.encode("utf-8"),
            string_to_sign.encode("utf-8"),
            hashlib.sha256,
        ).hexdigest()
        
        return {
            "X-API-Key": self.api_key,
            "X-Timestamp": timestamp,
            "X-Request-Id": request_id,
            "X-Signature": signature,
            "X-Body-Hash": body_hash,
        }


class HMACSignedClient:
    """
    HTTP client that signs every request with HMAC.
    """
    
    def __init__(self, base_url: str, api_key: str, api_secret: str):
        self.signer = HMACRequestSigner(api_key, api_secret)
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=httpx.Timeout(connect=5.0, read=30.0),
        )
    
    async def post(self, path: str, payload: dict) -> dict:
        body = json.dumps(payload, separators=(",", ":")).encode("utf-8")
        full_url = f"{self._client.base_url}{path}"
        
        signature_headers = self.signer.sign(
            method="POST",
            url=full_url,
            body=body,
        )
        
        response = await self._client.post(
            path,
            content=body,
            headers={
                **signature_headers,
                "Content-Type": "application/json",
            },
        )
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        await self._client.aclose()


# AWS SigV4 (the real one — use boto3 for actual AWS calls,
# but understanding it helps with SigV4-like APIs)
def sign_aws_request(
    method: str,
    url: str,
    region: str,
    service: str,
    access_key: str,
    secret_key: str,
    body: bytes = b"",
    additional_headers: Optional[dict] = None,
) -> dict:
    """
    AWS Signature Version 4.
    
    This is here for understanding — in practice use boto3 which handles this.
    Understanding SigV4 helps when:
    - Debugging auth failures
    - Building tools on top of AWS services without boto3
    - Working with services that use SigV4-style signing (some third parties)
    """
    import datetime as dt
    
    t = dt.datetime.utcnow()
    amzdate = t.strftime("%Y%m%dT%H%M%SZ")
    datestamp = t.strftime("%Y%m%d")
    
    # Parse URL
    from urllib.parse import urlparse, parse_qsl, urlencode
    parsed = urlparse(url)
    host = parsed.netloc
    canonical_uri = parsed.path or "/"
    
    # Canonical query string (sorted)
    query_params = sorted(parse_qsl(parsed.query))
    canonical_querystring = urlencode(query_params)
    
    # Canonical headers (sorted, lowercase, trimmed)
    headers = {
        "host": host,
        "x-amz-date": amzdate,
    }
    if additional_headers:
        headers.update({k.lower(): v.strip() for k, v in additional_headers.items()})
    
    sorted_header_keys = sorted(headers.keys())
    canonical_headers = "\n".join(f"{k}:{headers[k]}" for k in sorted_header_keys) + "\n"
    signed_headers = ";".join(sorted_header_keys)
    
    # Payload hash
    payload_hash = hashlib.sha256(body).hexdigest()
    
    # Canonical request
    canonical_request = "\n".join([
        method.upper(),
        canonical_uri,
        canonical_querystring,
        canonical_headers,
        signed_headers,
        payload_hash,
    ])
    
    # String to sign
    credential_scope = f"{datestamp}/{region}/{service}/aws4_request"
    string_to_sign = "\n".join([
        "AWS4-HMAC-SHA256",
        amzdate,
        credential_scope,
        hashlib.sha256(canonical_request.encode()).hexdigest(),
    ])
    
    # Signing key
    def _sign(key: bytes, msg: str) -> bytes:
        return hmac.new(key, msg.encode(), hashlib.sha256).digest()
    
    signing_key = _sign(
        _sign(
            _sign(
                _sign(f"AWS4{secret_key}".encode(), datestamp),
                region,
            ),
            service,
        ),
        "aws4_request",
    )
    
    signature = hmac.new(signing_key, string_to_sign.encode(), hashlib.sha256).hexdigest()
    
    # Authorization header
    auth_header = (
        f"AWS4-HMAC-SHA256 "
        f"Credential={access_key}/{credential_scope}, "
        f"SignedHeaders={signed_headers}, "
        f"Signature={signature}"
    )
    
    return {
        "Authorization": auth_header,
        "x-amz-date": amzdate,
        "x-amz-content-sha256": payload_hash,
    }
```

---

## Mutual TLS (mTLS) {#mtls}

### What mTLS is

Standard TLS: server proves its identity to client (SSL certificate you trust).
Mutual TLS: **both** sides prove identity. The client also presents a certificate.

```
Standard TLS:
  Client → Hello → Server
  Server → "Here's my certificate" → Client
  Client verifies server cert, encrypts session
  
Mutual TLS:
  Client → Hello + "Here's my certificate" → Server
  Server → "Here's my certificate" → Client
  BOTH verify each other's certificates
  Session only proceeds if both trust each other
```

### When customers require mTLS

- Financial services (PCI DSS environments)
- Government/federal systems (FedRAMP)
- Healthcare (HIPAA high-security zones)
- Banking/payment processing
- Internal APIs in zero-trust network architectures

### Certificate formats

```
PEM format: Base64-encoded, human-readable
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJALMBovqXR3N5MA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
... (base64 encoded DER) ...
-----END CERTIFICATE-----

Files you'll work with:
- client.crt or client.pem: client certificate (share with server so it can trust you)
- client.key or client-key.pem: your private key (NEVER share — keep secret)
- ca.crt or ca.pem: CA certificate (trust bundle to verify server's cert)

PKCS12 (.p12 / .pfx): Binary format bundling cert + private key. 
Often received from enterprise security teams. Convert with openssl:
openssl pkcs12 -in client.p12 -out client.pem -nokeys
openssl pkcs12 -in client.p12 -out client.key -nocerts -nodes
```

### Implementation with httpx

```python
import ssl
import httpx
from pathlib import Path
from typing import Optional, Union


def build_mtls_client(
    base_url: str,
    client_cert_path: Union[str, Path],
    client_key_path: Union[str, Path],
    ca_cert_path: Optional[Union[str, Path]] = None,
    client_key_password: Optional[str] = None,
    verify_server: bool = True,
) -> httpx.AsyncClient:
    """
    Build an httpx client that presents a client certificate (mTLS).
    
    Args:
        base_url: API base URL
        client_cert_path: Path to client certificate (.pem or .crt)
        client_key_path: Path to client private key (.pem or .key)
        ca_cert_path: Path to CA certificate bundle to verify server
                      If None: uses system default CA bundle
        client_key_password: Password for encrypted private key (if applicable)
        verify_server: Set False ONLY for development with self-signed certs
    
    Returns:
        Configured httpx.AsyncClient with mTLS
    """
    # Build SSL context
    if verify_server and ca_cert_path:
        # Verify server against custom CA (common in enterprise: internal CA)
        ssl_context = ssl.create_default_context(cafile=str(ca_cert_path))
    elif verify_server:
        # Verify server against system CA bundle (standard HTTPS)
        ssl_context = ssl.create_default_context()
    else:
        # Disable verification — DEVELOPMENT ONLY
        ssl_context = ssl.create_default_context()
        ssl_context.check_hostname = False
        ssl_context.verify_mode = ssl.CERT_NONE
    
    # Load client certificate and key
    ssl_context.load_cert_chain(
        certfile=str(client_cert_path),
        keyfile=str(client_key_path),
        password=client_key_password,
    )
    
    return httpx.AsyncClient(
        base_url=base_url,
        verify=ssl_context,
        timeout=httpx.Timeout(connect=10.0, read=30.0),
    )


# Loading certificates from AWS Secrets Manager (production pattern)
import boto3
import json
import tempfile
import os


async def build_mtls_client_from_secrets(
    base_url: str,
    secret_name: str,
    region: str = "us-east-1",
) -> httpx.AsyncClient:
    """
    Load certificates from AWS Secrets Manager and build mTLS client.
    
    Secret format (JSON):
    {
        "client_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
        "client_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----",
        "ca_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
    }
    
    Why Secrets Manager not files?
    - Certificates have expiry dates and need rotation
    - AWS Secrets Manager supports automatic rotation hooks
    - No certificate files sitting on disk that could be leaked
    - Centralized audit log of who accessed which certs
    """
    session = boto3.Session(region_name=region)
    sm_client = session.client("secretsmanager")
    
    response = sm_client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response["SecretString"])
    
    # Write to temp files (ssl module needs file paths)
    # tempfile.NamedTemporaryFile with delete=False so we can pass the path
    # Clean up after client is created
    cert_file = tempfile.NamedTemporaryFile(mode="w", suffix=".pem", delete=False)
    key_file = tempfile.NamedTemporaryFile(mode="w", suffix=".key", delete=False)
    ca_file = tempfile.NamedTemporaryFile(mode="w", suffix=".pem", delete=False)
    
    try:
        cert_file.write(secret["client_cert"])
        cert_file.close()
        
        key_file.write(secret["client_key"])
        key_file.close()
        
        ca_file.write(secret["ca_cert"])
        ca_file.close()
        
        client = build_mtls_client(
            base_url=base_url,
            client_cert_path=cert_file.name,
            client_key_path=key_file.name,
            ca_cert_path=ca_file.name,
        )
        
        return client
    finally:
        # Clean up temp files — they've been loaded into the SSL context
        for f in [cert_file.name, key_file.name, ca_file.name]:
            try:
                os.unlink(f)
            except OSError:
                pass


# Convenience function for loading from PEM strings directly
def build_mtls_client_from_pem_strings(
    base_url: str,
    client_cert_pem: str,
    client_key_pem: str,
    ca_cert_pem: Optional[str] = None,
) -> httpx.AsyncClient:
    """
    Build mTLS client from PEM strings (e.g., loaded from env vars or Vault).
    """
    import tempfile
    import os
    
    # ssl.SSLContext.load_cert_chain() needs file paths, not strings
    # Write to temp files with tight permissions
    with tempfile.NamedTemporaryFile(mode="w", suffix=".pem", delete=False) as f:
        f.write(client_cert_pem)
        cert_path = f.name
    
    with tempfile.NamedTemporaryFile(mode="w", suffix=".key", delete=False) as f:
        f.write(client_key_pem)
        key_path = f.name
    
    ca_path = None
    if ca_cert_pem:
        with tempfile.NamedTemporaryFile(mode="w", suffix=".pem", delete=False) as f:
            f.write(ca_cert_pem)
            ca_path = f.name
    
    try:
        client = build_mtls_client(
            base_url=base_url,
            client_cert_path=cert_path,
            client_key_path=key_path,
            ca_cert_path=ca_path,
        )
    finally:
        os.unlink(cert_path)
        os.unlink(key_path)
        if ca_path:
            os.unlink(ca_path)
    
    return client
```

### Certificate validation and debugging

```python
import ssl
import datetime


def inspect_certificate(cert_path: str) -> dict:
    """
    Inspect a PEM certificate file — useful for debugging mTLS issues.
    
    Common mTLS failures:
    1. Certificate expired
    2. Wrong CA — server doesn't trust your CA, or you don't trust theirs
    3. Certificate CN/SAN doesn't match hostname
    4. Private key doesn't match certificate (mismatched pair)
    """
    import subprocess
    
    result = subprocess.run(
        ["openssl", "x509", "-in", cert_path, "-text", "-noout"],
        capture_output=True, text=True
    )
    
    if result.returncode != 0:
        raise ValueError(f"Cannot parse certificate: {result.stderr}")
    
    # Parse expiry date
    expiry_result = subprocess.run(
        ["openssl", "x509", "-in", cert_path, "-enddate", "-noout"],
        capture_output=True, text=True
    )
    
    return {
        "text": result.stdout,
        "expiry_line": expiry_result.stdout.strip(),
    }


def verify_cert_key_match(cert_path: str, key_path: str) -> bool:
    """
    Verify that a certificate and private key are a matching pair.
    Common issue: getting mismatched cert/key from an ops team.
    """
    import subprocess
    
    # Get modulus of cert and key — they must match
    cert_result = subprocess.run(
        ["openssl", "x509", "-in", cert_path, "-noout", "-modulus"],
        capture_output=True, text=True
    )
    key_result = subprocess.run(
        ["openssl", "rsa", "-in", key_path, "-noout", "-modulus"],
        capture_output=True, text=True
    )
    
    cert_modulus = cert_result.stdout.strip()
    key_modulus = key_result.stdout.strip()
    
    return cert_modulus == key_modulus
```

---

## JWT Service-to-Service Auth {#jwt-service-to-service}

### When you use JWT assertions

Some APIs (particularly financial services and enterprise APIs) authenticate callers via JWT (JSON Web Token) assertions rather than client credentials. You:
1. Create a JWT signed with your private key
2. Send the JWT as your authentication credential
3. The server verifies the JWT using your public key (pre-registered with them)

Common systems: some OAuth2 flows (JWT Bearer Grant), internal microservice auth, Google APIs.

```python
import time
import uuid
import jwt  # PyJWT library: pip install PyJWT[cryptography]
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class JWTAssertionAuth:
    """
    JWT service-to-service authentication.
    
    You hold a private RSA or ECDSA key.
    The server has your public key (you registered it with them).
    You sign a JWT, the server verifies it — no secret ever travels over the wire.
    
    This is more secure than client_secret because:
    - Private key never leaves your infrastructure
    - Each assertion is time-limited (exp claim)
    - Can't be replayed after expiry
    """
    
    def __init__(
        self,
        private_key_pem: str,
        client_id: str,
        audience: str,  # the API/token endpoint you're authenticating to
        algorithm: str = "RS256",  # RS256 (RSA) or ES256 (ECDSA)
        token_lifetime_seconds: int = 300,  # 5 minutes
    ):
        self.client_id = client_id
        self.audience = audience
        self.algorithm = algorithm
        self.token_lifetime = token_lifetime_seconds
        
        # Load private key
        if algorithm.startswith("RS"):
            self._private_key = serialization.load_pem_private_key(
                private_key_pem.encode(),
                password=None,
                backend=default_backend(),
            )
        else:
            self._private_key = private_key_pem  # EC key loaded differently
    
    def create_assertion(self, subject: Optional[str] = None) -> str:
        """
        Create a signed JWT assertion.
        
        Standard JWT claims:
        - iss (issuer): who created the JWT — usually your client_id
        - sub (subject): who the JWT is about — often same as iss for service-to-service
        - aud (audience): who the JWT is for — the API endpoint or token URL
        - iat (issued at): when it was created
        - exp (expiry): when it expires — REQUIRED, short-lived
        - jti (JWT ID): unique ID for this JWT — prevents replay
        """
        now = int(time.time())
        
        claims = {
            "iss": self.client_id,
            "sub": subject or self.client_id,
            "aud": self.audience,
            "iat": now,
            "exp": now + self.token_lifetime,
            "jti": str(uuid.uuid4()),
        }
        
        token = jwt.encode(
            payload=claims,
            key=self._private_key,
            algorithm=self.algorithm,
        )
        
        logger.debug(
            "Created JWT assertion",
            extra={
                "client_id": self.client_id,
                "audience": self.audience,
                "jti": claims["jti"],
                "expires_in": self.token_lifetime,
            }
        )
        
        return token
    
    async def get_oauth_token(
        self,
        token_url: str,
        scope: Optional[str] = None,
    ) -> dict:
        """
        Use JWT assertion to get an OAuth2 access token.
        
        RFC 7523: JWT Bearer Token Grant
        grant_type = "urn:ietf:params:oauth:grant-type:jwt-bearer"
        assertion = <your signed JWT>
        """
        assertion = self.create_assertion()
        
        data = {
            "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
            "assertion": assertion,
        }
        if scope:
            data["scope"] = scope
        
        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                token_url,
                data=data,
                headers={"Accept": "application/json"},
            )
            response.raise_for_status()
            return response.json()


def decode_jwt_for_debugging(token: str, public_key: Optional[str] = None) -> dict:
    """
    Decode a JWT for debugging. Shows claims without verification if no key provided.
    
    Use this to inspect:
    - Is the token expired? (exp claim)
    - What audience is it for? (aud claim)
    - What scopes? (scope claim, if present)
    """
    if public_key:
        # Verified decode
        return jwt.decode(
            token,
            public_key,
            algorithms=["RS256", "RS384", "RS512", "ES256", "ES384"],
        )
    else:
        # Unverified decode — for debugging only
        return jwt.decode(token, options={"verify_signature": False})


# Google Service Account auth (real-world JWT assertion example)
def google_service_account_jwt(
    service_account_key: dict,  # parsed JSON from service account key file
    target_audience: str,  # the API endpoint being called
) -> str:
    """
    Create a Google service account JWT.
    Used for Google APIs that support service account authentication.
    
    service_account_key: contents of the .json downloaded from GCP console
    """
    now = int(time.time())
    
    claims = {
        "iss": service_account_key["client_email"],
        "sub": service_account_key["client_email"],
        "aud": target_audience,
        "iat": now,
        "exp": now + 3600,
    }
    
    private_key = service_account_key["private_key"]
    
    return jwt.encode(
        payload=claims,
        key=private_key,
        algorithm="RS256",
        headers={"kid": service_account_key["private_key_id"]},
    )
```

---

## Combining Auth Patterns {#combining-auth}

Real integrations often require multiple auth mechanisms simultaneously:

```python
class MultiLayerAuth:
    """
    Real-world example: an enterprise integration that needs:
    1. mTLS for the transport layer (required by compliance team)
    2. OAuth2 client credentials for the API's application layer auth
    3. HMAC signature on individual sensitive operations
    
    This is realistic for financial services or government APIs.
    """
    
    def __init__(
        self,
        # mTLS config
        client_cert_pem: str,
        client_key_pem: str,
        ca_cert_pem: str,
        # OAuth2 config
        token_url: str,
        client_id: str,
        client_secret: str,
        scope: str,
        # HMAC config
        hmac_key_id: str,
        hmac_secret: str,
        # API config
        base_url: str,
    ):
        # Layer 1: mTLS client
        self._mtls_client = build_mtls_client_from_pem_strings(
            base_url=base_url,
            client_cert_pem=client_cert_pem,
            client_key_pem=client_key_pem,
            ca_cert_pem=ca_cert_pem,
        )
        
        # Layer 2: OAuth2 token manager
        self._oauth = OAuth2ClientCredentials(
            token_url=token_url,
            client_id=client_id,
            client_secret=client_secret,
            scope=scope,
        )
        # Note: the OAuth token request also goes over the mTLS client
        
        # Layer 3: HMAC signer
        self._signer = HMACRequestSigner(hmac_key_id, hmac_secret)
    
    async def secure_post(self, path: str, payload: dict) -> dict:
        """
        POST that uses all three auth layers.
        """
        # Get OAuth2 token
        token = await self._oauth.get_token()
        
        # Prepare body
        body = json.dumps(payload, separators=(",", ":")).encode()
        
        # HMAC sign
        signature_headers = self._signer.sign("POST", path, body)
        
        # Make request over mTLS channel with Bearer token + HMAC headers
        response = await self._mtls_client.post(
            path,
            content=body,
            headers={
                "Authorization": token.authorization_header,
                "Content-Type": "application/json",
                **signature_headers,
            },
        )
        response.raise_for_status()
        return response.json()
```

---

## Secrets Management {#secrets-management}

### Never hardcode credentials — load from environment or secrets manager

```python
import os
import json
import boto3
import logging
from functools import lru_cache
from typing import Optional

logger = logging.getLogger(__name__)


@lru_cache(maxsize=None)
def get_secret(secret_name: str, region: str = "us-east-1") -> dict:
    """
    Fetch a secret from AWS Secrets Manager.
    lru_cache ensures we only call the API once per process lifetime.
    
    In Lambda: cache persists across warm invocations (significant cost saving).
    In long-running service: cache invalidates on process restart.
    For frequently-rotated secrets: don't use lru_cache — check TTL instead.
    """
    client = boto3.client("secretsmanager", region_name=region)
    
    response = client.get_secret_value(SecretId=secret_name)
    
    if "SecretString" in response:
        return json.loads(response["SecretString"])
    else:
        import base64
        return {"binary": base64.b64decode(response["SecretBinary"])}


def load_carrier_credentials() -> dict:
    """
    Load carrier API credentials from appropriate source based on environment.
    
    Development: environment variables
    Production: AWS Secrets Manager
    
    This pattern makes local development easy while keeping production secure.
    """
    env = os.getenv("APP_ENV", "development")
    
    if env == "production":
        return get_secret("prod/carrier-api/credentials")
    elif env == "staging":
        return get_secret("staging/carrier-api/credentials")
    else:
        # Local development: load from env vars
        return {
            "api_key": os.environ["CARRIER_API_KEY"],
            "api_secret": os.environ.get("CARRIER_API_SECRET", ""),
            "client_id": os.environ.get("CARRIER_CLIENT_ID", ""),
            "client_secret": os.environ.get("CARRIER_CLIENT_SECRET", ""),
        }


# Complete setup function — puts everything together
async def setup_carrier_client() -> OAuth2HTTPClient:
    """
    Production-grade carrier client setup.
    Loads credentials from Secrets Manager, builds auth layer, returns ready client.
    """
    creds = load_carrier_credentials()
    
    auth = OAuth2ClientCredentials(
        token_url="https://auth.carrier.com/oauth2/token",
        client_id=creds["client_id"],
        client_secret=creds["client_secret"],
        scope="rates:read shipments:write tracking:read",
    )
    
    return OAuth2HTTPClient(
        auth=auth,
        base_url="https://api.carrier.com/v2",
    )
```

---

## Common Failure Modes {#common-failure-modes}

### 1. Token not cached — re-fetching on every request

**Symptom**: Auth server rate limits you, or latency doubles.

**Diagnosis**: Add logging to your `_fetch_token` method — if it fires more than once every ~50 minutes, you're not caching.

**Fix**: Implement `OAuth2ClientCredentials` with `asyncio.Lock` as shown above.

### 2. Race condition on token refresh

**Symptom**: Occasional 401 errors followed by immediate success on retry, especially under load.

**Root cause**: Two coroutines both see token as expired, both try to refresh. One's token gets overwritten.

**Fix**: Use `asyncio.Lock` around refresh logic with double-check pattern.

### 3. Clock skew in JWT

**Symptom**: JWT assertions rejected with "token not yet valid" or "token expired" even though it looks fine.

**Root cause**: Your server's clock is ahead of the auth server's clock. `iat` claim appears to be in the future.

**Fix**: Add `nbf` (not before) tolerance; use NTP sync; subtract 30 seconds from `iat`.

### 4. mTLS "certificate verify failed"

**Symptom**: SSL handshake fails.

**Root cause options**:
- Wrong CA cert (you're verifying against wrong authority)
- Certificate expired
- Hostname mismatch (CN doesn't match API hostname)
- Wrong cert/key pair (loaded mismatched files)

**Diagnosis**:
```bash
# Test mTLS connection directly with openssl
openssl s_client -connect api.example.com:443 \
  -cert client.crt \
  -key client.key \
  -CAfile ca.crt \
  -verify_return_error

# Check certificate expiry
openssl x509 -in client.crt -noout -enddate

# Verify cert/key match (modulus must match)
openssl x509 -in client.crt -noout -modulus | openssl md5
openssl rsa -in client.key -noout -modulus | openssl md5
```

### 5. OAuth2 "invalid_client" error

**Symptom**: `{"error": "invalid_client"}` on token request.

**Root causes**:
- Wrong client_id or client_secret
- client_secret needs URL encoding but isn't
- Wrong token endpoint URL (common: `/oauth/token` vs `/oauth2/token` vs `/token`)
- Client credentials must be in Basic Auth header, not form body (some servers)

**Fix for Basic Auth requirement**:
```python
import base64

# Some servers require client_id:client_secret as Basic Auth
credentials = base64.b64encode(f"{client_id}:{client_secret}".encode()).decode()
headers = {"Authorization": f"Basic {credentials}"}

response = await client.post(
    token_url,
    data={"grant_type": "client_credentials", "scope": scope},
    headers=headers,  # credentials in header, NOT in body
)
```

### 6. HMAC signature mismatch

**Symptom**: API returns 401 or 403 "signature invalid" even though code looks correct.

**Checklist**:
1. Are you encoding the body consistently? `json.dumps(payload)` vs `json.dumps(payload, separators=(",", ":"))` produce different strings
2. Are query parameters in the same order as the server expects?
3. Is the timestamp format correct? Seconds vs milliseconds?
4. Is the key encoded as UTF-8?
5. Hex string vs Base64? Check the API docs — both are common.

**Debugging**:
```python
# Add this to your signer to debug
def sign_debug(self, method, url, body):
    string_to_sign = self._build_string_to_sign(method, url, body)
    print(f"STRING TO SIGN:\n{repr(string_to_sign)}\n")  # repr shows escape chars
    signature = self._compute_signature(string_to_sign)
    print(f"SIGNATURE: {signature}")
    return signature
```

---

## Interview Angles {#interview-angles}

### Q: "Walk me through OAuth2 client credentials flow from scratch."

**Great answer**: "Client credentials is the service-to-service flow. My backend makes a POST to the authorization server's token endpoint with grant_type=client_credentials plus my client_id and client_secret. The server returns an access token with an expiry. I cache that token and inject it as a Bearer token in subsequent API calls. When the token expires — I buffer by 60 seconds to avoid using nearly-expired tokens — I fetch a new one. The key implementation detail is using an asyncio.Lock so that if 50 concurrent requests all see the token expired at the same time, only one actually makes the refresh call — the others wait and use the newly-fetched token. Without this lock you get a thundering herd of token refresh requests."

### Q: "A customer says their carrier API requires mTLS. What does that mean and how do you implement it?"

**Great answer**: "mTLS means mutual TLS — both sides present certificates. Standard HTTPS only has the server prove its identity to the client. With mTLS, the client also presents a certificate that the server validates. In practice, the carrier will give us a client certificate and private key — usually PEM format. We load those into our HTTP client's SSL context using ssl.create_default_context() and load_cert_chain(). The tricky parts are: storing the private key securely — use AWS Secrets Manager or Vault, not environment variables — and handling certificate rotation when the cert expires. I'd also keep the CA cert they send so we can verify their server cert if they use an internal CA."

### Q: "How do you prevent an API key from appearing in your application logs?"

**Great answer**: "Several layers. First, the key is injected as a header — never in a query parameter where it'd appear in request URLs in logs. Second, I configure the HTTP client to not log the full request headers, or if I do log headers, I mask the key value with a pattern like 'sk_live_***'. Third, structured logging keys like 'api_key' should be in a deny-list for any log aggregation system. Fourth, in error handling, I never log the raw request object without filtering credentials out. The key insight is defense in depth — assume some layer will fail, design each layer to minimize the blast radius."

### Q: "What's the difference between OAuth2 Authorization Code and Client Credentials flows?"

**Great answer**: "Authorization Code is for when a human user needs to authorize your application to act on their behalf — they get redirected to a login page, grant consent, and you get tokens representing their authorization. Client Credentials is purely machine-to-machine — no human in the loop. Your service authenticates as itself. In logistics integrations, I almost always use Client Credentials — I'm integrating with a carrier API as our platform, not on behalf of individual users. Authorization Code comes up when a merchant is connecting their own account to our platform — like 'click here to connect your Shopify store.' The token in that case represents the merchant's authorization of our app."

---

## Practice Exercise {#practice-exercise}

### Exercise: Multi-Auth Integration Client

Build a client that can authenticate using any of the four main mechanisms, chosen at runtime.

**Scenario**: You're building a freight broker integration that might connect to carriers using different auth methods:
- **Carrier A**: OAuth2 Client Credentials with Azure AD
- **Carrier B**: API Key in X-API-Key header
- **Carrier C**: HMAC-signed requests
- **Carrier D**: mTLS with custom CA

**Requirements**:

1. **`AuthFactory`** (`auth_factory.py`):
   - `create_client(carrier_config: dict) -> httpx.AsyncClient`
   - `carrier_config` has `auth_type` field: `"oauth2_client_credentials"`, `"api_key"`, `"hmac"`, `"mtls"`
   - Each type reads appropriate credentials from environment variables
   - Factory pattern — callers don't need to know which auth is being used

2. **`TokenManager`** (`token_manager.py`):
   - Manages OAuth2 client credentials tokens for multiple carriers simultaneously
   - Each carrier has its own token with its own expiry
   - Thread-safe: using asyncio.Lock per carrier
   - Test: start 100 concurrent requests to a carrier — verify only 1 token fetch occurs

3. **HMAC signer test** (`test_hmac.py`):
   - Sign a test payload with a known key and secret
   - Verify the signature against a pre-computed expected value
   - Test that body modification changes the signature
   - Test that timestamp modification changes the signature

**Skeleton**:

```python
# auth_factory.py
from dataclasses import dataclass
from typing import Literal
import httpx

@dataclass
class CarrierConfig:
    carrier_id: str
    base_url: str
    auth_type: Literal["oauth2_client_credentials", "api_key", "hmac", "mtls"]
    credentials: dict  # type-specific credentials

async def create_auth_client(config: CarrierConfig) -> httpx.AsyncClient:
    """
    TODO: implement factory that returns appropriate client based on config.auth_type
    Each client type must:
    - Authenticate correctly
    - Inject auth on every request
    - Handle token refresh (for OAuth2)
    """
    ...
```

**Evaluation criteria**:
- Does OAuth2 implementation cache tokens correctly? (1 fetch per ~hour)
- Does HMAC signer use `hmac.compare_digest` when verifying?
- Are private keys/secrets loaded from environment, not hardcoded?
- Does mTLS implementation handle encrypted private keys (with password)?
- Can the factory handle an unknown `auth_type` gracefully (raises `ValueError`, not `KeyError`)?
- Does the token manager correctly serialize concurrent refresh calls with a lock?

---

## Related Files

- [REST, GraphQL, and Webhooks](01-rest-graphql-webhooks.md) — HTTP client fundamentals; this file adds auth layer on top
- [Resilience patterns](03-resilience-patterns.md) — retrying 401s, circuit breaking auth servers
- [Legacy connectivity (SFTP)](05-legacy-connectivity.md) — SFTP key-based auth follows similar principles
- [Message queues](06-message-queues.md) — AWS SQS/SNS use IAM authentication (SigV4 under the hood via boto3)
