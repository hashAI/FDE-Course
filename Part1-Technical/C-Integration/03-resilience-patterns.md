# Resilience Patterns — Retry, Idempotency, Rate Limiting, Circuit Breakers

**FDE Course | Part 1: Technical | Section C: Integration**

> **FDE Principle**: Integrations fail. The question is not "will this fail?" but "when it fails, does it fail gracefully, does it heal automatically, and can I diagnose it from logs alone at 3am?" Every pattern in this file directly maps to a category of production incident you will face.

---

## Table of Contents

1. [The Resilience Stack](#resilience-stack)
2. [Retries: When To, When Not To](#retries)
3. [Tenacity Deep Dive](#tenacity-deep-dive)
4. [Exponential Backoff and Jitter](#backoff-jitter)
5. [Idempotency](#idempotency)
6. [Rate Limiting](#rate-limiting)
7. [Circuit Breakers](#circuit-breakers)
8. [Timeouts: All Three Kinds](#timeouts)
9. [Dead Letter Patterns](#dead-letter-patterns)
10. [The 2am API Ban Post-Mortem](#the-2am-incident)
11. [Composing Resilience Patterns](#composing-patterns)
12. [Common Failure Modes](#common-failure-modes)
13. [Interview Angles](#interview-angles)
14. [Practice Exercise](#practice-exercise)

---

## The Resilience Stack {#resilience-stack}

Think of resilience as layers, each protecting against a different failure:

```
Request path (outside → in):
────────────────────────────────────────────────────────
1. CIRCUIT BREAKER — stops calls to failed dependency entirely
2. RATE LIMITER — prevents you from exceeding vendor quotas
3. TIMEOUT — prevents hanging indefinitely
4. RETRY — recovers from transient failures
5. IDEMPOTENCY — ensures retries don't cause side effects
────────────────────────────────────────────────────────

Reading order matters:
- Circuit breaker OPENS → no requests reach rate limiter or network
- Rate limiter BLOCKS → request waits, doesn't go to network
- Timeout FIRES → exception raised, retry logic catches it
- Retry EXHAUSTED → dead letter queue catches permanently failed work
- Idempotency → makes all of the above safe to combine
```

All five must be in place before an integration is production-ready. Missing even one creates an incident waiting to happen.

---

## Retries: When To, When Not To {#retries}

### The decision tree

```
Should I retry?
│
├─ Is this a 4xx error?
│   ├─ 400, 422: YOUR payload is invalid → DO NOT RETRY (fix the data)
│   ├─ 401: Token expired → refresh token once, then retry ONCE
│   ├─ 403: Permission denied → DO NOT RETRY (escalate)
│   ├─ 404: Not found → DO NOT RETRY (resource doesn't exist)
│   ├─ 409: Conflict → DEPENDS (may be idempotent duplicate — treat as success)
│   └─ 429: Rate limited → RETRY with Retry-After delay
│
├─ Is this a 5xx error?
│   ├─ 500: Internal server error → RETRY (may be transient)
│   ├─ 502: Bad gateway → RETRY (upstream dependency issue)
│   ├─ 503: Service unavailable → RETRY with backoff
│   └─ 504: Gateway timeout → RETRY with backoff
│
├─ Is this a network/transport error?
│   ├─ Connection refused → RETRY (server may be restarting)
│   ├─ DNS failure → RETRY (with longer backoff)
│   ├─ Read timeout → DEPENDS on idempotency
│   └─ Write timeout → DO NOT RETRY (request may have been received)
│
└─ Is the operation idempotent?
    ├─ YES (GET, PUT, DELETE, PATCH for most resources): RETRY freely
    └─ NO (POST creating a resource): Only retry if idempotency key is present
```

### Idempotent vs non-idempotent: the critical distinction

```python
# These are SAFE to retry unconditionally:
GET /shipments/SHIP-001           # Always safe — reading
PUT /inventory/SKU-001            # Safe — sets absolute value
DELETE /shipments/SHIP-001        # Safe — second delete returns 404 or 200
PATCH /order/123 {"status":"cancelled"}  # Usually safe — cancelling an already-cancelled order

# These are ONLY safe to retry with an idempotency key:
POST /shipments                   # Creates a new resource each time without idempotency key
POST /invoices                    # Same issue
POST /payments                    # Critical — double payment is a real problem

# NEVER retry these (not idempotent, potential serious consequences):
POST /shipments/SHIP-001/cancel   # If cancelled+refunded, cancelling again may cause issues
POST /auth/login                  # May lock account after N failures
```

---

## Tenacity Deep Dive {#tenacity-deep-dive}

### Installation

```bash
pip install tenacity
```

### Core API

```python
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    stop_never,
    wait_fixed,
    wait_random,
    wait_exponential,
    wait_exponential_jitter,
    wait_combine,
    wait_none,
    retry_if_exception_type,
    retry_if_exception,
    retry_if_result,
    retry_unless_exception_type,
    before_log,
    after_log,
    before_sleep_log,
    RetryError,
    TryAgain,
)
import logging
import httpx

logger = logging.getLogger(__name__)


# ─── Basic patterns ──────────────────────────────────────────

# Simple: retry 3 times on any exception
@retry(stop=stop_after_attempt(3))
def simple_retry_example():
    pass


# Retry with fixed wait
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),  # wait exactly 2 seconds between attempts
)
def fixed_wait_example():
    pass


# Retry only on specific exception types
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type(httpx.TransportError),
)
def transport_error_retry():
    pass


# Retry with condition on exception
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=60, jitter=2),
    retry=retry_if_exception(lambda e: (
        isinstance(e, httpx.HTTPStatusError) 
        and e.response.status_code in {429, 500, 502, 503, 504}
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
def smart_http_retry(client: httpx.Client, url: str) -> dict:
    response = client.get(url)
    response.raise_for_status()
    return response.json()


# ─── Advanced patterns ───────────────────────────────────────

# Retry on result (not exception) — e.g., polling for job completion
@retry(
    stop=stop_after_delay(300),  # Give up after 5 minutes total
    wait=wait_exponential(multiplier=2, min=5, max=60),
    retry=retry_if_result(lambda result: result.get("status") == "pending"),
    before_sleep=before_sleep_log(logger, logging.INFO),
)
def poll_job_status(client: httpx.Client, job_id: str) -> dict:
    """
    Poll a job until it's no longer 'pending'.
    Retries whenever the result is still pending, waits up to 5 minutes total.
    """
    response = client.get(f"/jobs/{job_id}")
    response.raise_for_status()
    return response.json()


# Combined stop conditions (first one to trigger wins)
@retry(
    stop=stop_after_attempt(10) | stop_after_delay(120),
    wait=wait_exponential_jitter(initial=1, max=30),
    reraise=True,
)
def retry_with_max_time_and_attempts():
    """Gives up after 10 attempts OR 2 minutes, whichever comes first."""
    pass


# Callback on each retry for structured logging
def log_retry_attempt(retry_state):
    """Custom callback for structured logging on retry."""
    logger.warning(
        "Retrying failed operation",
        extra={
            "attempt": retry_state.attempt_number,
            "fn_name": retry_state.fn.__name__,
            "elapsed_seconds": retry_state.outcome_timestamp - retry_state.start_time,
            "exception": str(retry_state.outcome.exception()) if retry_state.outcome else None,
            "next_wait": retry_state.next_action.sleep if retry_state.next_action else None,
        }
    )


@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=60),
    before_sleep=log_retry_attempt,
    reraise=True,
)
def operation_with_detailed_logging():
    pass


# ─── Async retry ─────────────────────────────────────────────

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=30),
    retry=retry_if_exception(lambda e: (
        isinstance(e, httpx.HTTPStatusError)
        and e.response.status_code in {429, 500, 502, 503, 504}
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def async_fetch_with_retry(client: httpx.AsyncClient, url: str) -> dict:
    """
    Tenacity works with async functions identically to sync.
    The @retry decorator detects async and uses asyncio.sleep instead of time.sleep.
    """
    response = await client.get(url)
    response.raise_for_status()
    return response.json()


# ─── Per-operation retry config ──────────────────────────────

# Define retry configs as objects for reuse
from tenacity import Retrying, AsyncRetrying

# Create reusable retry configs
CONSERVATIVE_RETRY = dict(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=30),
    retry=retry_if_exception_type((httpx.TransportError, httpx.TimeoutException)),
    reraise=True,
)

AGGRESSIVE_RETRY = dict(
    stop=stop_after_attempt(10),
    wait=wait_exponential_jitter(initial=0.5, max=60),
    retry=retry_if_exception(lambda e: (
        isinstance(e, (httpx.TransportError, httpx.TimeoutException)) or
        (isinstance(e, httpx.HTTPStatusError) and e.response.status_code >= 500)
    )),
    reraise=True,
)

# Use as context manager (more flexible than decorator)
async def fetch_with_configurable_retry(client, url, retry_config=None):
    retry_config = retry_config or CONSERVATIVE_RETRY
    
    async for attempt in AsyncRetrying(**retry_config):
        with attempt:
            response = await client.get(url)
            response.raise_for_status()
            return response.json()
```

---

## Exponential Backoff and Jitter {#backoff-jitter}

### Why jitter is not optional

Imagine 500 customers all making API calls that fail at 14:00:00. Without jitter:

```
14:00:01 — all 500 retry at the exact same time → server still overwhelmed
14:00:03 — all 500 retry again at the exact same time → still overwhelmed
14:00:07 — all 500 retry again → still overwhelmed
```

With jitter (each waits `backoff + random(0, backoff * 0.5)`):

```
14:00:01 to 14:00:02.5 — retries spread across 1.5 second window
14:00:03 to 14:00:06.5 — retries spread across 3.5 second window
Server load is 1/3 of the no-jitter case — it can recover
```

### The four jitter strategies

```python
import random
import math
from typing import Callable


def wait_exponential_no_jitter(attempt: int, base: float = 1.0, max_wait: float = 60.0) -> float:
    """Pure exponential: 1, 2, 4, 8, 16, 32, 60, 60..."""
    return min(base * (2 ** attempt), max_wait)


def wait_exponential_full_jitter(attempt: int, base: float = 1.0, max_wait: float = 60.0) -> float:
    """
    Full jitter: random between 0 and cap.
    Best for reducing contention — highest spread.
    AWS recommend this for SQS retry.
    """
    cap = min(base * (2 ** attempt), max_wait)
    return random.uniform(0, cap)


def wait_exponential_equal_jitter(attempt: int, base: float = 1.0, max_wait: float = 60.0) -> float:
    """
    Equal jitter: base exponential + random(0, base/2).
    Balance between spread and progress.
    """
    cap = min(base * (2 ** attempt), max_wait)
    return cap / 2 + random.uniform(0, cap / 2)


def wait_decorrelated_jitter(attempt: int, base: float = 1.0, max_wait: float = 60.0) -> float:
    """
    Decorrelated jitter: each wait is random(base, previous_wait * 3).
    Actually converges faster than equal jitter on average.
    """
    # Note: this is stateful — needs to track previous wait
    # Implementation shown for understanding, use tenacity's version in practice
    sleep = base
    for _ in range(attempt):
        sleep = min(random.uniform(base, sleep * 3), max_wait)
    return sleep


# Practical demonstration: which strategy under load

def simulate_retry_load(num_clients: int, strategy_fn: Callable, attempts: int = 5):
    """
    Simulate N clients all failing at t=0 and retrying.
    Shows how concentrated the retry load is.
    """
    from collections import Counter
    
    all_retry_times = []
    
    for _ in range(num_clients):
        t = 0
        for attempt in range(attempts):
            wait = strategy_fn(attempt)
            t += wait
            all_retry_times.append(int(t))  # bucket by second
    
    # Count retries per second
    counter = Counter(all_retry_times)
    
    # Return max retries in any single second (lower = better)
    return max(counter.values()) if counter else 0


# In practice, tenacity's wait_exponential_jitter is equivalent to equal jitter
# and is sufficient for most use cases
from tenacity import wait_exponential_jitter

# This is what you use in production:
wait = wait_exponential_jitter(
    initial=1,    # first wait: 1s base + jitter
    max=60,       # never wait more than 60s
    jitter=2,     # add up to 2s of random jitter
    exp_base=2,   # double each time
)
```

---

## Idempotency {#idempotency}

### Precise definition

An operation is idempotent if calling it once produces the same result as calling it N times. The system state after one call equals the system state after N calls.

```
GET /orders/123           → idempotent (reading doesn't change state)
PUT /inventory {qty: 10}  → idempotent (setting to 10 ten times = setting to 10 once)
POST /orders              → NOT idempotent (creates a new order each time)
DELETE /orders/123        → idempotent (deleting an already-deleted order = no change)
POST /order/123/ship      → depends on implementation
```

### Idempotency keys: client-generated UUID

```python
import uuid
import hashlib
import json
import sqlite3
import logging
from datetime import datetime, UTC, timedelta
from typing import Optional, Any

logger = logging.getLogger(__name__)


class IdempotencyKey:
    """
    Generate deterministic idempotency keys.
    
    Two strategies:
    1. Random UUID: Generate once, store with the operation record, reuse on retry
    2. Deterministic hash: Hash of the business key, survives process restarts
    
    Deterministic is better for integrations:
    - If your process crashes between "send request" and "record response",
      a random UUID would create a new key on restart, bypassing idempotency.
    - A deterministic key based on business data generates the same key on restart.
    """
    
    @staticmethod
    def random() -> str:
        """
        Random UUID — use when you don't have a stable business key.
        Must be stored and reused on retry.
        """
        return str(uuid.uuid4())
    
    @staticmethod
    def from_business_key(*args: Any) -> str:
        """
        Deterministic key from business data.
        Same inputs always produce the same key.
        
        Example:
        IdempotencyKey.from_business_key("ORD-123", "FEDEX", "2024-01-15")
        → always returns the same 32-char hex string
        """
        content = "|".join(str(a) for a in args)
        return hashlib.sha256(content.encode()).hexdigest()[:32]
    
    @staticmethod
    def from_payload(payload: dict, *extra_fields: str) -> str:
        """
        Deterministic key from a specific subset of payload fields.
        Useful when the payload contains the business key fields.
        """
        key_parts = []
        for field in extra_fields:
            value = payload.get(field, "")
            key_parts.append(str(value))
        
        content = "|".join(key_parts)
        return hashlib.sha256(content.encode()).hexdigest()[:32]


class ServerSideIdempotencyStore:
    """
    Server-side deduplication store.
    Used when YOU are the server receiving requests that might be retried.
    
    When a client sends an idempotency key:
    1. Check if we've seen this key before
    2. If yes: return the stored response (don't reprocess)
    3. If no: process the request, store result with key
    
    This is what Stripe does for payment APIs.
    """
    
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._create_table()
    
    def _create_table(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS idempotency_records (
                key TEXT PRIMARY KEY,
                response_status INTEGER NOT NULL,
                response_body TEXT NOT NULL,
                created_at TEXT NOT NULL,
                expires_at TEXT NOT NULL
            )
        """)
        self.conn.commit()
    
    def get_stored_response(self, key: str) -> Optional[dict]:
        """Returns stored response if key exists and hasn't expired."""
        cursor = self.conn.execute(
            "SELECT response_status, response_body FROM idempotency_records "
            "WHERE key = ? AND expires_at > ?",
            (key, datetime.now(UTC).isoformat()),
        )
        row = cursor.fetchone()
        if row:
            return {
                "status_code": row[0],
                "body": json.loads(row[1]),
            }
        return None
    
    def store_response(
        self,
        key: str,
        status_code: int,
        response_body: dict,
        ttl_hours: int = 24,
    ) -> None:
        """Store a response for later deduplication."""
        expires_at = (datetime.now(UTC) + timedelta(hours=ttl_hours)).isoformat()
        
        self.conn.execute(
            """
            INSERT OR REPLACE INTO idempotency_records
            (key, response_status, response_body, created_at, expires_at)
            VALUES (?, ?, ?, ?, ?)
            """,
            (
                key,
                status_code,
                json.dumps(response_body),
                datetime.now(UTC).isoformat(),
                expires_at,
            )
        )
        self.conn.commit()


# FastAPI endpoint with server-side idempotency
from fastapi import FastAPI, Request, Header, HTTPException
from fastapi.responses import JSONResponse
import asyncio

app = FastAPI()
idempotency_store = ServerSideIdempotencyStore()
_processing_locks: dict[str, asyncio.Lock] = {}


@app.post("/shipments")
async def create_shipment(
    request: Request,
    idempotency_key: Optional[str] = Header(None, alias="Idempotency-Key"),
) -> JSONResponse:
    """
    Shipment creation endpoint with idempotency.
    
    If same Idempotency-Key is sent twice:
    - First time: process and return result
    - Second time: return SAME result without reprocessing
    
    This makes retries safe — carrier won't be called twice.
    """
    body = await request.json()
    
    if idempotency_key:
        # Check for existing response
        stored = idempotency_store.get_stored_response(idempotency_key)
        if stored:
            logger.info(
                "Idempotent request — returning stored response",
                extra={"idempotency_key": idempotency_key},
            )
            return JSONResponse(
                content=stored["body"],
                status_code=stored["status_code"],
                headers={"X-Idempotency-Replayed": "true"},
            )
        
        # Acquire lock for this key to prevent concurrent processing of same key
        if idempotency_key not in _processing_locks:
            _processing_locks[idempotency_key] = asyncio.Lock()
        
        async with _processing_locks[idempotency_key]:
            # Double-check after acquiring lock
            stored = idempotency_store.get_stored_response(idempotency_key)
            if stored:
                return JSONResponse(content=stored["body"], status_code=stored["status_code"])
            
            # Process the request
            result = await _do_create_shipment(body)
            
            # Store result for future duplicate requests
            idempotency_store.store_response(
                key=idempotency_key,
                status_code=201,
                response_body=result,
                ttl_hours=24,
            )
            
            return JSONResponse(content=result, status_code=201)
    else:
        # No idempotency key — process without deduplication
        # Log a warning — idempotency key should be required for POST
        logger.warning(
            "Shipment creation without idempotency key",
            extra={"client_ip": request.client.host if request.client else "unknown"},
        )
        result = await _do_create_shipment(body)
        return JSONResponse(content=result, status_code=201)


async def _do_create_shipment(body: dict) -> dict:
    """Actual shipment creation business logic."""
    # ... call carrier API, create DB record, etc.
    return {"id": str(uuid.uuid4()), "status": "created"}
```

### Database-level idempotency

```python
import asyncpg
import logging
from typing import Optional

logger = logging.getLogger(__name__)


async def create_shipment_idempotent(
    conn: asyncpg.Connection,
    order_id: str,
    carrier_code: str,
    carrier_shipment_id: str,
    tracking_number: str,
) -> dict:
    """
    Database-level idempotency using UNIQUE constraint + INSERT ON CONFLICT.
    
    The shipments table has:
    UNIQUE(order_id, carrier_code)  -- can't have two shipments for same order+carrier
    
    INSERT ... ON CONFLICT DO UPDATE returns existing row on conflict.
    This means calling this function 100 times is the same as calling it once.
    """
    result = await conn.fetchrow(
        """
        INSERT INTO shipments (order_id, carrier_code, carrier_shipment_id, tracking_number, created_at)
        VALUES ($1, $2, $3, $4, NOW())
        ON CONFLICT (order_id, carrier_code) DO UPDATE
            SET updated_at = NOW()  -- update something innocuous to return the row
        RETURNING id, order_id, carrier_code, tracking_number, created_at
        """,
        order_id, carrier_code, carrier_shipment_id, tracking_number
    )
    
    # Whether we just created it or it already existed, we return the canonical record
    return dict(result)


async def process_shipment_event_idempotent(
    conn: asyncpg.Connection,
    event_id: str,
    event_data: dict,
) -> bool:
    """
    At-least-once event processing with database deduplication.
    
    Returns True if this was a new (not duplicate) event.
    Returns False if this event was already processed (duplicate).
    """
    try:
        await conn.execute(
            """
            INSERT INTO processed_events (event_id, processed_at, data)
            VALUES ($1, NOW(), $2)
            """,
            event_id, json.dumps(event_data)
        )
        return True  # New event
        
    except asyncpg.UniqueViolationError:
        logger.info(
            "Duplicate event detected, skipping",
            extra={"event_id": event_id},
        )
        return False  # Duplicate
```

---

## Rate Limiting {#rate-limiting}

### Token bucket algorithm (conceptual)

The token bucket is the standard algorithm for rate limiting:

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second
Each request: costs 1 token

At t=0: bucket has 100 tokens
Make 100 requests instantly: bucket empty
t=0.1: 1 new token added → can make 1 more request
t=1.0: 10 new tokens added → can make 10 requests
t=10: bucket full again (100 tokens)

Result: burst of 100, then sustained 10/s
```

The leaky bucket is similar but flattens bursts — outputs at a constant rate regardless of input.

### Rate-limited client wrapper

```python
import asyncio
import time
import logging
from typing import Optional
import httpx

logger = logging.getLogger(__name__)


class TokenBucket:
    """
    Token bucket rate limiter.
    Thread-safe using asyncio.Lock.
    
    Usage:
        bucket = TokenBucket(rate=10, capacity=100)
        await bucket.acquire()  # waits if needed
        # make request
    """
    
    def __init__(
        self,
        rate: float,       # tokens per second
        capacity: float,   # maximum tokens (controls burst size)
        initial_tokens: Optional[float] = None,
    ):
        self.rate = rate
        self.capacity = capacity
        self._tokens = initial_tokens if initial_tokens is not None else capacity
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()
    
    async def acquire(self, tokens: float = 1.0) -> float:
        """
        Acquire tokens, waiting if the bucket is empty.
        Returns the time waited in seconds.
        """
        async with self._lock:
            wait_time = self._refill_and_get_wait(tokens)
        
        if wait_time > 0:
            logger.debug(f"Rate limit: waiting {wait_time:.2f}s")
            await asyncio.sleep(wait_time)
        
        return wait_time
    
    def _refill_and_get_wait(self, tokens: float) -> float:
        """Refill bucket based on elapsed time, return wait time needed."""
        now = time.monotonic()
        elapsed = now - self._last_refill
        
        # Add new tokens based on elapsed time
        new_tokens = elapsed * self.rate
        self._tokens = min(self.capacity, self._tokens + new_tokens)
        self._last_refill = now
        
        if self._tokens >= tokens:
            self._tokens -= tokens
            return 0.0
        else:
            # Calculate how long to wait for enough tokens
            deficit = tokens - self._tokens
            wait_time = deficit / self.rate
            self._tokens = 0  # consume what's available
            return wait_time


class RateLimitedHTTPClient:
    """
    HTTP client with built-in rate limiting.
    
    Two modes:
    1. Proactive: use TokenBucket to limit our own request rate
    2. Reactive: detect 429 responses and back off
    
    Both modes together = production-ready client that won't get banned.
    """
    
    def __init__(
        self,
        base_url: str,
        headers: dict,
        requests_per_second: float,
        burst_capacity: int,
        max_retry_after: float = 300.0,  # max seconds to wait on 429
    ):
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers=headers,
            timeout=httpx.Timeout(connect=5.0, read=30.0),
        )
        self._bucket = TokenBucket(rate=requests_per_second, capacity=burst_capacity)
        self._max_retry_after = max_retry_after
        
        # Track observed rate limit headers
        self._rate_limit_remaining: Optional[int] = None
        self._rate_limit_reset: Optional[float] = None
    
    def _extract_rate_limit_info(self, response: httpx.Response) -> None:
        """Extract and cache rate limit headers for proactive limiting."""
        remaining = response.headers.get("X-RateLimit-Remaining")
        reset = response.headers.get("X-RateLimit-Reset")
        
        if remaining is not None:
            try:
                self._rate_limit_remaining = int(remaining)
            except ValueError:
                pass
        
        if reset is not None:
            try:
                # Reset may be a Unix timestamp or seconds-until-reset
                reset_val = float(reset)
                if reset_val > 1e9:  # Unix timestamp
                    self._rate_limit_reset = reset_val
                else:  # Seconds until reset
                    self._rate_limit_reset = time.time() + reset_val
            except ValueError:
                pass
    
    def _should_proactively_slow_down(self) -> bool:
        """
        Should we slow down proactively before hitting the limit?
        If we have < 10% of rate limit remaining, start throttling.
        """
        if self._rate_limit_remaining is not None and self._rate_limit_remaining < 10:
            return True
        return False
    
    async def request(
        self,
        method: str,
        path: str,
        **kwargs,
    ) -> httpx.Response:
        """
        Make an HTTP request with rate limiting.
        """
        # Proactive slow-down if we're running low on quota
        if self._should_proactively_slow_down():
            wait_until = self._rate_limit_reset or (time.time() + 1.0)
            wait_time = max(0, wait_until - time.time())
            if wait_time > 0:
                logger.info(
                    f"Proactively slowing down — rate limit nearly exhausted, waiting {wait_time:.1f}s"
                )
                await asyncio.sleep(wait_time)
        
        # Acquire token from our own bucket
        await self._bucket.acquire()
        
        # Make request
        response = await self._client.request(method, path, **kwargs)
        
        # Extract rate limit info for next request
        self._extract_rate_limit_info(response)
        
        # Handle 429 reactively
        if response.status_code == 429:
            retry_after_header = response.headers.get("Retry-After")
            
            wait_time = 60.0  # default if no header
            
            if retry_after_header:
                try:
                    wait_time = float(retry_after_header)
                except ValueError:
                    # HTTP date format — parse it
                    from email.utils import parsedate_to_datetime
                    try:
                        reset_dt = parsedate_to_datetime(retry_after_header)
                        wait_time = max(0, (reset_dt - datetime.now(UTC)).total_seconds())
                    except Exception:
                        pass
            
            wait_time = min(wait_time, self._max_retry_after)
            
            logger.warning(
                "Rate limited (429)",
                extra={
                    "path": path,
                    "wait_seconds": wait_time,
                    "retry_after_header": retry_after_header,
                }
            )
            
            await asyncio.sleep(wait_time)
            
            # Retry once after waiting
            await self._bucket.acquire()
            response = await self._client.request(method, path, **kwargs)
        
        return response
    
    async def get(self, path: str, **kwargs) -> dict:
        response = await self.request("GET", path, **kwargs)
        response.raise_for_status()
        return response.json()
    
    async def post(self, path: str, json: dict, **kwargs) -> dict:
        response = await self.request("POST", path, json=json, **kwargs)
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        await self._client.aclose()


# Example: FedEx API has 100 req/minute limit for rate tracking
fedex_client = RateLimitedHTTPClient(
    base_url="https://apis.fedex.com",
    headers={"Authorization": "Bearer ..."},
    requests_per_second=100 / 60,  # 100 per minute = 1.67/second
    burst_capacity=20,  # allow burst of 20 before throttling
)
```

---

## Circuit Breakers {#circuit-breakers}

### State machine

```
                    failure threshold exceeded
CLOSED ─────────────────────────────────────────► OPEN
(normal operation)                               (all calls blocked)
     ▲                                               │
     │                                               │ after timeout
     │   probe succeeds                              ▼
     └──────────────────────────────────── HALF-OPEN
                                           (let a few calls through)
```

### States in detail

- **CLOSED**: Normal operation. Requests flow through. Failures are counted.
- **OPEN**: Something is wrong. All requests fail immediately (fast-fail). No real calls made.
- **HALF-OPEN**: Recovery probe. Allow a small number of requests through. If they succeed → CLOSED. If they fail → back to OPEN.

### Implementation with `pybreaker`

```bash
pip install pybreaker
```

```python
import pybreaker
import httpx
import logging
from typing import Callable, Any, Optional

logger = logging.getLogger(__name__)


class CircuitBreakerOpenError(Exception):
    """Raised when circuit breaker is open — fail fast without calling downstream."""
    pass


def create_circuit_breaker(
    name: str,
    fail_max: int = 5,        # open circuit after this many failures
    reset_timeout: int = 60,  # seconds before trying to close (OPEN → HALF-OPEN)
) -> pybreaker.CircuitBreaker:
    """
    Create a circuit breaker with logging listeners.
    
    fail_max: how many failures before opening
    reset_timeout: how many seconds in OPEN state before testing recovery
    """
    
    class LoggingListener(pybreaker.CircuitBreakerListener):
        def state_change(self, cb, old_state, new_state):
            logger.warning(
                "Circuit breaker state change",
                extra={
                    "breaker_name": cb.name,
                    "old_state": old_state.name,
                    "new_state": new_state.name,
                    "fail_count": cb.fail_counter,
                }
            )
        
        def failure(self, cb, exc):
            logger.warning(
                "Circuit breaker: failure recorded",
                extra={
                    "breaker_name": cb.name,
                    "fail_count": cb.fail_counter,
                    "fail_max": cb.fail_max,
                    "exception": str(exc),
                }
            )
        
        def success(self, cb):
            if cb.current_state == "half-open":
                logger.info(
                    "Circuit breaker: success in half-open state, closing",
                    extra={"breaker_name": cb.name}
                )
    
    return pybreaker.CircuitBreaker(
        fail_max=fail_max,
        reset_timeout=reset_timeout,
        name=name,
        listeners=[LoggingListener()],
    )


class ResilientAPIClient:
    """
    HTTP client combining circuit breaker, rate limiting, and retry.
    
    Order of operations:
    1. Circuit breaker check: if OPEN, fail fast
    2. Rate limiter: wait for token
    3. HTTP request
    4. Retry on transient failure
    5. Circuit breaker records outcome
    """
    
    def __init__(
        self,
        base_url: str,
        api_name: str,
        rate_per_second: float = 10.0,
        circuit_fail_max: int = 5,
        circuit_reset_timeout: int = 60,
    ):
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=httpx.Timeout(connect=5.0, read=30.0),
        )
        self._breaker = create_circuit_breaker(
            name=api_name,
            fail_max=circuit_fail_max,
            reset_timeout=circuit_reset_timeout,
        )
        self._bucket = TokenBucket(rate=rate_per_second, capacity=rate_per_second * 5)
        self.api_name = api_name
    
    async def get(self, path: str, **kwargs) -> dict:
        return await self._request("GET", path, **kwargs)
    
    async def post(self, path: str, json: dict = None, **kwargs) -> dict:
        return await self._request("POST", path, json=json, **kwargs)
    
    async def _request(self, method: str, path: str, **kwargs) -> dict:
        """
        Execute request with full resilience stack.
        """
        # Rate limit
        await self._bucket.acquire()
        
        # Circuit breaker wraps the actual call
        try:
            @self._breaker
            async def _make_request():
                response = await self._client.request(method, path, **kwargs)
                
                # 5xx errors count as failures for circuit breaker
                if response.status_code >= 500:
                    raise httpx.HTTPStatusError(
                        f"Server error {response.status_code}",
                        request=response.request,
                        response=response,
                    )
                
                response.raise_for_status()
                return response.json()
            
            return await _make_request()
            
        except pybreaker.CircuitBreakerError as e:
            logger.error(
                "Circuit breaker OPEN — failing fast without calling API",
                extra={
                    "api_name": self.api_name,
                    "path": path,
                    "state": self._breaker.current_state,
                }
            )
            raise CircuitBreakerOpenError(
                f"{self.api_name} is currently unavailable (circuit breaker open)"
            ) from e


# Per-dependency circuit breakers
# Critical: each external dependency gets its OWN circuit breaker
# If FedEx's API goes down, it shouldn't affect UPS's circuit breaker

class CarrierClients:
    """
    Manage multiple carrier clients, each with independent circuit breakers.
    If Carrier A is down, Carrier B's circuit breaker is unaffected.
    """
    
    def __init__(self):
        self.clients: dict[str, ResilientAPIClient] = {}
    
    def add_carrier(
        self,
        carrier_id: str,
        base_url: str,
        rate_per_second: float,
    ) -> None:
        self.clients[carrier_id] = ResilientAPIClient(
            base_url=base_url,
            api_name=f"carrier_{carrier_id}",
            rate_per_second=rate_per_second,
        )
    
    async def get_rates(self, carrier_id: str, shipment: dict) -> dict:
        if carrier_id not in self.clients:
            raise ValueError(f"Unknown carrier: {carrier_id}")
        
        try:
            return await self.clients[carrier_id].post("/rates", json=shipment)
        except CircuitBreakerOpenError:
            # Graceful degradation: carrier unavailable
            return {"error": "carrier_unavailable", "carrier": carrier_id}
    
    async def get_rates_all_carriers(self, shipment: dict) -> dict[str, dict]:
        """
        Get rates from all carriers concurrently.
        Circuit-breaker-open carriers return error dict, don't block others.
        """
        results = await asyncio.gather(
            *[
                self.get_rates(carrier_id, shipment)
                for carrier_id in self.clients
            ],
            return_exceptions=True,
        )
        
        return {
            carrier_id: result if not isinstance(result, Exception) else {"error": str(result)}
            for carrier_id, result in zip(self.clients.keys(), results)
        }
```

---

## Timeouts: All Three Kinds {#timeouts}

### What each timeout covers

```
Request lifecycle:
                    connect_timeout
                    │         │
Client ────────────►│ TCP SYN │◄─── Server
                    │ TCP SYN+ACK
                    │ TCP ACK │
                    └─────────┘
                         │
                    write_timeout (if large body)
                         │
Client ────[POST body]──►│
                         │
                    read_timeout
                         │
Client ◄───[Response]────│
                         │
                    pool_timeout (waiting for connection from pool)
```

| Timeout type | What it limits | Typical value |
|--------------|----------------|---------------|
| connect | TCP handshake + TLS negotiation | 3-10 seconds |
| read | Waiting for response data | 10-120 seconds |
| write | Sending request body (large uploads) | 10-60 seconds |
| pool | Waiting for connection from pool | 3-10 seconds |

### What happens without timeouts

```
Without timeouts:
- Server hangs, never responds
- Your thread/coroutine hangs indefinitely
- Thread pool exhausted (sync) or connection pool exhausted (async)
- New requests start failing immediately ("connection pool full")
- Service appears healthy from circuit breaker's perspective
- Your service becomes effectively unavailable
Time to notice: 0-30 minutes
Result: complete outage of your service
```

```python
import httpx


# The wrong way: no timeout (hangs forever)
bad_client = httpx.AsyncClient()  # No timeout!


# The right way: explicit timeouts for every scenario
def get_timeouts_for_operation(operation: str) -> httpx.Timeout:
    """
    Per-operation timeout configuration.
    
    Different operations have different latency expectations:
    - Rate lookup: fast API call, should be <2s
    - Shipment creation: involves external processing, allow 30s
    - Label generation: PDF rendering, allow 60s
    - Bulk export: large response, allow 120s
    """
    timeout_map = {
        "rate_lookup": httpx.Timeout(connect=3.0, read=5.0, write=5.0, pool=3.0),
        "shipment_create": httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
        "label_generate": httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=5.0),
        "bulk_export": httpx.Timeout(connect=5.0, read=120.0, write=30.0, pool=5.0),
        "health_check": httpx.Timeout(connect=2.0, read=5.0, write=2.0, pool=2.0),
        "default": httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
    }
    return timeout_map.get(operation, timeout_map["default"])


# Per-request timeout override in httpx
async def fetch_rates_with_tight_timeout(client: httpx.AsyncClient, payload: dict) -> dict:
    """
    Override timeout per-request for latency-sensitive operations.
    Rate lookups should fail fast — better to show "unavailable" than wait 30s.
    """
    try:
        response = await client.post(
            "/rates",
            json=payload,
            timeout=httpx.Timeout(connect=2.0, read=3.0),  # Tight timeout for rates
        )
        response.raise_for_status()
        return response.json()
    except httpx.ReadTimeout:
        logger.warning("Rate lookup timed out — returning cached/default rates")
        return {"error": "timeout", "fallback": True}


# Handling timeout exceptions correctly
async def create_shipment_safe(
    client: httpx.AsyncClient,
    payload: dict,
    idempotency_key: str,
) -> dict:
    """
    Safe shipment creation that handles timeouts correctly.
    
    The tricky case: ReadTimeout on a POST.
    - Your request was sent
    - The server may or may not have processed it
    - You cannot know without checking
    
    With idempotency key: retry safely — server deduplicates
    Without idempotency key: DON'T retry — risk of duplicate shipment
    """
    try:
        response = await client.post(
            "/shipments",
            json=payload,
            headers={"Idempotency-Key": idempotency_key},
        )
        response.raise_for_status()
        return response.json()
    
    except httpx.ConnectTimeout:
        # Request never left — safe to retry immediately with same key
        logger.warning("Connect timeout on shipment creation — will retry")
        raise  # Let tenacity handle retry
    
    except httpx.ReadTimeout:
        # Request was sent — carrier MAY have processed it
        # Retry is SAFE because we have an idempotency key
        # The carrier will return the same result for the same key
        logger.warning(
            "Read timeout on shipment creation — will retry with same idempotency key",
            extra={"idempotency_key": idempotency_key}
        )
        raise  # Let tenacity handle retry — idempotency key protects us
    
    except httpx.WriteTimeout:
        # Our body upload was interrupted
        # Safe to retry from scratch — request wasn't fully received
        logger.warning("Write timeout — request body not fully sent, will retry")
        raise
```

---

## Dead Letter Patterns {#dead-letter-patterns}

### What goes to the dead letter queue

Messages in the DLQ are ones that permanently failed — exhausted all retries, non-retryable error, or exceeded maximum age.

```python
import boto3
import json
import logging
from dataclasses import dataclass, asdict
from datetime import datetime, UTC
from typing import Optional

logger = logging.getLogger(__name__)


@dataclass
class FailedMessage:
    """
    A message that has exhausted all retries.
    Sent to DLQ with full context for later investigation/replay.
    """
    original_message_id: str
    original_body: dict
    failure_reason: str
    failure_type: str  # "max_retries_exceeded", "non_retryable_error", "invalid_payload"
    attempt_count: int
    first_failure_at: str
    last_failure_at: str
    queue_name: str
    metadata: dict  # additional context (carrier, order_id, etc.)


class DeadLetterHandler:
    """
    Handles failed messages by sending to SQS DLQ and triggering alerts.
    """
    
    def __init__(
        self,
        dlq_url: str,
        alert_threshold: int = 10,  # alert when DLQ reaches this depth
        region: str = "us-east-1",
    ):
        self.dlq_url = dlq_url
        self.alert_threshold = alert_threshold
        self.sqs = boto3.client("sqs", region_name=region)
    
    def send_to_dlq(self, failed_message: FailedMessage) -> str:
        """Send a failed message to the dead letter queue."""
        message_body = {
            **asdict(failed_message),
            "sent_to_dlq_at": datetime.now(UTC).isoformat(),
        }
        
        response = self.sqs.send_message(
            QueueUrl=self.dlq_url,
            MessageBody=json.dumps(message_body),
            MessageAttributes={
                "failure_type": {
                    "StringValue": failed_message.failure_type,
                    "DataType": "String",
                },
                "queue_name": {
                    "StringValue": failed_message.queue_name,
                    "DataType": "String",
                },
            },
        )
        
        message_id = response["MessageId"]
        
        logger.error(
            "Message sent to DLQ",
            extra={
                "dlq_message_id": message_id,
                "original_message_id": failed_message.original_message_id,
                "failure_reason": failed_message.failure_reason,
                "attempt_count": failed_message.attempt_count,
            }
        )
        
        # Check DLQ depth and alert if threshold exceeded
        self._check_dlq_depth()
        
        return message_id
    
    def _check_dlq_depth(self) -> int:
        """Check DLQ depth and trigger alert if needed."""
        response = self.sqs.get_queue_attributes(
            QueueUrl=self.dlq_url,
            AttributeNames=["ApproximateNumberOfMessages"],
        )
        
        depth = int(response["Attributes"].get("ApproximateNumberOfMessages", 0))
        
        if depth >= self.alert_threshold:
            logger.error(
                "DLQ depth exceeded threshold — investigate failed messages",
                extra={
                    "dlq_url": self.dlq_url,
                    "depth": depth,
                    "threshold": self.alert_threshold,
                }
            )
            # In production: call PagerDuty, send Slack alert, etc.
        
        return depth
    
    def replay_dlq_messages(
        self,
        target_queue_url: str,
        max_messages: int = 100,
        message_filter: Optional[dict] = None,
    ) -> int:
        """
        Replay messages from DLQ back to the original queue.
        Call this after fixing the root cause.
        
        Returns number of messages replayed.
        """
        replayed = 0
        
        while replayed < max_messages:
            # Receive batch from DLQ
            response = self.sqs.receive_message(
                QueueUrl=self.dlq_url,
                MaxNumberOfMessages=min(10, max_messages - replayed),
                MessageAttributeNames=["All"],
            )
            
            messages = response.get("Messages", [])
            if not messages:
                break
            
            for msg in messages:
                body = json.loads(msg["Body"])
                
                # Apply filter if specified
                if message_filter:
                    should_replay = all(
                        body.get(k) == v for k, v in message_filter.items()
                    )
                    if not should_replay:
                        continue
                
                # Re-send original message body to target queue
                original_body = body.get("original_body", body)
                
                self.sqs.send_message(
                    QueueUrl=target_queue_url,
                    MessageBody=json.dumps(original_body),
                )
                
                # Delete from DLQ
                self.sqs.delete_message(
                    QueueUrl=self.dlq_url,
                    ReceiptHandle=msg["ReceiptHandle"],
                )
                
                replayed += 1
                logger.info(
                    "Replayed DLQ message",
                    extra={
                        "original_message_id": body.get("original_message_id"),
                        "target_queue": target_queue_url,
                    }
                )
        
        logger.info(f"DLQ replay complete: replayed {replayed} messages")
        return replayed
```

---

## The 2am API Ban Post-Mortem {#the-2am-incident}

> **FDE Context**: "Our integration was hammering their API at 2am and got us banned." This is a real incident type. Here's the full diagnosis and fix.

### What happened

```
Timeline:
23:45 — Carrier API experiences intermittent errors (their internal issue)
23:46 — Our retry logic fires for every in-flight request
23:46 — All retries hit simultaneously (no jitter): thundering herd
23:47 — Carrier API recovers but is overwhelmed by our retry storm
23:48 — Carrier API's WAF detects abnormal traffic pattern, blocks our IP
00:00 — We wake up to "all shipments failing"
02:00 — Carrier's ops team manually lifts the ban after 2 hours of escalation
```

### What was wrong (and the fixes)

```python
# ─── BAD: What the code looked like before the incident ──────────────────────

import time
import requests

def create_shipment_bad(payload: dict) -> dict:
    for attempt in range(5):
        try:
            response = requests.post(
                "https://api.carrier.com/shipments",
                json=payload,
                timeout=60,  # Only total timeout, not broken out
            )
            if response.status_code in (500, 503):
                time.sleep(1)  # Fixed 1s wait — no jitter, all clients sleep same duration
                continue
            response.raise_for_status()
            return response.json()
        except Exception:
            time.sleep(1)  # Same issue
    raise RuntimeError("Failed after 5 attempts")


# ─── GOOD: What it looks like after the incident ─────────────────────────────

import asyncio
import random
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

# 1. Per-dependency circuit breaker — opens after 5 failures, tests recovery after 60s
carrier_circuit_breaker = create_circuit_breaker(
    name="carrier_api",
    fail_max=5,
    reset_timeout=60,
)

# 2. Rate limiter — stay well under the API's actual limit
carrier_rate_limiter = TokenBucket(
    rate=8.0,     # Our limit: 8/s (actual API limit: 10/s — leave margin)
    capacity=20,  # Burst up to 20 but then sustained 8/s
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=2, max=120, jitter=5),  # jitter is critical
    retry=retry_if_exception(lambda e: (
        isinstance(e, httpx.HTTPStatusError)
        and e.response.status_code in {429, 500, 502, 503, 504}
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def create_shipment_good(
    client: httpx.AsyncClient,
    payload: dict,
    idempotency_key: str,
) -> dict:
    """
    Fixed version:
    1. Rate limiter prevents exceeding API quota
    2. Jitter prevents thundering herd on recovery
    3. Circuit breaker prevents hammering during outage
    4. Idempotency key makes retries safe
    5. Proper timeout configuration (separate connect/read)
    """
    # Rate limit
    await carrier_rate_limiter.acquire()
    
    # Circuit breaker
    @carrier_circuit_breaker
    async def _call():
        response = await client.post(
            "/shipments",
            json=payload,
            headers={"Idempotency-Key": idempotency_key},
            timeout=httpx.Timeout(connect=5.0, read=30.0),
        )
        if response.status_code in {500, 502, 503, 504}:
            raise httpx.HTTPStatusError(
                f"{response.status_code}", request=response.request, response=response
            )
        response.raise_for_status()
        return response.json()
    
    return await _call()
```

### Incident prevention checklist

```python
CARRIER_INTEGRATION_CHECKLIST = {
    "timeouts": {
        "connect_timeout": "Set, not default",
        "read_timeout": "Set, appropriate for operation",
        "write_timeout": "Set for large payloads",
    },
    "rate_limiting": {
        "proactive_limiting": "Token bucket at 80% of vendor limit",
        "react_to_429": "Read Retry-After header, respect it",
        "concurrency_cap": "asyncio.Semaphore limits concurrent requests",
    },
    "retries": {
        "jitter": "Exponential backoff with jitter — not fixed sleep",
        "no_retry_on_4xx": "400, 403, 422 etc. are not retried",
        "idempotency": "POST retries use idempotency keys",
    },
    "circuit_breaker": {
        "per_dependency": "Each vendor API has its own breaker",
        "fail_threshold": "5 failures in 60s opens breaker",
        "recovery_test": "Single probe after 60s in OPEN state",
        "alert_on_open": "PagerDuty/Slack alert when breaker opens",
    },
    "dead_letters": {
        "dlq_configured": "Failed messages go to DLQ, not silently dropped",
        "dlq_monitored": "Alert when DLQ depth > threshold",
        "dlq_replayable": "Replay script tested quarterly",
    },
    "logging": {
        "structured_logs": "All fields machine-parseable",
        "idempotency_key_logged": "Trace retries across attempts",
        "rate_limit_events": "429 responses logged with wait time",
        "circuit_state_changes": "OPEN/CLOSED transitions logged",
    },
}
```

---

## Composing Resilience Patterns {#composing-patterns}

### The complete resilience wrapper

```python
import asyncio
import httpx
import pybreaker
import logging
from tenacity import AsyncRetrying, stop_after_attempt, wait_exponential_jitter, retry_if_exception

logger = logging.getLogger(__name__)


class ResilientClient:
    """
    Production-ready client composing all resilience patterns.
    
    Request flow:
    1. Circuit breaker check (fail fast if open)
    2. Rate limiter (throttle to stay under quota)  
    3. Retry loop starts
    4. Timeout protection on each attempt
    5. HTTP request
    6. Check for retryable status codes
    7. On failure: record in circuit breaker, schedule retry with jitter
    8. On exhaustion: send to dead letter queue
    """
    
    def __init__(
        self,
        base_url: str,
        name: str,
        api_key: str,
        requests_per_second: float = 5.0,
        burst_capacity: int = 20,
        circuit_fail_max: int = 5,
        circuit_reset_timeout: int = 60,
        max_retries: int = 3,
        connect_timeout: float = 5.0,
        read_timeout: float = 30.0,
    ):
        self.name = name
        
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers={"X-API-Key": api_key},
            timeout=httpx.Timeout(
                connect=connect_timeout,
                read=read_timeout,
                write=10.0,
                pool=5.0,
            ),
        )
        
        self._breaker = create_circuit_breaker(
            name=name,
            fail_max=circuit_fail_max,
            reset_timeout=circuit_reset_timeout,
        )
        
        self._rate_limiter = TokenBucket(
            rate=requests_per_second,
            capacity=burst_capacity,
        )
        
        self._max_retries = max_retries
    
    def _is_retryable(self, exc: Exception) -> bool:
        if isinstance(exc, pybreaker.CircuitBreakerError):
            return False  # Don't retry open circuits
        if isinstance(exc, httpx.HTTPStatusError):
            return exc.response.status_code in {429, 500, 502, 503, 504}
        if isinstance(exc, (httpx.ConnectError, httpx.ReadTimeout)):
            return True
        return False
    
    async def request(
        self,
        method: str,
        path: str,
        idempotency_key: Optional[str] = None,
        **kwargs,
    ) -> dict:
        """
        Execute a request with full resilience stack.
        """
        headers = kwargs.pop("headers", {})
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key
        
        last_exception = None
        
        async for attempt in AsyncRetrying(
            stop=stop_after_attempt(self._max_retries),
            wait=wait_exponential_jitter(initial=1, max=60, jitter=2),
            retry=retry_if_exception(self._is_retryable),
            before_sleep=before_sleep_log(logger, logging.WARNING),
            reraise=True,
        ):
            with attempt:
                # Rate limit first
                await self._rate_limiter.acquire()
                
                # Circuit breaker check
                try:
                    @self._breaker
                    async def _call():
                        response = await self._client.request(
                            method, path, headers=headers, **kwargs
                        )
                        
                        if response.status_code >= 500:
                            raise httpx.HTTPStatusError(
                                f"Server error {response.status_code}",
                                request=response.request,
                                response=response,
                            )
                        
                        response.raise_for_status()
                        return response.json()
                    
                    return await _call()
                    
                except pybreaker.CircuitBreakerError as e:
                    logger.error(
                        "Circuit breaker open — failing fast",
                        extra={"api": self.name, "path": path}
                    )
                    raise CircuitBreakerOpenError(f"{self.name} unavailable") from e
    
    async def close(self):
        await self._client.aclose()
```

---

## Common Failure Modes {#common-failure-modes}

### 1. Retry loop with no jitter causes thundering herd

**Pattern**: `time.sleep(2)` or `wait_fixed(2)` — all clients retry at the same moment.

**Fix**: `wait_exponential_jitter(initial=1, max=60, jitter=5)`

### 2. Circuit breaker never opens because 4xx errors don't count

**Symptom**: Server is returning 401 for every request (token expired), circuit never opens, 1000 requests all get 401.

**Fix**: Define which HTTP status codes count as "failures" for the circuit breaker. 401 should count.

### 3. Idempotency key is a random UUID generated fresh on every retry

**Symptom**: Shipment created twice even though you have "idempotency keys."

**Root cause**: Each retry generates a new UUID → carrier sees it as a new request.

**Fix**: Generate idempotency key once, store it, reuse on every retry attempt.

### 4. Rate limiter doesn't count failed requests

**Symptom**: Rate limit is 100/minute. You have a burst of 100 failures, all retried at same rate, plus 100 new requests = 200 total requests = rate limited again.

**Fix**: Rate limiter should count ALL requests (successful AND failed) toward the limit.

### 5. Circuit breaker per-class, not per-instance

**Symptom**: FedEx API goes down, UPS API stops working too.

**Root cause**: Circuit breaker object is a class variable shared between all instances.

```python
# BAD
class CarrierClient:
    _breaker = pybreaker.CircuitBreaker(fail_max=5)  # shared by all instances!

# GOOD
class CarrierClient:
    def __init__(self, carrier_id: str):
        self._breaker = create_circuit_breaker(name=carrier_id)  # per instance
```

### 6. Timeout too generous for rate lookups

**Symptom**: Rate shopping takes 30+ seconds, customers wait.

**Root cause**: Rate lookup timeouts set same as shipment creation (30s).

**Fix**: Rate lookups should have 2-5s timeout. If no response in 5s, the rate isn't competitive anyway.

---

## Interview Angles {#interview-angles}

### Q: "Walk me through how you'd design retry logic for creating shipments."

**Great answer**: "Shipment creation is a POST that creates a resource on the carrier's side — it's not inherently idempotent. So the first thing I establish is an idempotency key strategy. I use a deterministic key based on the order ID, carrier code, and date — so if my process crashes and restarts, the same key is generated and the carrier deduplicates. With that in place, I'm safe to retry. I use tenacity with exponential backoff and jitter — jitter is critical to prevent thundering herd when the carrier recovers from an outage. I retry on 5xx and 429, but not on 4xx (those are payload issues, retrying won't help). I also add a circuit breaker so that if the carrier is down, I fail fast instead of queueing 10,000 retry attempts that will all hammer them when they come back up."

### Q: "What's a token bucket rate limiter and when would you use it?"

**Great answer**: "A token bucket maintains a pool of tokens that refill at a constant rate — say 10 per second — up to a maximum capacity. Each request costs one token. If the bucket is empty, you wait until a token is available. The bucket allows a burst equal to the capacity (useful for bursty workloads) but enforces the sustained rate. I use this proactively when calling vendor APIs: if the API allows 100 requests/minute, I configure my bucket to refill at 80/minute — leaving a 20% buffer before hitting the actual limit. This prevents 429 errors entirely, which is better than reacting to them."

### Q: "A customer says their integration was hammering a carrier API and got banned. How do you diagnose this?"

**Great answer**: "I'd start by pulling the request logs for the time window of the ban. I'd look for: (1) rate of requests — were we exceeding the carrier's documented limit? (2) retry pattern — were all the retries clustered in time with no jitter? (3) status codes — were we retrying 4xx errors that should have been terminal? The root cause is almost always one of: no rate limiting at all, rate limiting that doesn't account for retries, or retries without jitter. I'd implement all three fixes: proactive token bucket rate limiter, exponential backoff with full jitter, and filter retries to 5xx/429 only. I'd also add a circuit breaker so that the next time the carrier has an outage, we fail fast instead of hammering them on recovery."

### Q: "Explain the difference between a 503 and a 429. How do you handle each?"

**Great answer**: "503 Service Unavailable means the server is temporarily down — it might be overloaded, restarting, or a dependency is down. The right response is exponential backoff and retry — the server needs time to recover, and you're not the cause. 429 Too Many Requests means you're sending too many requests — you're the cause. The right response is to read the Retry-After header (which tells you exactly how long to wait), wait that long, and then resume — but also throttle your rate going forward so you don't hit 429 again. The common mistake is treating both the same way. If you respond to 429 with aggressive retry, you're making the problem worse."

---

## Practice Exercise {#practice-exercise}

### Exercise: Build the Resilient Carrier Integration Stack

**Scenario**: Your company integrates with a carrier API that:
- Has a rate limit of 60 requests/minute
- Goes down for 2-5 minute windows occasionally
- Returns 429 with `Retry-After: 60` when rate-limited
- Processes shipment creation in 5-15 seconds (long read timeout needed)
- Sometimes returns 200 with `{"error": "internal_error"}` in the body (a real gotcha)

**Requirements**:

1. **`ResilientCarrierClient`** (`resilient_carrier.py`):
   - Token bucket rate limiter at 50 req/min (leave 10/min margin)
   - Circuit breaker: opens after 5 failures, tests recovery after 90 seconds
   - Retry: max 3 attempts, exponential backoff with jitter, jitter range 0-5 seconds
   - Timeouts: connect 5s, read 20s
   - Idempotency key support on POST
   - Handle the 200-with-error-in-body gotcha
   - Structured logging on every retry, every rate limit event, every circuit state change

2. **`LoadTest`** (`load_test.py`):
   - Simulate 200 concurrent shipment creation requests
   - Show that the rate limiter prevents exceeding 50/min
   - Show that with jitter, retries are spread across a time window (not all at once)
   - Measure: total time, successful count, failed count, average wait time

3. **`CircuitBreakerTest`** (`test_circuit_breaker.py`):
   - Simulate carrier going down (mock returns 503)
   - Verify circuit opens after 5 failures
   - Verify no requests are made while circuit is open
   - Simulate carrier recovery (mock returns 200)
   - Verify circuit closes after successful probe
   - Measure: how many requests hit the carrier during outage vs with circuit breaker

**The 200-with-error gotcha**:
```python
# This response has HTTP 200 but indicates failure:
# {"status": "error", "message": "Rate limit exceeded", "code": "RATE_LIMIT"}
# Your client must detect this and treat it as retryable.
def is_application_error(response: dict) -> bool:
    """TODO: implement detection of application-level errors in 200 responses."""
    ...
```

**Evaluation criteria**:
- Is jitter implemented? (verify by logging wait times — should not be uniform)
- Does circuit breaker have its own lock? (not a class-level singleton)
- Are rate limit events logged at WARNING level with wait time?
- Does idempotency key persist across retry attempts? (same key, not new each time)
- Are 4xx errors excluded from retry? (only 5xx and 429)
- Does the 200-with-error case trigger retry?

---

## Related Files

- [REST, GraphQL, and Webhooks](01-rest-graphql-webhooks.md) — HTTP client basics that resilience wraps around
- [Auth flows](02-auth-flows.md) — 401 handling in retry logic (refresh token, then retry once)
- [Message queues (SQS)](06-message-queues.md) — dead letter queues in SQS, visibility timeouts as a form of retry
- [Data mapping](04-data-mapping-transformation.md) — validation errors (400/422) that should never be retried
