# REST, GraphQL, and Webhooks — Integration Fundamentals

**FDE Course | Part 1: Technical | Section C: Integration**

> **FDE Principle**: You are not a backend engineer building greenfield systems. You are a **technical translator** sitting between a customer's legacy infrastructure and the product you're deploying. Every concept in this file appears in real customer engagements. Master the mechanics, not just the vocabulary.

---

## Table of Contents

1. [HTTP Fundamentals (FDE Frame)](#http-fundamentals)
2. [REST Clients with `httpx`](#rest-clients-with-httpx)
3. [Retry Logic with `tenacity`](#retry-logic-with-tenacity)
4. [GraphQL Clients](#graphql-clients)
5. [Webhooks Inbound: Receiving and Verifying](#webhooks-inbound)
6. [Webhooks Outbound: Reliable Delivery](#webhooks-outbound)
7. [Event-Driven vs Polling: Trade-offs](#event-driven-vs-polling)
8. [The Duplicate Order Problem](#the-duplicate-order-problem)
9. [Common Failure Modes](#common-failure-modes)
10. [Interview Angles](#interview-angles)
11. [Practice Exercise](#practice-exercise)

---

## HTTP Fundamentals (FDE Frame) {#http-fundamentals}

You know HTTP. But knowing it from an FDE lens means understanding the **failure surface**, not just the happy path.

### Status codes that matter in integrations

| Code | Meaning | Your action |
|------|---------|-------------|
| 200 | OK | Proceed — but **check the response body**, some APIs (GraphQL, some REST) return errors with 200 |
| 201 | Created | Resource was created — store the `Location` header or response ID |
| 204 | No content | Success with no body — don't try to parse JSON |
| 400 | Bad Request | Your payload is wrong — **do not retry**, fix the data |
| 401 | Unauthorized | Token expired or wrong credentials — refresh token, then retry once |
| 403 | Forbidden | Authenticated but not authorized — escalate to customer, not a retry |
| 404 | Not Found | Resource doesn't exist — could mean deleted, could mean wrong ID |
| 409 | Conflict | Duplicate resource — treat as idempotent success if you sent the same idempotency key |
| 422 | Unprocessable Entity | Validation error — **do not retry**, log full payload for debugging |
| 429 | Rate Limited | Back off — read `Retry-After` header |
| 500 | Server Error | **Maybe retry** — transient server failure |
| 502/503/504 | Gateway errors | **Retry with backoff** — upstream dependency issue |

### The idempotency question

Before you write a single line of integration code, ask: **"Is this operation idempotent?"**

- `GET /orders/123` — idempotent, retry freely
- `POST /shipments` — **not inherently idempotent** — calling twice creates two shipments, carrier bills you twice
- `PUT /inventory/SKU-001` — idempotent by nature (sets absolute value), retry freely
- `PATCH /order/123/cancel` — idempotent if the order is already cancelled, but check the API contract

This question drives everything: your retry logic, your idempotency key design, your error handling.

---

## REST Clients with `httpx` {#rest-clients-with-httpx}

### Why `httpx` over `requests`

`requests` is synchronous only. In production integrations:
- You're often processing batches of 10,000 shipment records
- Each record requires an API call
- Sequential: 10,000 calls × 200ms each = **33 minutes**
- Async with `httpx`: ~30 seconds with a 100-connection pool

`httpx` gives you:
- Identical API to `requests` (easy migration)
- First-class async support
- HTTP/2 support
- Built-in timeout configuration that actually works
- Connection pooling by default

### Installation

```bash
pip install httpx tenacity
```

### Synchronous Client (connection pooling, timeouts, headers)

```python
import httpx
import logging
from typing import Any

logger = logging.getLogger(__name__)


def build_sync_client(
    base_url: str,
    api_key: str,
    connect_timeout: float = 5.0,
    read_timeout: float = 30.0,
    write_timeout: float = 10.0,
    pool_connections: int = 10,
    pool_max_keepalive: int = 5,
) -> httpx.Client:
    """
    Build a production-ready synchronous httpx client.

    Timeout values explained:
    - connect_timeout: how long to wait for the TCP connection to be established
      Set low (3-5s) — if a server can't accept a connection in 5s, something is wrong
    - read_timeout: how long to wait for the server to send data AFTER the request is sent
      Set higher (30s) for slow APIs, lower (5s) for fast ones
    - write_timeout: how long to wait while SENDING the request body
      Relevant for large upload payloads
    
    Pool settings:
    - connections: max simultaneous open connections to this host
    - max_keepalive: keep this many connections alive for reuse (reduces TCP overhead)
    """
    timeout = httpx.Timeout(
        connect=connect_timeout,
        read=read_timeout,
        write=write_timeout,
        pool=5.0,  # how long to wait for a connection FROM the pool
    )
    
    limits = httpx.Limits(
        max_connections=pool_connections,
        max_keepalive_connections=pool_max_keepalive,
        keepalive_expiry=30.0,  # seconds before idle connections are closed
    )
    
    return httpx.Client(
        base_url=base_url,
        timeout=timeout,
        limits=limits,
        headers={
            "X-API-Key": api_key,
            "Content-Type": "application/json",
            "Accept": "application/json",
            "User-Agent": "MyIntegration/1.0",  # always set — helps vendor debug logs
        },
        follow_redirects=True,  # some APIs redirect after POST (303 See Other)
    )


# Usage pattern — always use as context manager
def fetch_order(client: httpx.Client, order_id: str) -> dict:
    """
    Fetch a single order. Context manager ensures connection is returned to pool.
    """
    try:
        response = client.get(f"/orders/{order_id}")
        response.raise_for_status()  # raises httpx.HTTPStatusError for 4xx/5xx
        return response.json()
    except httpx.HTTPStatusError as e:
        logger.error(
            "HTTP error fetching order",
            extra={
                "order_id": order_id,
                "status_code": e.response.status_code,
                "response_body": e.response.text[:1000],  # truncate long error bodies
                "request_url": str(e.request.url),
            }
        )
        raise
    except httpx.TimeoutException as e:
        logger.error(
            "Timeout fetching order",
            extra={"order_id": order_id, "timeout_type": type(e).__name__}
        )
        raise
    except httpx.ConnectError as e:
        logger.error(
            "Connection failed fetching order",
            extra={"order_id": order_id, "error": str(e)}
        )
        raise


# Real usage
def main():
    with build_sync_client(
        base_url="https://api.example-carrier.com/v2",
        api_key="sk_live_abc123",
    ) as client:
        order = fetch_order(client, "ORD-001")
        print(order)
```

### Async Client (the production pattern for batch processing)

```python
import asyncio
import httpx
import logging
from typing import Any, Optional

logger = logging.getLogger(__name__)


class CarrierAPIClient:
    """
    Production async HTTP client for a carrier API.
    
    Design decisions:
    1. Client is created once and shared across the application lifetime
    2. Not a context manager at call site — managed by app lifecycle (startup/shutdown)
    3. Semaphore limits concurrency beyond the pool limit for rate-limiting purposes
    4. All methods are async — callers use 'await'
    """
    
    def __init__(
        self,
        base_url: str,
        api_key: str,
        max_concurrent_requests: int = 20,
        connect_timeout: float = 5.0,
        read_timeout: float = 30.0,
    ):
        self.base_url = base_url
        self.api_key = api_key
        self._semaphore = asyncio.Semaphore(max_concurrent_requests)
        
        timeout = httpx.Timeout(
            connect=connect_timeout,
            read=read_timeout,
            write=10.0,
            pool=5.0,
        )
        
        limits = httpx.Limits(
            max_connections=100,
            max_keepalive_connections=20,
            keepalive_expiry=30.0,
        )
        
        self._client = httpx.AsyncClient(
            base_url=base_url,
            timeout=timeout,
            limits=limits,
            headers={
                "X-API-Key": api_key,
                "Content-Type": "application/json",
                "Accept": "application/json",
                "User-Agent": "WarehouseIntegration/2.0",
            },
            follow_redirects=True,
            http2=True,  # Use HTTP/2 if server supports it (multiplexing = faster)
        )
        
    async def close(self):
        """Call this on application shutdown."""
        await self._client.aclose()
    
    async def get(
        self, 
        path: str, 
        params: Optional[dict] = None,
        extra_headers: Optional[dict] = None,
    ) -> dict:
        async with self._semaphore:
            response = await self._client.get(
                path,
                params=params,
                headers=extra_headers or {},
            )
            response.raise_for_status()
            return response.json()
    
    async def post(
        self,
        path: str,
        json: Optional[dict] = None,
        idempotency_key: Optional[str] = None,
    ) -> dict:
        """
        POST with optional idempotency key.
        Many carrier APIs support idempotency keys to prevent duplicate shipments.
        """
        headers = {}
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key
        
        async with self._semaphore:
            response = await self._client.post(
                path,
                json=json,
                headers=headers,
            )
            response.raise_for_status()
            return response.json()
    
    async def get_with_pagination(
        self,
        path: str,
        params: Optional[dict] = None,
        page_size: int = 100,
    ):
        """
        Generator that handles cursor-based pagination.
        Yields individual items, not pages — callers don't need to know pagination exists.
        
        Handles the common pattern:
        response.json() → {"data": [...], "next_cursor": "abc123" or null}
        """
        params = params or {}
        params["limit"] = page_size
        cursor = None
        
        while True:
            if cursor:
                params["cursor"] = cursor
            
            data = await self.get(path, params=params)
            
            items = data.get("data", [])
            for item in items:
                yield item
            
            cursor = data.get("next_cursor") or data.get("next_page_token")
            if not cursor:
                break  # no more pages
    
    async def batch_get(
        self,
        path_template: str,
        ids: list[str],
        batch_size: int = 50,
    ) -> list[dict]:
        """
        Fetch multiple resources concurrently with batching.
        
        path_template: "/orders/{id}"
        ids: ["ORD-001", "ORD-002", ...]
        
        Why batch_size? Prevents creating 10,000 concurrent tasks.
        Each batch of 50 runs concurrently, then next batch starts.
        """
        results = []
        
        for i in range(0, len(ids), batch_size):
            batch = ids[i : i + batch_size]
            tasks = [
                self.get(path_template.format(id=id))
                for id in batch
            ]
            # gather returns results in order, raises on first exception
            batch_results = await asyncio.gather(*tasks, return_exceptions=True)
            
            for id, result in zip(batch, batch_results):
                if isinstance(result, Exception):
                    logger.error(
                        "Failed to fetch resource",
                        extra={"id": id, "error": str(result)}
                    )
                    results.append(None)  # preserve ordering
                else:
                    results.append(result)
        
        return results


# FastAPI integration example — client lifecycle tied to app lifecycle
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Proper client lifecycle management in FastAPI.
    Client is created once on startup, shared across all requests, closed on shutdown.
    """
    app.state.carrier_client = CarrierAPIClient(
        base_url="https://api.carrier.com/v1",
        api_key="sk_live_abc123",
        max_concurrent_requests=50,
    )
    yield
    await app.state.carrier_client.close()


app = FastAPI(lifespan=lifespan)


async def process_shipment_batch(order_ids: list[str]) -> list[dict]:
    """Example: fetch 10,000 orders asynchronously."""
    client = app.state.carrier_client
    results = await client.batch_get("/orders/{id}", order_ids)
    return [r for r in results if r is not None]
```

### Query Parameters Done Right

```python
# Query params with httpx — handles encoding, lists, None filtering
response = client.get(
    "/shipments",
    params={
        "status": ["pending", "in_transit"],  # httpx handles list → ?status=pending&status=in_transit
        "carrier": "FEDEX",
        "page": 1,
        "include_cancelled": False,
        "created_after": "2024-01-01T00:00:00Z",
    }
)

# For params where None should be excluded (don't send ?field=None):
def clean_params(params: dict) -> dict:
    """Remove None values from params dict before passing to httpx."""
    return {k: v for k, v in params.items() if v is not None}

response = client.get("/shipments", params=clean_params({
    "carrier": "FEDEX",
    "status": None,  # will be excluded
    "page": 1,
}))
```

---

## Retry Logic with `tenacity` {#retry-logic-with-tenacity}

### Why tenacity

`tenacity` is the industry standard for retry logic in Python. It gives you:
- Declarative retry policies (decorator or context manager)
- Exponential backoff with jitter (critical — explained below)
- Per-exception-type retry logic
- Retry callbacks for logging
- Stop conditions (max attempts, max time)

```bash
pip install tenacity
```

### Understanding exponential backoff with jitter

**Without jitter (bad)**: All failed requests retry at exactly t+1s, t+2s, t+4s. If 100 clients fail simultaneously, they all hammer the server at the same moments. This is the **thundering herd problem**.

**With jitter (good)**: Each client adds random noise to the wait time. 100 clients spread their retries across a time window, reducing peak load on the recovering server.

```python
import httpx
import logging
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    wait_exponential,
    retry_if_exception_type,
    retry_if_result,
    before_sleep_log,
    after_log,
    RetryError,
)

logger = logging.getLogger(__name__)


# Retryable exception types for HTTP
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}
NON_RETRYABLE_STATUS_CODES = {400, 401, 403, 404, 409, 422}


def is_retryable_http_error(exception: Exception) -> bool:
    """
    Returns True if the exception is one we should retry.
    
    Key decisions:
    - 429 (rate limit): yes retry, but respect Retry-After
    - 5xx: yes retry — server-side transient errors
    - 4xx: NO retry — our request is wrong, retrying won't help
    - Connection errors: yes retry — transient network issue
    - Timeout: yes retry (carefully) — might be transient
    """
    if isinstance(exception, httpx.HTTPStatusError):
        return exception.response.status_code in RETRYABLE_STATUS_CODES
    if isinstance(exception, (httpx.ConnectError, httpx.RemoteProtocolError)):
        return True
    if isinstance(exception, httpx.ReadTimeout):
        return True  # but be careful — the request may have been received
    return False


def get_retry_after(exception: Exception) -> float | None:
    """
    Extract Retry-After header from 429 responses.
    Returns seconds to wait, or None if not present.
    """
    if isinstance(exception, httpx.HTTPStatusError):
        if exception.response.status_code == 429:
            retry_after = exception.response.headers.get("Retry-After")
            if retry_after:
                try:
                    return float(retry_after)
                except ValueError:
                    pass  # it's an HTTP date string — parse it if needed
    return None


# Basic retry decorator
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=60, jitter=2),
    retry=retry_if_exception_type((httpx.HTTPStatusError, httpx.TransportError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,  # re-raise the original exception after all retries fail
)
def fetch_shipment_status(client: httpx.Client, tracking_number: str) -> dict:
    """Simple retry with exponential backoff + jitter."""
    response = client.get(f"/tracking/{tracking_number}")
    response.raise_for_status()
    return response.json()


# Advanced retry with conditional logic
class RetryableAPIClient:
    """
    Production retry pattern with:
    - Per-operation retry configuration
    - Retry-After header respect
    - Non-retryable error fast-fail
    - Structured logging on each retry
    """
    
    def __init__(self, client: httpx.AsyncClient):
        self.client = client
    
    async def _request_with_retry(
        self,
        method: str,
        url: str,
        max_attempts: int = 3,
        initial_wait: float = 1.0,
        max_wait: float = 60.0,
        **kwargs,
    ) -> httpx.Response:
        last_exception = None
        
        for attempt in range(1, max_attempts + 1):
            try:
                response = await self.client.request(method, url, **kwargs)
                
                # Check for retryable status codes
                if response.status_code in RETRYABLE_STATUS_CODES:
                    if attempt == max_attempts:
                        response.raise_for_status()  # raise on final attempt
                    
                    # Respect Retry-After for 429
                    wait_time = None
                    if response.status_code == 429:
                        retry_after = response.headers.get("Retry-After")
                        if retry_after:
                            try:
                                wait_time = float(retry_after)
                            except ValueError:
                                pass
                    
                    if wait_time is None:
                        # Exponential backoff with jitter
                        import random
                        wait_time = min(
                            initial_wait * (2 ** (attempt - 1)) + random.uniform(0, 2),
                            max_wait
                        )
                    
                    logger.warning(
                        "Retryable HTTP error, backing off",
                        extra={
                            "method": method,
                            "url": url,
                            "status_code": response.status_code,
                            "attempt": attempt,
                            "max_attempts": max_attempts,
                            "wait_seconds": wait_time,
                        }
                    )
                    await asyncio.sleep(wait_time)
                    continue
                
                # Non-retryable 4xx — raise immediately
                if response.status_code in NON_RETRYABLE_STATUS_CODES:
                    response.raise_for_status()
                
                response.raise_for_status()
                return response
                
            except httpx.HTTPStatusError:
                raise  # already handled above
                
            except (httpx.ConnectError, httpx.ReadTimeout, httpx.RemoteProtocolError) as e:
                last_exception = e
                if attempt == max_attempts:
                    raise
                
                import random
                wait_time = min(
                    initial_wait * (2 ** (attempt - 1)) + random.uniform(0, 1),
                    max_wait
                )
                
                logger.warning(
                    "Transport error, retrying",
                    extra={
                        "method": method,
                        "url": url,
                        "error": str(e),
                        "attempt": attempt,
                        "wait_seconds": wait_time,
                    }
                )
                await asyncio.sleep(wait_time)
        
        raise last_exception  # should not reach here
    
    async def get(self, url: str, **kwargs) -> dict:
        response = await self._request_with_retry("GET", url, max_attempts=5, **kwargs)
        return response.json()
    
    async def post(self, url: str, json: dict, idempotency_key: str | None = None, **kwargs) -> dict:
        """
        POST with fewer retries — be careful with non-idempotent operations.
        Only retry if we KNOW the server didn't process the request (connection errors).
        For timeout errors on POST, we DON'T know if the server processed it.
        """
        headers = kwargs.pop("headers", {})
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key
        
        # With idempotency key: safe to retry all errors
        # Without idempotency key: only retry connection errors (not timeouts)
        max_attempts = 3 if idempotency_key else 2
        
        response = await self._request_with_retry(
            "POST", url, 
            max_attempts=max_attempts,
            json=json, 
            headers=headers,
            **kwargs
        )
        return response.json()
```

---

## GraphQL Clients {#graphql-clients}

### GraphQL fundamentals for FDEs

GraphQL is increasingly common in modern platforms (Shopify, GitHub, many B2B SaaS). Key differences from REST that trip up engineers:

1. **Single endpoint**: All requests go to `/graphql` (POST)
2. **Client specifies shape**: You define exactly what fields you want
3. **200 OK even on error**: This is the #1 gotcha — GraphQL returns 200 with `{"errors": [...]}` in the body
4. **No pagination standard**: Each API does it differently (cursor-based, offset-based, relay spec)

```bash
pip install gql[httpx] gql[aiohttp]
```

### The GraphQL 200-Error Gotcha

```python
import httpx
import json
from typing import Any, Optional

# BAD — misses GraphQL errors
def bad_graphql_request(client: httpx.Client, query: str, variables: dict) -> dict:
    response = client.post(
        "/graphql",
        json={"query": query, "variables": variables}
    )
    response.raise_for_status()  # Only catches HTTP errors, not GraphQL errors
    return response.json()["data"]  # Will throw KeyError if errors present, losing error info


# GOOD — properly handles GraphQL errors
def graphql_request(
    client: httpx.Client,
    query: str,
    variables: Optional[dict] = None,
    operation_name: Optional[str] = None,
) -> Any:
    """
    Proper GraphQL request handler.
    
    GraphQL response structure:
    {
      "data": { ... },          # present on success (may also be null on partial error)
      "errors": [ ... ],        # present on error — can coexist with data!
      "extensions": { ... }     # optional metadata (query cost, debug info)
    }
    """
    payload = {"query": query}
    if variables:
        payload["variables"] = variables
    if operation_name:
        payload["operationName"] = operation_name
    
    response = client.post("/graphql", json=payload)
    response.raise_for_status()  # Catches HTTP-level errors
    
    body = response.json()
    
    # Check for GraphQL errors — these come back with HTTP 200!
    if "errors" in body:
        errors = body["errors"]
        error_messages = [e.get("message", str(e)) for e in errors]
        
        # Some APIs return partial data + errors — log both
        if "data" not in body or body["data"] is None:
            raise GraphQLError(
                f"GraphQL errors: {'; '.join(error_messages)}",
                errors=errors,
            )
        else:
            # Partial success — data AND errors
            # Log the errors but return the data
            import logging
            logging.getLogger(__name__).warning(
                "GraphQL partial success — errors present with data",
                extra={"errors": errors, "query_excerpt": query[:200]},
            )
    
    return body.get("data")


class GraphQLError(Exception):
    def __init__(self, message: str, errors: list[dict]):
        super().__init__(message)
        self.errors = errors
        # Extract error codes if present (some APIs include them)
        self.codes = [
            e.get("extensions", {}).get("code") 
            for e in errors 
            if e.get("extensions", {}).get("code")
        ]
```

### Full GraphQL Client Wrapper with `gql`

```python
from gql import gql, Client
from gql.transport.httpx import HTTPXTransport
from gql.transport.aiohttp import AIOHTTPTransport
import logging
from typing import Any, Optional

logger = logging.getLogger(__name__)


class ShopifyGraphQLClient:
    """
    GraphQL client for Shopify Admin API.
    
    Shopify uses GraphQL with:
    - Cursor-based pagination (Relay spec)
    - Cost-based rate limiting (query cost, not request count)
    - Throttle status in extensions
    """
    
    SHOPIFY_ENDPOINT = "https://{shop}.myshopify.com/admin/api/2024-01/graphql.json"
    
    def __init__(self, shop: str, access_token: str):
        self.shop = shop
        transport = HTTPXTransport(
            url=self.SHOPIFY_ENDPOINT.format(shop=shop),
            headers={
                "X-Shopify-Access-Token": access_token,
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )
        self.client = Client(
            transport=transport,
            fetch_schema_from_transport=False,  # Don't auto-introspect — saves a round trip
        )
    
    def execute(
        self, 
        query_string: str, 
        variables: Optional[dict] = None
    ) -> dict:
        """
        Execute a GraphQL query with proper error handling.
        """
        query = gql(query_string)
        
        try:
            result = self.client.execute(query, variable_values=variables)
            return result
        except Exception as e:
            logger.error(
                "GraphQL execution failed",
                extra={
                    "shop": self.shop,
                    "error": str(e),
                    "query_excerpt": query_string[:300],
                    "variables": variables,
                }
            )
            raise
    
    def get_orders_page(
        self, 
        after_cursor: Optional[str] = None,
        page_size: int = 50,
    ) -> tuple[list[dict], Optional[str], bool]:
        """
        Returns: (orders, next_cursor, has_next_page)
        
        Shopify Relay pagination pattern:
        Each node in a Connection has a cursor.
        pageInfo tells you if there are more pages.
        """
        query = """
        query GetOrders($first: Int!, $after: String) {
            orders(first: $first, after: $after, sortKey: CREATED_AT) {
                edges {
                    cursor
                    node {
                        id
                        name
                        createdAt
                        displayFinancialStatus
                        lineItems(first: 50) {
                            edges {
                                node {
                                    title
                                    quantity
                                    originalUnitPriceSet {
                                        shopMoney {
                                            amount
                                            currencyCode
                                        }
                                    }
                                }
                            }
                        }
                        shippingAddress {
                            address1
                            city
                            provinceCode
                            countryCodeV2
                            zip
                        }
                    }
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
        """
        
        variables = {"first": page_size}
        if after_cursor:
            variables["after"] = after_cursor
        
        result = self.execute(query, variables)
        
        orders_connection = result["orders"]
        orders = [edge["node"] for edge in orders_connection["edges"]]
        page_info = orders_connection["pageInfo"]
        
        return (
            orders,
            page_info["endCursor"] if page_info["hasNextPage"] else None,
            page_info["hasNextPage"],
        )
    
    def iterate_all_orders(self, page_size: int = 50):
        """
        Generator that yields all orders handling pagination transparently.
        """
        cursor = None
        page_count = 0
        
        while True:
            orders, next_cursor, has_next = self.get_orders_page(
                after_cursor=cursor,
                page_size=page_size,
            )
            
            page_count += 1
            logger.debug(f"Fetched orders page {page_count}, count={len(orders)}")
            
            for order in orders:
                yield order
            
            if not has_next:
                break
            cursor = next_cursor
    
    def introspect_type(self, type_name: str) -> dict:
        """
        Introspection query — useful when you need to understand what fields are available.
        Run this in development to explore the schema.
        """
        query = """
        query IntrospectType($name: String!) {
            __type(name: $name) {
                name
                kind
                fields {
                    name
                    type {
                        name
                        kind
                        ofType {
                            name
                            kind
                        }
                    }
                    description
                }
            }
        }
        """
        return self.execute(query, {"name": type_name})
```

### GraphQL Mutations (Create/Update)

```python
def create_fulfillment(
    client: ShopifyGraphQLClient,
    order_id: str,
    tracking_number: str,
    carrier: str,
) -> dict:
    """
    GraphQL mutation to create a fulfillment.
    Mutations use the 'mutation' keyword instead of 'query'.
    """
    mutation = """
    mutation FulfillOrder(
        $orderId: ID!,
        $trackingNumber: String!,
        $carrier: String!
    ) {
        fulfillmentCreate(
            input: {
                orderId: $orderId,
                trackingInfo: {
                    number: $trackingNumber,
                    company: $carrier
                }
            }
        ) {
            fulfillment {
                id
                status
                trackingInfo {
                    number
                    company
                }
            }
            userErrors {
                field
                message
            }
        }
    }
    """
    
    result = client.execute(mutation, {
        "orderId": f"gid://shopify/Order/{order_id}",
        "trackingNumber": tracking_number,
        "carrier": carrier,
    })
    
    fulfillment_result = result["fulfillmentCreate"]
    
    # Shopify pattern: userErrors in response body (not GraphQL-level errors)
    # This is different from GraphQL errors — it's application-level validation
    if fulfillment_result["userErrors"]:
        errors = fulfillment_result["userErrors"]
        error_str = "; ".join(f"{e['field']}: {e['message']}" for e in errors)
        raise ValueError(f"Fulfillment creation failed: {error_str}")
    
    return fulfillment_result["fulfillment"]
```

---

## Webhooks Inbound: Receiving and Verifying {#webhooks-inbound}

### The webhook security problem

When you receive a POST to your webhook endpoint, you have no way to know it's actually from the legitimate sender — unless you verify the payload signature. Every production webhook system uses HMAC-SHA256 signatures.

**Pattern**: Sender computes `HMAC-SHA256(raw_body, shared_secret)` → sends it in a header → you recompute with the same secret → compare.

```python
import hashlib
import hmac
import json
import logging
import asyncio
from datetime import datetime, UTC
from typing import Any, Callable
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks, Depends
from fastapi.responses import JSONResponse

logger = logging.getLogger(__name__)

app = FastAPI()


# ============================================================
# HMAC Signature Verification
# ============================================================

def verify_hmac_signature(
    raw_body: bytes,
    signature_header: str,
    secret: str,
    algorithm: str = "sha256",
) -> bool:
    """
    Verify HMAC-SHA256 webhook signature.
    
    CRITICAL: Use 'raw_body' (bytes before parsing), NOT parsed JSON.
    Parsing and re-serializing JSON changes whitespace, key order — signature won't match.
    
    Args:
        raw_body: Raw request body bytes — get this BEFORE parsing
        signature_header: Value from the signature header (may have 'sha256=' prefix)
        secret: Shared secret between you and the sender
        algorithm: Hash algorithm (usually sha256)
    
    Returns:
        True if signature is valid
    """
    # Strip prefix if present (Shopify sends "sha256=abc123...", Stripe sends "t=...,v1=...")
    if "=" in signature_header:
        # Handle "sha256=<hash>" format
        parts = signature_header.split("=", 1)
        if parts[0].lower() in ("sha256", "sha1", "md5"):
            signature_header = parts[1]
    
    # Compute expected signature
    expected = hmac.new(
        secret.encode("utf-8"),
        raw_body,
        getattr(hashlib, algorithm),
    ).hexdigest()
    
    # CRITICAL: Use hmac.compare_digest to prevent timing attacks
    # Regular string comparison leaks timing information about where strings differ
    return hmac.compare_digest(expected, signature_header)


def verify_shopify_webhook(raw_body: bytes, hmac_header: str, secret: str) -> bool:
    """
    Shopify-specific: header is 'X-Shopify-Hmac-Sha256', value is base64-encoded.
    """
    import base64
    expected = hmac.new(
        secret.encode("utf-8"),
        raw_body,
        hashlib.sha256,
    ).digest()  # .digest() for bytes, .hexdigest() for hex string
    
    try:
        received = base64.b64decode(signature_header)
    except Exception:
        return False
    
    return hmac.compare_digest(expected, received)


def verify_stripe_webhook(raw_body: bytes, signature_header: str, secret: str) -> bool:
    """
    Stripe uses a timestamp + signature to prevent replay attacks.
    Header format: "t=1614556800,v1=abc123..."
    
    Stripe's pattern:
    1. Extract timestamp from header
    2. Check timestamp is recent (within 5 minutes)
    3. Compute HMAC over "{timestamp}.{body}"
    4. Compare to v1 signature
    """
    import time
    
    # Parse header
    parts = dict(part.split("=", 1) for part in signature_header.split(","))
    timestamp = parts.get("t")
    v1_signature = parts.get("v1")
    
    if not timestamp or not v1_signature:
        return False
    
    # Check timestamp freshness (prevent replay attacks)
    try:
        ts = int(timestamp)
        if abs(time.time() - ts) > 300:  # 5 minutes tolerance
            logger.warning("Webhook timestamp too old — possible replay attack")
            return False
    except ValueError:
        return False
    
    # Compute signature over "timestamp.body"
    signed_payload = f"{timestamp}.{raw_body.decode('utf-8')}".encode()
    expected = hmac.new(
        secret.encode("utf-8"),
        signed_payload,
        hashlib.sha256,
    ).hexdigest()
    
    return hmac.compare_digest(expected, v1_signature)


# ============================================================
# Idempotency: Handling Duplicate Deliveries
# ============================================================

class WebhookIdempotencyStore:
    """
    In-memory idempotency store for demonstration.
    In production: use Redis with TTL or a database with a unique constraint.
    
    Why duplicates happen:
    - Sender retries on network timeout (your server got the request, but ACK was lost)
    - Sender's retry logic fires on 200 that arrived late
    - Sender has a bug and sends twice
    
    Your system MUST handle this. "We got duplicate orders" is a real, embarrassing incident.
    """
    
    def __init__(self):
        self._processed: dict[str, datetime] = {}
    
    def is_duplicate(self, event_id: str) -> bool:
        """Returns True if we've already processed this event."""
        return event_id in self._processed
    
    def mark_processed(self, event_id: str) -> None:
        """Mark event as processed. Call AFTER successful processing."""
        self._processed[event_id] = datetime.now(UTC)
    
    def cleanup_old_entries(self, max_age_hours: int = 48) -> None:
        """Remove old entries to prevent unbounded memory growth."""
        cutoff = datetime.now(UTC)
        to_delete = [
            k for k, v in self._processed.items()
            if (cutoff - v).total_seconds() > max_age_hours * 3600
        ]
        for k in to_delete:
            del self._processed[k]


# Global store — in production, inject this as a dependency
idempotency_store = WebhookIdempotencyStore()


# ============================================================
# FastAPI Webhook Receiver
# ============================================================

WEBHOOK_SECRET = "your_shared_secret_here"  # load from env/secrets manager


async def process_order_created(payload: dict, event_id: str) -> None:
    """
    Actual business logic — called asynchronously after ack.
    This runs AFTER we've returned 200 to the sender.
    """
    logger.info(
        "Processing order created webhook",
        extra={"event_id": event_id, "order_id": payload.get("id")},
    )
    # ... your actual processing logic
    # Mark as processed AFTER successful processing
    idempotency_store.mark_processed(event_id)


@app.post("/webhooks/orders/created")
async def receive_order_created_webhook(
    request: Request,
    background_tasks: BackgroundTasks,
) -> JSONResponse:
    """
    Production webhook receiver pattern:
    
    1. Read raw body FIRST (before any parsing)
    2. Verify HMAC signature
    3. Check idempotency
    4. Return 200 IMMEDIATELY
    5. Process asynchronously in background
    
    Why return 200 immediately?
    - Webhook senders typically have a short timeout (5-30 seconds)
    - Your processing might take minutes (DB writes, downstream API calls)
    - If you don't ack fast, sender retries → you get duplicates
    - Processing happens in background task / worker queue
    """
    # Step 1: Read raw body — MUST happen before any parsing
    raw_body = await request.body()
    
    # Step 2: Verify signature
    signature = request.headers.get("X-Signature-SHA256")
    if not signature:
        logger.warning("Webhook received without signature header")
        raise HTTPException(status_code=401, detail="Missing signature")
    
    if not verify_hmac_signature(raw_body, signature, WEBHOOK_SECRET):
        logger.warning(
            "Webhook signature verification failed",
            extra={
                "ip": request.client.host if request.client else "unknown",
                "signature": signature[:20] + "...",
            }
        )
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # Step 3: Parse body (now that we've verified it)
    try:
        payload = json.loads(raw_body)
    except json.JSONDecodeError:
        raise HTTPException(status_code=400, detail="Invalid JSON body")
    
    # Step 4: Check idempotency
    # Most webhook systems send a unique event ID in the payload or headers
    event_id = (
        request.headers.get("X-Webhook-Event-Id")
        or request.headers.get("X-Delivery-Id")  # GitHub's header
        or payload.get("event_id")
        or payload.get("id")
    )
    
    if not event_id:
        # If no event ID, we can't deduplicate — log and proceed (or reject — your call)
        logger.warning("Webhook received without event ID — cannot deduplicate")
        event_id = f"unknown_{datetime.now(UTC).timestamp()}"
    
    if idempotency_store.is_duplicate(event_id):
        logger.info(
            "Duplicate webhook received, acking without reprocessing",
            extra={"event_id": event_id},
        )
        return JSONResponse(
            content={"status": "duplicate", "event_id": event_id},
            status_code=200,  # Return 200 — sender doesn't care it was a duplicate
        )
    
    # Step 5: Return 200 FAST — enqueue processing
    # background_tasks runs AFTER the response is sent
    background_tasks.add_task(process_order_created, payload, event_id)
    
    logger.info(
        "Webhook received and acked",
        extra={"event_id": event_id, "topic": "orders/created"},
    )
    
    return JSONResponse(
        content={"status": "accepted", "event_id": event_id},
        status_code=200,
    )
```

---

## Webhooks Outbound: Reliable Delivery {#webhooks-outbound}

### The outbound webhook problem

When YOUR platform is the sender:
- Customer's server might be down
- Customer's server might be slow
- Network might be flaky
- You must guarantee at-least-once delivery
- You must handle permanent failures gracefully

```python
import asyncio
import hashlib
import hmac
import json
import logging
import uuid
from dataclasses import dataclass, field
from datetime import datetime, UTC
from enum import Enum
from typing import Optional

logger = logging.getLogger(__name__)


class DeliveryStatus(Enum):
    PENDING = "pending"
    DELIVERED = "delivered"
    FAILED = "failed"
    DEAD = "dead"  # permanently failed, in DLQ


@dataclass
class WebhookDelivery:
    """Represents one delivery attempt of a webhook."""
    delivery_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    event_id: str = ""
    endpoint_url: str = ""
    payload: dict = field(default_factory=dict)
    attempt_count: int = 0
    max_attempts: int = 5
    status: DeliveryStatus = DeliveryStatus.PENDING
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    next_attempt_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    last_response_code: Optional[int] = None
    last_error: Optional[str] = None


class OutboundWebhookSender:
    """
    Reliable outbound webhook delivery with:
    - At-least-once delivery guarantee
    - Exponential backoff retry
    - HMAC signature on outbound payloads
    - Dead-letter handling
    - Payload design best practices
    """
    
    def __init__(
        self,
        signing_secret: str,
        max_attempts: int = 5,
        initial_backoff: float = 1.0,
        max_backoff: float = 3600.0,  # 1 hour max
    ):
        self.signing_secret = signing_secret
        self.max_attempts = max_attempts
        self.initial_backoff = initial_backoff
        self.max_backoff = max_backoff
        self._http_client = httpx.AsyncClient(timeout=httpx.Timeout(30.0))
    
    def _build_payload(
        self,
        event_type: str,
        data: dict,
        event_id: Optional[str] = None,
    ) -> dict:
        """
        Build a well-designed webhook payload.
        
        Best practices:
        1. Include event_id for deduplication
        2. Include timestamp
        3. Include event_type for routing
        4. Include API version for forward compatibility
        5. Wrap data in a 'data' key (leave room for metadata at top level)
        """
        return {
            "id": event_id or str(uuid.uuid4()),
            "type": event_type,
            "created_at": datetime.now(UTC).isoformat(),
            "api_version": "2024-01",
            "data": data,
        }
    
    def _sign_payload(self, raw_body: bytes, timestamp: str) -> str:
        """
        Sign outbound payload.
        Include timestamp in signature to allow replay attack prevention on receiver end.
        """
        signed_content = f"{timestamp}.{raw_body.decode('utf-8')}".encode()
        signature = hmac.new(
            self.signing_secret.encode("utf-8"),
            signed_content,
            hashlib.sha256,
        ).hexdigest()
        return f"t={timestamp},v1={signature}"
    
    def _calculate_next_attempt(self, attempt_count: int) -> datetime:
        """
        Exponential backoff: 1s, 2s, 4s, 8s, 16s, ... up to max_backoff.
        Add jitter to spread retries.
        """
        import random
        from datetime import timedelta
        
        base_wait = min(
            self.initial_backoff * (2 ** attempt_count),
            self.max_backoff,
        )
        jitter = random.uniform(0, base_wait * 0.1)  # 10% jitter
        wait_seconds = base_wait + jitter
        
        return datetime.now(UTC) + timedelta(seconds=wait_seconds)
    
    async def send(
        self,
        endpoint_url: str,
        event_type: str,
        data: dict,
        delivery: Optional[WebhookDelivery] = None,
    ) -> WebhookDelivery:
        """
        Attempt to deliver one webhook.
        Returns updated delivery record — caller persists it.
        """
        if delivery is None:
            payload = self._build_payload(event_type, data)
            delivery = WebhookDelivery(
                event_id=payload["id"],
                endpoint_url=endpoint_url,
                payload=payload,
                max_attempts=self.max_attempts,
            )
        
        delivery.attempt_count += 1
        raw_body = json.dumps(delivery.payload, separators=(",", ":")).encode("utf-8")
        timestamp = str(int(datetime.now(UTC).timestamp()))
        signature = self._sign_payload(raw_body, timestamp)
        
        try:
            response = await self._http_client.post(
                endpoint_url,
                content=raw_body,
                headers={
                    "Content-Type": "application/json",
                    "X-Webhook-Signature": signature,
                    "X-Webhook-Event-Id": delivery.event_id,
                    "X-Webhook-Attempt": str(delivery.attempt_count),
                },
            )
            
            delivery.last_response_code = response.status_code
            
            # 2xx = success
            if 200 <= response.status_code < 300:
                delivery.status = DeliveryStatus.DELIVERED
                logger.info(
                    "Webhook delivered successfully",
                    extra={
                        "event_id": delivery.event_id,
                        "endpoint": endpoint_url,
                        "attempt": delivery.attempt_count,
                        "status_code": response.status_code,
                    }
                )
                return delivery
            
            # 4xx = our payload is wrong, don't retry
            if 400 <= response.status_code < 500:
                delivery.status = DeliveryStatus.DEAD
                delivery.last_error = f"Non-retryable {response.status_code}: {response.text[:200]}"
                logger.error(
                    "Webhook rejected with non-retryable error, moving to DLQ",
                    extra={
                        "event_id": delivery.event_id,
                        "endpoint": endpoint_url,
                        "status_code": response.status_code,
                        "response_body": response.text[:500],
                    }
                )
                return delivery
            
            # 5xx = server error, schedule retry
            delivery.last_error = f"Server error {response.status_code}"
            
        except httpx.TimeoutException as e:
            delivery.last_error = f"Timeout: {type(e).__name__}"
        except httpx.ConnectError as e:
            delivery.last_error = f"Connection failed: {str(e)}"
        except Exception as e:
            delivery.last_error = f"Unexpected error: {type(e).__name__}: {str(e)}"
        
        # Schedule retry or move to dead letter
        if delivery.attempt_count >= delivery.max_attempts:
            delivery.status = DeliveryStatus.DEAD
            logger.error(
                "Webhook delivery permanently failed, moving to DLQ",
                extra={
                    "event_id": delivery.event_id,
                    "endpoint": endpoint_url,
                    "total_attempts": delivery.attempt_count,
                    "last_error": delivery.last_error,
                }
            )
            # In production: write to DLQ (SQS dead letter queue, separate DB table)
            # Trigger alert: PagerDuty / Slack notification
        else:
            delivery.next_attempt_at = self._calculate_next_attempt(delivery.attempt_count)
            logger.warning(
                "Webhook delivery failed, scheduled for retry",
                extra={
                    "event_id": delivery.event_id,
                    "endpoint": endpoint_url,
                    "attempt": delivery.attempt_count,
                    "max_attempts": delivery.max_attempts,
                    "next_attempt_at": delivery.next_attempt_at.isoformat(),
                    "error": delivery.last_error,
                }
            )
        
        return delivery
```

---

## Event-Driven vs Polling: Trade-offs {#event-driven-vs-polling}

```
FDE Context: This is a real architectural conversation you'll have with customers.
"Should we poll your API every minute, or can we subscribe to events?"
The answer has cost, reliability, and latency implications.
```

### Comparison

| Dimension | Polling | Webhooks (Event-Driven) |
|-----------|---------|------------------------|
| Latency | Up to poll interval (1-15 min) | Near-real-time (seconds) |
| API calls | Constant (wasteful when nothing changes) | Only on events |
| Complexity | Simple to implement | Requires public endpoint, signature verification |
| Reliability | Inherently retryable | Requires your own retry logic |
| Dependency | Your system drives | Sender must be reliable |
| Firewall/NAT | Works from behind firewall | You must be publicly reachable |
| Ordering | Natural (poll in order) | Can arrive out-of-order |
| Historical | Easy (query by date range) | Need event replay capability |

### When to use polling

- Vendor doesn't offer webhooks (common with legacy carriers)
- You're behind a firewall with no public endpoint
- You need eventual consistency, not real-time
- The polling interval matches your SLA (hourly file sync is fine for end-of-day reports)

### When to use webhooks

- Real-time requirements (customer wants order status in <30 seconds)
- High event volume (polling would generate too many API calls)
- Vendor charges per API call (polling is expensive)

### Hybrid pattern (the FDE's real-world answer)

```python
import asyncio
import logging
from datetime import datetime, UTC, timedelta
from typing import Optional

logger = logging.getLogger(__name__)


class HybridSyncOrchestrator:
    """
    Hybrid sync strategy:
    1. Webhooks for real-time incremental updates
    2. Periodic polling as reconciliation/catch-up
    
    Solves the key problem: webhooks can be missed (network blip, server restart).
    Polling ensures you eventually catch everything.
    
    This is the pattern used by Stripe, Shopify, and most mature platforms.
    """
    
    def __init__(
        self,
        api_client,
        storage,
        reconciliation_interval_hours: int = 1,
    ):
        self.api_client = api_client
        self.storage = storage
        self.reconciliation_interval = timedelta(hours=reconciliation_interval_hours)
        self._last_reconciliation: Optional[datetime] = None
    
    async def handle_webhook_event(self, event: dict) -> None:
        """
        Fast path: process real-time webhook event immediately.
        """
        event_type = event.get("type")
        event_data = event.get("data", {})
        
        logger.info(
            "Processing webhook event",
            extra={"event_type": event_type, "event_id": event.get("id")}
        )
        
        await self.storage.upsert(
            entity_type=event_type,
            data=event_data,
            source="webhook",
        )
    
    async def run_reconciliation(self) -> int:
        """
        Slow path: poll for changes since last reconciliation.
        Catches any events missed by webhooks.
        Returns count of records synced.
        """
        since = self._last_reconciliation or (datetime.now(UTC) - timedelta(hours=24))
        
        logger.info(
            "Starting reconciliation",
            extra={"since": since.isoformat()}
        )
        
        count = 0
        async for record in self.api_client.get_updated_since(since):
            await self.storage.upsert(
                entity_type="order",
                data=record,
                source="reconciliation",
            )
            count += 1
        
        self._last_reconciliation = datetime.now(UTC)
        logger.info(f"Reconciliation complete, synced {count} records")
        return count
    
    async def run_background_reconciliation_loop(self) -> None:
        """
        Background task that runs reconciliation on schedule.
        """
        while True:
            await asyncio.sleep(self.reconciliation_interval.total_seconds())
            try:
                await self.run_reconciliation()
            except Exception as e:
                logger.error(f"Reconciliation failed: {e}", exc_info=True)
                # Don't crash the loop — just log and continue
```

---

## The Duplicate Order Problem {#the-duplicate-order-problem}

> **FDE Context**: This is the #1 complaint you'll hear in customer escalations. "Our system created duplicate orders." Here's exactly how it happens and how to prevent it.

### How duplicates occur

```
Timeline of a duplicate creation:
1. Your system sends POST /shipments → carrier server takes 28 seconds to respond
2. Your httpx client has a 30-second read timeout
3. At 29s, timeout fires — you raise TimeoutException
4. You retry: POST /shipments again
5. Carrier server processed BOTH requests (it just responded slowly to the first)
6. Two shipments created, carrier bills you twice
```

### Prevention strategy: layered approach

```python
import uuid
import hashlib
import logging
from datetime import datetime, UTC
from typing import Optional

logger = logging.getLogger(__name__)


class IdempotentShipmentCreator:
    """
    Layered idempotency for shipment creation:
    
    Layer 1: Idempotency key — client-generated UUID sent with every request
    Layer 2: Database unique constraint — prevents duplicate records at DB level
    Layer 3: Pre-flight check — check if shipment already exists before creating
    
    The key insight: not all three are needed for every API, but you want
    at least two layers for critical operations like shipment creation.
    """
    
    def __init__(self, carrier_client, shipment_repository):
        self.carrier = carrier_client
        self.repo = shipment_repository
    
    def _generate_idempotency_key(
        self,
        order_id: str,
        carrier_code: str,
        service_code: str,
    ) -> str:
        """
        Generate a deterministic idempotency key.
        
        Why deterministic? If your process crashes and restarts, you want to
        generate the SAME key for the same logical operation. A random UUID
        generated fresh would bypass idempotency on the carrier's side.
        
        Base it on: order_id + carrier + service + date (day granularity is usually fine)
        """
        today = datetime.now(UTC).strftime("%Y-%m-%d")
        content = f"{order_id}:{carrier_code}:{service_code}:{today}"
        return hashlib.sha256(content.encode()).hexdigest()[:32]
    
    async def create_shipment(
        self,
        order_id: str,
        carrier_code: str,
        service_code: str,
        package_details: dict,
    ) -> dict:
        """
        Create a shipment with idempotency guarantees.
        Safe to call multiple times for the same order.
        """
        # Layer 3: Pre-flight check
        existing = await self.repo.find_shipment_by_order(order_id, carrier_code)
        if existing:
            logger.info(
                "Shipment already exists, returning existing",
                extra={"order_id": order_id, "shipment_id": existing["id"]}
            )
            return existing
        
        # Layer 1: Deterministic idempotency key
        idempotency_key = self._generate_idempotency_key(
            order_id, carrier_code, service_code
        )
        
        try:
            # The actual API call with idempotency key
            response = await self.carrier.post(
                "/shipments",
                json={
                    "order_id": order_id,
                    "carrier": carrier_code,
                    "service": service_code,
                    **package_details,
                },
                headers={"Idempotency-Key": idempotency_key},
            )
            
            shipment = response
            
            # Layer 2: Store with unique constraint on (order_id, carrier_code)
            # If a race condition caused two requests to get here simultaneously,
            # the DB unique constraint catches it
            try:
                await self.repo.insert_shipment(
                    shipment_id=shipment["id"],
                    order_id=order_id,
                    carrier_code=carrier_code,
                    idempotency_key=idempotency_key,
                    carrier_response=shipment,
                )
            except UniqueConstraintViolation:
                # Another process inserted while we were processing
                # Fetch and return the existing one
                logger.warning(
                    "Race condition on shipment insert, fetching existing",
                    extra={"order_id": order_id}
                )
                return await self.repo.find_shipment_by_order(order_id, carrier_code)
            
            return shipment
            
        except Exception as e:
            # On any error: don't create a new idempotency key on retry
            # The same key will be reused, so the carrier handles deduplication
            logger.error(
                "Shipment creation failed",
                extra={
                    "order_id": order_id,
                    "idempotency_key": idempotency_key,
                    "error": str(e),
                }
            )
            raise


class UniqueConstraintViolation(Exception):
    pass
```

---

## Common Failure Modes {#common-failure-modes}

### 1. Missing timeout → hanging integration

**Symptom**: Threads pile up, memory usage grows, eventually OOM.

**Diagnosis**:
```bash
# See hanging HTTP connections
ss -n state ESTABLISHED '( dport = :443 )'
# Check thread count
cat /proc/<pid>/status | grep Threads
```

**Fix**: Always set explicit timeouts. `httpx` without timeout config can hang indefinitely.

### 2. Logging parsed body instead of raw body for signature verification

**Symptom**: `verify_hmac_signature` always returns False even with correct secret.

**Root cause**: You parsed the JSON first, re-serialized it, whitespace changed, signature broken.

**Fix**: Call `await request.body()` before any parsing.

### 3. Retrying non-idempotent POST requests

**Symptom**: Duplicate shipments/orders/invoices in customer's system.

**Root cause**: Retry logic retries `POST /shipments` on timeout.

**Fix**: Use idempotency keys; on timeout, only retry if you have an idempotency key.

### 4. GraphQL returning 200 with errors that you treat as success

**Symptom**: Integration looks like it's working but data is missing/stale.

**Root cause**: GraphQL errors are in the body, not the HTTP status. You're not checking `body["errors"]`.

**Fix**: Always inspect `errors` key in GraphQL response body.

### 5. Thundering herd on retry

**Symptom**: After an outage, all your retry workers fire simultaneously and re-saturate the recovering server.

**Root cause**: No jitter in backoff.

**Fix**: `wait_exponential_jitter` from tenacity, or add `random.uniform(0, wait * 0.1)` manually.

### 6. Webhook endpoint not returning 200 fast enough

**Symptom**: Sender retries even though you did process the event; you get duplicates.

**Root cause**: Your webhook handler does synchronous DB writes/API calls before returning.

**Fix**: Return 200 immediately, process in background task or queue.

### 7. Using string comparison for HMAC (timing attack vulnerability)

**Symptom**: Technically works but vulnerable to timing-based forgery.

**Root cause**: `signature == expected` leaks timing information.

**Fix**: `hmac.compare_digest(signature, expected)`.

---

## Interview Angles {#interview-angles}

### Q: "Walk me through how you'd handle a scenario where we're sending webhooks to customers and we're not sure they're receiving them."

**Great answer**: "I'd instrument delivery at three levels. First, every outbound webhook gets a unique event ID logged with attempt count and endpoint URL. Second, I'd implement exponential backoff retries with a dead letter queue after N failures. Third, I'd expose a customer-facing webhook delivery log in the UI so they can see individual delivery attempts and response codes. The DLQ is key — permanently failed deliveries go there with full payload preserved, so a customer can replay them after fixing their endpoint. I'd also alert our team when DLQ depth exceeds a threshold, because that usually means a systematic problem with one customer's endpoint."

### Q: "What's the difference between at-least-once and exactly-once delivery, and which would you implement?"

**Great answer**: "Exactly-once is theoretically ideal but practically very hard — it requires distributed transactions or something like Kafka's transactional producer, and it has significant performance cost. For most webhook and queue systems I'd design for at-least-once delivery plus idempotent consumers. The consumer checks a unique event ID against a deduplication store before processing. This is simpler, more reliable, and the industry standard. The mental model is: make your processing idempotent, then deliver with at-least-once guarantee — the combination gives you the semantics of exactly-once without the complexity."

### Q: "A customer says their system is receiving duplicate order events from your platform. How do you debug it?"

**Great answer**: "First, I'd pull delivery logs for that customer's endpoint filtered to that order ID — how many delivery attempts were made? Were any of them duplicates of the same event ID, or were they genuinely different events? If same event ID delivered twice, my delivery deduplication has a bug. If different event IDs, I created the event twice — probably a bug in my event emission code where the same state change triggers the event multiple times. I'd check if the customer's endpoint was returning non-200 responses that caused retries, which then processed the event again on their side. I'd also check their idempotency handling — if they're not deduplicating on their end, even a single duplicate delivery from us is a problem."

### Q: "How would you design a polling integration with a carrier API that has a 1000 req/hour rate limit?"

**Great answer**: "1000/hour is 16.6/minute or roughly one request every 3.6 seconds. I'd implement a token bucket that replenishes at that rate, with the consumer waiting when the bucket is empty. I'd also watch for `X-RateLimit-Remaining` headers and proactively slow down as I approach the limit rather than hitting 429. For batch fetching, I'd use the carrier's bulk API if available — fetching 100 shipments in one call vs 100 individual calls makes the rate limit effectively 100x more useful. I'd also implement circuit breaker logic so that if I do hit rate limits, I back off exponentially rather than immediately retrying."

---

## Practice Exercise {#practice-exercise}

### Exercise: Build a Carrier Integration Client

**Objective**: Build a complete, production-grade integration client for a fictional carrier API.

**Requirements**:

1. **AsyncCarrierClient** (`carrier_client.py`):
   - Connection pool with max 20 connections
   - Connect timeout 5s, read timeout 30s
   - All requests include `X-API-Key` header
   - All POST requests support `Idempotency-Key` header
   - `get_shipment(tracking_number)` — fetch single shipment
   - `create_shipment(payload, idempotency_key)` — create shipment
   - `list_shipments(status, page_size, cursor)` — paginated list
   - Retry on 429/5xx with exponential backoff + jitter (max 3 attempts)
   - Log all requests at DEBUG level, errors at ERROR level

2. **WebhookReceiver** (`webhook_receiver.py`):
   - FastAPI endpoint `POST /webhooks/shipments`
   - Verify HMAC-SHA256 signature from `X-Carrier-Signature` header
   - Idempotency using in-memory store keyed on `X-Delivery-Id` header
   - Return 200 within 200ms — process in background
   - Background processing: log the event and write to a local SQLite DB

3. **Integration test** (`test_integration.py`):
   - Mock the carrier API using `respx` (httpx mock library)
   - Test: successful shipment creation
   - Test: retry on 503, success on second attempt
   - Test: duplicate webhook delivery is deduplicated
   - Test: invalid webhook signature returns 401

**Starter code skeleton**:

```python
# carrier_client.py skeleton
import httpx
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

class AsyncCarrierClient:
    def __init__(self, base_url: str, api_key: str):
        # TODO: initialize httpx.AsyncClient with proper timeouts and limits
        ...
    
    async def create_shipment(self, payload: dict, idempotency_key: str) -> dict:
        # TODO: implement with retry logic
        ...

# webhook_receiver.py skeleton
from fastapi import FastAPI, Request, BackgroundTasks, HTTPException

app = FastAPI()

@app.post("/webhooks/shipments")
async def receive_shipment_webhook(request: Request, background_tasks: BackgroundTasks):
    # TODO: verify signature, check idempotency, return 200 fast
    ...
```

**Evaluation criteria**:
- Does the client respect timeouts at all three levels (connect/read/write)?
- Is the idempotency key deterministic or random? (deterministic is better)
- Does the webhook receiver read raw bytes before parsing?
- Does it use `hmac.compare_digest` not string equality?
- Are background tasks truly asynchronous (response not blocked on processing)?
- Do logs include enough context to debug failures from logs alone?

---

## Related Files

- [Auth flows (OAuth2, mTLS, API Keys)](02-auth-flows.md) — authentication for your HTTP clients
- [Resilience patterns (retry, circuit breaker, rate limiting)](03-resilience-patterns.md) — deeper dive on retry and circuit breakers
- [Data mapping and transformation](04-data-mapping-transformation.md) — transforming payloads before sending
- [Message queues (SQS, Kafka)](06-message-queues.md) — queueing webhook processing tasks
