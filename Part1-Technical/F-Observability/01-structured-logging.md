# Structured Logging for Forward Deployment Engineers

> **FDE Principle**: When a customer says "can you find what happened to order #12345 on Tuesday?", the answer should be a 10-second query — not a 2-hour grep session. Structured logging is what makes that possible.

**Related files:**
- [02-distributed-tracing-otel.md](./02-distributed-tracing-otel.md) — Correlation IDs connect logging to tracing
- [04-log-aggregation.md](./04-log-aggregation.md) — Where these logs go and how to query them
- [03-metrics-prometheus-grafana.md](./03-metrics-prometheus-grafana.md) — Metrics complement logs for system health

---

## Table of Contents

1. [Why Structured Logging](#1-why-structured-logging)
2. [The structlog Library](#2-the-structlog-library)
3. [Standard Fields Every Log Must Have](#3-standard-fields-every-log-must-have)
4. [Correlation IDs](#4-correlation-ids)
5. [Log Levels: When to Use Each](#5-log-levels-when-to-use-each)
6. [Logging Sensitive Data](#6-logging-sensitive-data)
7. [Exception Logging](#7-exception-logging)
8. [Request/Response Logging Middleware](#8-requestresponse-logging-middleware)
9. [Full Working Example: Multi-Service Order Flow](#9-full-working-example-multi-service-order-flow)
10. [FDE Context: Before and After](#10-fde-context-before-and-after)
11. [Common Failure Modes](#11-common-failure-modes)
12. [Interview Angle](#12-interview-angle)
13. [Practice Exercise](#13-practice-exercise)

---

## 1. Why Structured Logging

### The Graveyard of Unstructured Logs

Here is what a typical Python service logs without structure:

```
INFO 2024-01-15 09:23:11 Processing order 12345 for customer ACME
WARNING 2024-01-15 09:23:12 Retry attempt 1 for order 12345
ERROR 2024-01-15 09:23:13 Failed to create shipment: Connection refused
INFO 2024-01-15 09:23:45 Processing order 12346 for customer Globex
INFO 2024-01-15 09:23:45 Order 12346 completed successfully
```

Now a customer calls on Monday morning and says: "Something went wrong with our orders on Friday afternoon — our 3PL says they never got shipment #X-9988. Can you find what happened?"

With unstructured logs, you are doing this:

```bash
# Searching through gigabytes of text for a shipment reference
grep -r "X-9988" /var/log/services/ 
# Maybe it's in a different format
grep -r "shipment.*9988\|9988.*shipment" /var/log/services/
# Trying to correlate across services
grep -B5 -A5 "9988" /var/log/order-service/2024-01-12.log
# Cross-referencing with the 3PL service logs
grep "9988" /var/log/tpl-connector/2024-01-12.log
```

**This takes hours.** The logs are in different formats across services. You cannot join them. You cannot aggregate them. You cannot run queries. You cannot sort by correlation ID to see the complete chain of events.

### What Structured Logs Give You

Structured logging means every log entry is a machine-readable JSON object with consistent fields:

```json
{
  "timestamp": "2024-01-12T14:23:11.432Z",
  "level": "error",
  "service": "order-service",
  "correlation_id": "req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c",
  "event": "shipment_creation_failed",
  "order_id": "12345",
  "shipment_ref": "X-9988",
  "customer_id": "acme-corp",
  "error": "Connection refused",
  "error_type": "ConnectionError",
  "duration_ms": 1823,
  "retry_count": 3
}
```

Now the same question — "what happened to shipment X-9988 on Friday?" — becomes a CloudWatch Insights query that runs in 3 seconds:

```sql
filter shipment_ref = "X-9988"
| sort @timestamp asc
| fields timestamp, level, service, event, error, duration_ms
```

Or you filter by correlation ID to see every single step the request took across all services:

```sql
filter correlation_id = "req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c"
| sort @timestamp asc
```

**Output:**
```
timestamp              service           event                          level
2024-01-12T14:23:10Z   api-gateway       request_received               info
2024-01-12T14:23:10Z   order-service     order_validation_started       info
2024-01-12T14:23:10Z   order-service     order_validation_passed        info
2024-01-12T14:23:11Z   order-service     shipment_creation_started      info
2024-01-12T14:23:11Z   tpl-connector     outbound_request_started       info
2024-01-12T14:23:13Z   tpl-connector     outbound_request_failed        error  ← here
2024-01-12T14:23:13Z   order-service     shipment_creation_failed       error
2024-01-12T14:23:13Z   order-service     order_marked_failed            warning
```

You just gave the customer a complete root-cause analysis in 30 seconds. That is the FDE superpower.

### The Business Case

- **MTTR (Mean Time to Resolve)**: Structured logs reduce debugging time from hours to minutes
- **On-call burden**: Engineers can answer customer questions without waking up the team
- **Customer trust**: "Here is exactly what happened" builds more trust than "we're looking into it"
- **Compliance**: Audit trails for "who did what, when" are queryable
- **Cost**: Cloud log aggregation pricing is often per GB — structured logs with filtering are cheaper to query than full-text scan

---

## 2. The structlog Library

`structlog` is the Python standard for structured logging. It wraps Python's standard `logging` module and adds:
- Bound loggers (attach context to a logger, not individual calls)
- Processor pipelines (transform log records through a chain of functions)
- JSON rendering for production
- Colored console rendering for development

### Installation

```bash
pip install structlog
# Optional but recommended for colored output in development
pip install colorama rich
```

### Basic Usage

```python
import structlog

log = structlog.get_logger()

log.info("order_received", order_id="12345", customer="acme-corp", total_amount=149.99)
log.warning("inventory_low", sku="WIDGET-001", quantity=3, reorder_threshold=10)
log.error("payment_failed", order_id="12345", error="insufficient_funds", amount=149.99)
```

### Bound Loggers: The Key Concept

A **bound logger** carries context that automatically appears in every subsequent log call. This eliminates the need to repeat `order_id=order_id` in every single log line inside a function.

```python
import structlog

log = structlog.get_logger()

def process_order(order_id: str, customer_id: str):
    # Bind context for this order — all subsequent logs include these fields
    log = structlog.get_logger().bind(
        order_id=order_id,
        customer_id=customer_id
    )
    
    log.info("order_processing_started")
    
    # Validate
    try:
        validate_order(order_id)
        log.info("order_validated")  # automatically includes order_id, customer_id
    except ValidationError as e:
        log.error("order_validation_failed", error=str(e))  # includes all bound fields
        raise
    
    # Create shipment
    shipment_ref = create_shipment(order_id)
    log = log.bind(shipment_ref=shipment_ref)  # add more context as it becomes available
    log.info("shipment_created")
    
    # Update status
    update_order_status(order_id, "shipped")
    log.info("order_completed", status="shipped")
```

Every single log line in this function automatically includes `order_id` and `customer_id`. When debugging, you can filter on either field and see the complete picture.

### Context Variables: Thread-Safe Request Context

For web services handling concurrent requests, Python's `contextvars` module provides per-request context. `structlog` integrates with this:

```python
import structlog
from contextvars import ContextVar
from typing import Optional

# Module-level context variable for the current request
_request_context: ContextVar[dict] = ContextVar('request_context', default={})

def bind_request_context(**kwargs):
    """Add fields to the current request's log context."""
    current = _request_context.get()
    _request_context.set({**current, **kwargs})

def get_request_context() -> dict:
    return _request_context.get()

# structlog processor that injects context variables
def inject_context_vars(logger, method, event_dict):
    """Processor: merge request context into every log entry."""
    context = _request_context.get()
    event_dict.update(context)
    return event_dict
```

### The Processor Pipeline: How structlog Works

`structlog` processes every log entry through a **pipeline** of processor functions. Each processor receives `(logger, method_name, event_dict)` and returns a modified `event_dict`. The final processor serializes and outputs the record.

```python
import structlog
import logging
import sys
from datetime import datetime, timezone

def configure_structlog(environment: str = "production"):
    """
    Configure structlog for the application.
    
    Call this once at application startup, before any logging occurs.
    
    Args:
        environment: "production" (JSON output) or "development" (colored console)
    """
    
    # Shared processors that run in both environments
    shared_processors = [
        # Add timestamp in ISO 8601 UTC format
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        
        # Add log level as a string field
        structlog.stdlib.add_log_level,
        
        # Add logger name (usually module name)
        structlog.stdlib.add_logger_name,
        
        # Inject per-request context variables
        inject_context_vars,
        
        # If exc_info=True was passed, format the exception
        structlog.processors.format_exc_info,
        
        # Handle UnicodeDecodeError and other encoding issues
        structlog.processors.UnicodeDecoder(),
    ]
    
    if environment == "development":
        # Pretty, colored console output for local development
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True)
        ]
    else:
        # JSON output for production (CloudWatch, Datadog, etc.)
        processors = shared_processors + [
            # Rename 'event' to 'message' for compatibility with some log systems
            # structlog.processors.EventRenamer("message"),
            structlog.processors.JSONRenderer()
        ]
    
    structlog.configure(
        processors=processors,
        
        # Use standard library logging as the underlying output mechanism
        wrapper_class=structlog.stdlib.BoundLogger,
        
        # Use context variables for request-scoped context
        context_class=structlog.contextvars.merge_contextvars,
        
        # Where to send the final output
        logger_factory=structlog.PrintLoggerFactory(sys.stdout),
        
        # Cache the logger on the class (performance optimization)
        cache_logger_on_first_use=True,
    )
    
    # Also configure the stdlib logging to use structlog
    # This ensures third-party libraries that use stdlib logging 
    # also get structured output
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=logging.INFO,
    )
    logging.getLogger("uvicorn.access").disabled = True  # We handle request logging ourselves
```

### Custom Processors

Processors are just functions — you can write your own to add any field:

```python
import os

def add_service_info(logger, method, event_dict):
    """Add service name and version to every log entry."""
    event_dict["service"] = os.environ.get("SERVICE_NAME", "unknown")
    event_dict["version"] = os.environ.get("SERVICE_VERSION", "unknown")
    event_dict["environment"] = os.environ.get("ENVIRONMENT", "unknown")
    return event_dict

def scrub_sensitive_fields(logger, method, event_dict):
    """Remove sensitive fields before logging."""
    sensitive_keys = {"password", "token", "api_key", "secret", "credit_card", "ssn", "cvv"}
    for key in list(event_dict.keys()):
        if key.lower() in sensitive_keys:
            event_dict[key] = "[REDACTED]"
    return event_dict

def add_host_info(logger, method, event_dict):
    """Add hostname for distinguishing between instances."""
    import socket
    event_dict["host"] = socket.gethostname()
    return event_dict
```

---

## 3. Standard Fields Every Log Must Have

Every single log entry in every service should have these fields. This is your logging contract.

### Required Fields

| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| `timestamp` | ISO 8601 UTC | `2024-01-15T14:23:11.432Z` | When it happened |
| `level` | string | `info`, `error` | Severity |
| `service` | string | `order-service` | Which service |
| `event` | string | `order_created` | What happened (snake_case verb) |
| `correlation_id` | UUID string | `req-7f3a9b2c...` | Links request across services |

### Recommended Fields

| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| `duration_ms` | float | `142.3` | How long it took |
| `error` | string | `Connection refused` | Error message (on error/warning) |
| `error_type` | string | `ConnectionError` | Exception class name |
| `user_id` | string | `user-456` | Who triggered it |
| `tenant_id` | string | `acme-corp` | Multi-tenant context |
| `request_path` | string | `/api/v1/orders` | HTTP path |
| `request_method` | string | `POST` | HTTP method |
| `status_code` | int | `200` | HTTP response code |
| `host` | string | `ip-10-0-1-42` | Which instance |
| `version` | string | `1.4.2` | Service version for debugging deploys |

### Event Naming Convention

Use **snake_case verb phrases** that describe what happened:
- `order_received` — not "order" or "got order"
- `shipment_creation_failed` — not "error" or "failed"
- `inventory_check_started` / `inventory_check_completed` — bracket long operations
- `webhook_delivered` / `webhook_delivery_failed` — specific outcomes

**The "event as verb" rule**: Someone should be able to read just the `event` field and understand what the system did. Avoid generic events like `"error"` or `"processing"`.

### Full Standard Log Entry

```json
{
  "timestamp": "2024-01-15T14:23:11.432Z",
  "level": "info",
  "service": "order-service",
  "version": "1.4.2",
  "environment": "production",
  "host": "ip-10-0-1-42",
  "correlation_id": "req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c",
  "event": "shipment_created",
  "order_id": "ORD-12345",
  "customer_id": "acme-corp",
  "shipment_ref": "X-9988",
  "carrier": "fedex",
  "duration_ms": 342.1,
  "request_id": "req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c"
}
```

---

## 4. Correlation IDs

### What Is a Correlation ID?

A correlation ID (also called request ID, trace ID, or request correlation ID) is a unique identifier — typically a UUID — that is:

1. Generated when a request first enters your system (at the API gateway or the first service it touches)
2. Attached to every log entry for the duration of that request
3. Propagated to every downstream service via HTTP headers
4. Returned in API responses so clients can report it in support tickets

When a request touches 5 services and generates 40 log entries, **all 40 entries share the same correlation ID**. Filtering on that ID gives you a complete timeline of everything that happened.

### Generating and Propagating Correlation IDs

The convention is to use the HTTP header `X-Request-ID` (or `X-Correlation-ID`). The OpenTelemetry standard uses `traceparent`. For maximum compatibility with both observability tools and simple logging, use both.

```python
import uuid
from fastapi import FastAPI, Request, Response
from contextvars import ContextVar
import structlog

# Context variable — one per request, isolated between concurrent requests
correlation_id_var: ContextVar[str] = ContextVar('correlation_id', default='')

log = structlog.get_logger()

class CorrelationIdMiddleware:
    """
    FastAPI middleware that:
    1. Extracts or generates a correlation ID for every request
    2. Stores it in a ContextVar so all code in this request can access it
    3. Binds it to structlog so all logs in this request include it
    4. Returns it in the response header
    """
    
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send):
        if scope["type"] not in ("http", "websocket"):
            await self.app(scope, receive, send)
            return
        
        request = Request(scope, receive)
        
        # Prefer existing correlation ID (from upstream gateway or client)
        # Fall back to generating a new one
        correlation_id = (
            request.headers.get("X-Request-ID") or
            request.headers.get("X-Correlation-ID") or
            f"req-{uuid.uuid4()}"
        )
        
        # Store in context variable — thread/task-safe
        token = correlation_id_var.set(correlation_id)
        
        # Bind to structlog so every log in this request includes it
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            request_path=request.url.path,
            request_method=request.method,
        )
        
        # Intercept the response to add the correlation ID header
        async def send_with_correlation_id(message):
            if message["type"] == "http.response.start":
                headers = list(message.get("headers", []))
                headers.append(
                    (b"x-request-id", correlation_id.encode())
                )
                message = {**message, "headers": headers}
            await send(message)
        
        try:
            await self.app(scope, receive, send_with_correlation_id)
        finally:
            # Clean up context — important for thread pool reuse
            correlation_id_var.reset(token)
            structlog.contextvars.unbind_contextvars(
                "correlation_id", "request_path", "request_method"
            )


def get_correlation_id() -> str:
    """Get the current request's correlation ID from anywhere in the call stack."""
    return correlation_id_var.get()
```

### Propagating to Downstream Services

When your service calls another service, include the correlation ID so it also appears in that service's logs:

```python
import httpx
from typing import Optional

class CorrelationAwareHttpClient:
    """
    HTTP client that automatically propagates correlation IDs to downstream services.
    
    Every outbound request includes the current request's correlation ID
    in the X-Request-ID header, so downstream services can include it in their logs.
    """
    
    def __init__(self, base_url: str, service_name: str):
        self.base_url = base_url
        self.service_name = service_name
        self._client = httpx.AsyncClient(base_url=base_url)
    
    def _get_headers(self, extra_headers: Optional[dict] = None) -> dict:
        correlation_id = get_correlation_id()
        headers = {
            "X-Request-ID": correlation_id,
            "X-Source-Service": "order-service",  # identify the caller
        }
        if extra_headers:
            headers.update(extra_headers)
        return headers
    
    async def post(self, path: str, json: dict, **kwargs) -> httpx.Response:
        headers = self._get_headers(kwargs.pop("headers", None))
        log.info(
            "outbound_request_started",
            target_service=self.service_name,
            path=path,
            method="POST"
        )
        
        import time
        start = time.monotonic()
        try:
            response = await self._client.post(path, json=json, headers=headers, **kwargs)
            duration_ms = (time.monotonic() - start) * 1000
            
            log.info(
                "outbound_request_completed",
                target_service=self.service_name,
                path=path,
                status_code=response.status_code,
                duration_ms=round(duration_ms, 2)
            )
            return response
        except Exception as e:
            duration_ms = (time.monotonic() - start) * 1000
            log.error(
                "outbound_request_failed",
                target_service=self.service_name,
                path=path,
                error=str(e),
                error_type=type(e).__name__,
                duration_ms=round(duration_ms, 2)
            )
            raise
    
    async def aclose(self):
        await self._client.aclose()

# Usage
tpl_client = CorrelationAwareHttpClient(
    base_url="https://tpl.example.com",
    service_name="tpl-connector"
)

async def create_shipment(order_id: str, items: list) -> dict:
    response = await tpl_client.post(
        "/shipments",
        json={"order_id": order_id, "items": items}
    )
    response.raise_for_status()
    return response.json()
```

### Returning Correlation ID in Responses

Always return the correlation ID in the response so clients can include it in support requests:

```
HTTP/1.1 422 Unprocessable Entity
X-Request-ID: req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c
Content-Type: application/json

{
  "error": "validation_failed",
  "message": "Order quantity must be positive",
  "request_id": "req-7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c"
}
```

When a customer says "order #12345 failed on Tuesday at 2pm", you ask: "Did you get a request_id in the response?" If yes, you have the exact trace in 10 seconds. Train your customers to log these IDs on their side.

---

## 5. Log Levels: When to Use Each

This seems obvious but most engineers get it wrong. Over-logging WARNING and under-logging ERROR is the most common mistake.

### The Rules

| Level | When to Use | Example |
|-------|-------------|---------|
| `DEBUG` | Fine-grained info useful only when debugging. Never in production by default. | SQL queries, function entry/exit, variable values |
| `INFO` | Normal business operations. Read by humans triaging incidents. | Order created, payment processed, sync completed |
| `WARNING` | Unexpected but handled. System continues. Worth investigating proactively. | Retry succeeded on attempt 2, deprecated API used, queue depth high |
| `ERROR` | Something failed. A business operation could not complete. Requires attention. | Payment failed, shipment creation failed, webhook delivery failed |
| `CRITICAL` | Service is impaired or down. Page someone. | DB unreachable, out of memory, message queue dead |

### Common Mistakes

**Over-logging WARNING:**
```python
# WRONG — this is just normal operation
for item in items:
    if item.quantity == 0:
        log.warning("item has zero quantity", sku=item.sku)  # noisy, not actionable

# RIGHT — just skip and note at debug level, or handle silently
for item in items:
    if item.quantity == 0:
        log.debug("skipping zero-quantity item", sku=item.sku)
        continue
```

**Under-logging ERROR:**
```python
# WRONG — this silently swallows a real error
try:
    send_email_notification(order_id)
except Exception:
    pass  # "emails are best-effort"

# RIGHT — log it even if you're swallowing it
try:
    send_email_notification(order_id)
except Exception as e:
    log.error(
        "email_notification_failed",
        order_id=order_id,
        error=str(e),
        error_type=type(e).__name__
    )
    # Still swallowing, but now it's visible
```

**Logging at ERROR for expected conditions:**
```python
# WRONG — not found is not an error if you're checking existence
try:
    order = get_order(order_id)
except OrderNotFound:
    log.error("order not found", order_id=order_id)  # creates false alert noise
    return None

# RIGHT — not found is INFO or WARNING depending on context
try:
    order = get_order(order_id)
except OrderNotFound:
    log.info("order not found, returning 404", order_id=order_id)
    raise HTTPException(status_code=404, detail="Order not found")
```

### Production Log Level Strategy

```python
import os

def get_log_level() -> str:
    """
    Production: INFO (debug logs are too noisy and expensive)
    Development: DEBUG (see everything)
    
    Override with LOG_LEVEL env var for temporary debugging in production.
    """
    return os.environ.get("LOG_LEVEL", "INFO").upper()
```

**FDE note**: When debugging a customer issue in production, temporarily set `LOG_LEVEL=DEBUG` on one instance, reproduce the issue, then set it back. Don't leave DEBUG on in production — costs money and creates noise.

---

## 6. Logging Sensitive Data

### The Rule: Never Log Secrets, Tokens, or PII

This seems obvious but happens constantly. Common accidents:
- Logging the entire request body (which contains payment card numbers)
- Logging headers (which contain `Authorization: Bearer <token>`)
- Logging the config dict (which contains database passwords)
- Logging a user object (which contains email, SSN, etc.)

### The Scrubber Pattern

```python
from typing import Any
import re

# Fields that should NEVER appear in logs
SENSITIVE_FIELD_NAMES = frozenset({
    "password", "passwd", "pwd",
    "token", "access_token", "refresh_token", "id_token",
    "api_key", "apikey", "api_secret",
    "secret", "private_key", "signing_key",
    "credit_card", "card_number", "cvv", "cvc",
    "ssn", "social_security",
    "authorization",  # HTTP header value
    "x-api-key",      # HTTP header value
})

# Regex patterns for values that look sensitive
SENSITIVE_VALUE_PATTERNS = [
    re.compile(r'^[A-Za-z0-9+/]{40,}={0,2}$'),  # Base64-encoded secrets
    re.compile(r'^[0-9]{13,19}$'),               # Credit card numbers
    re.compile(r'^\d{3}-\d{2}-\d{4}$'),          # SSN
]

def scrub_sensitive_data(data: Any, depth: int = 0) -> Any:
    """
    Recursively scrub sensitive fields from a dict/list before logging.
    
    Usage:
        log.info("request_received", payload=scrub_sensitive_data(request_body))
    """
    if depth > 10:  # Prevent infinite recursion on circular structures
        return "[MAX_DEPTH]"
    
    if isinstance(data, dict):
        return {
            key: "[REDACTED]" if key.lower() in SENSITIVE_FIELD_NAMES
                 else scrub_sensitive_data(value, depth + 1)
            for key, value in data.items()
        }
    elif isinstance(data, (list, tuple)):
        return [scrub_sensitive_data(item, depth + 1) for item in data]
    elif isinstance(data, str):
        # Check if the value itself looks sensitive
        for pattern in SENSITIVE_VALUE_PATTERNS:
            if pattern.match(data):
                return "[REDACTED]"
        return data
    else:
        return data


def scrub_headers(headers: dict) -> dict:
    """Scrub sensitive HTTP headers before logging."""
    sensitive_headers = {"authorization", "x-api-key", "cookie", "set-cookie"}
    return {
        key: "[REDACTED]" if key.lower() in sensitive_headers else value
        for key, value in headers.items()
    }


# Structlog processor version — runs in the pipeline
def scrub_log_fields(logger, method, event_dict):
    """
    Structlog processor that scrubs sensitive fields from every log entry.
    Add this to your processor chain before JSONRenderer.
    """
    return scrub_sensitive_data(event_dict)
```

### PII Handling Strategy

For PII (Personally Identifiable Information), prefer logging IDs over values:

```python
# WRONG — logs customer's actual email
log.info("customer_updated", email="john.doe@example.com", old_email="john@old.com")

# RIGHT — log the ID, not the PII
log.info("customer_email_updated", customer_id="cust-456", updated_fields=["email"])

# If you must log PII for debugging, use a hash
import hashlib
def hash_pii(value: str) -> str:
    """Pseudonymize PII for logging — preserves searchability without exposing data."""
    return "pii-" + hashlib.sha256(value.encode()).hexdigest()[:12]

log.info("customer_updated", 
         customer_id_hash=hash_pii("john.doe@example.com"),
         updated_fields=["email"])
```

### Request Body Logging

```python
async def log_request_body(request: Request) -> None:
    """
    Log request body safely — scrub sensitive fields, truncate large payloads.
    Only call this in DEBUG level or for specific diagnostic paths.
    """
    try:
        body = await request.json()
        scrubbed = scrub_sensitive_data(body)
        
        # Truncate large payloads — don't log 10MB EDI files
        body_str = str(scrubbed)
        if len(body_str) > 2000:
            body_str = body_str[:2000] + "...[TRUNCATED]"
        
        log.debug("request_body", body=body_str)
    except Exception:
        log.debug("request_body_unparseable")
```

---

## 7. Exception Logging

### Always Log with Stack Trace

```python
import structlog

log = structlog.get_logger()

# WRONG — loses the stack trace
try:
    result = process_order(order_id)
except Exception as e:
    log.error("processing failed", error=str(e))  # no traceback!

# RIGHT — include exc_info for full stack trace
try:
    result = process_order(order_id)
except Exception as e:
    log.error(
        "order_processing_failed",
        order_id=order_id,
        error=str(e),
        error_type=type(e).__name__,
        exc_info=True  # includes full stack trace in the log entry
    )
    raise  # re-raise unless you're handling it

# ALSO RIGHT — log.exception() is shorthand for log.error(..., exc_info=True)
try:
    result = process_order(order_id)
except Exception as e:
    log.exception("order_processing_failed", order_id=order_id)
    raise
```

### The Stack Trace in JSON Logs

With `exc_info=True` and the `format_exc_info` processor, your JSON log entry includes:

```json
{
  "timestamp": "2024-01-15T14:23:11.432Z",
  "level": "error",
  "service": "order-service",
  "event": "order_processing_failed",
  "order_id": "ORD-12345",
  "error": "Connection refused",
  "error_type": "ConnectionError",
  "exception": "Traceback (most recent call last):\n  File \"/app/services/order.py\", line 87, in process_order\n    shipment = await create_shipment(order_id)\n  File \"/app/services/shipment.py\", line 43, in create_shipment\n    response = await client.post(\"/shipments\", json=payload)\nConnectionError: Connection refused"
}
```

The entire stack trace is a string in the JSON — it's fully searchable and parseable by log aggregation tools.

### Structured Error Taxonomy

For integration services, establish error categories:

```python
from enum import Enum

class ErrorCategory(str, Enum):
    # Permanent errors — do not retry
    VALIDATION = "validation_error"          # Bad input data
    AUTH = "auth_error"                      # Authentication/authorization failed
    NOT_FOUND = "not_found"                  # Resource doesn't exist
    CONFLICT = "conflict"                    # Duplicate, version conflict
    
    # Transient errors — retry with backoff
    UPSTREAM_UNAVAILABLE = "upstream_unavailable"  # 503, connection refused
    UPSTREAM_TIMEOUT = "upstream_timeout"          # Request timed out
    RATE_LIMITED = "rate_limited"                  # 429
    
    # Unknown — investigate
    UNEXPECTED = "unexpected_error"

def log_error(
    event: str,
    exception: Exception,
    error_category: ErrorCategory,
    **context
) -> None:
    """Standardized error logging with category for alert routing."""
    log.error(
        event,
        error=str(exception),
        error_type=type(exception).__name__,
        error_category=error_category.value,
        retriable=error_category in {
            ErrorCategory.UPSTREAM_UNAVAILABLE,
            ErrorCategory.UPSTREAM_TIMEOUT,
            ErrorCategory.RATE_LIMITED,
        },
        exc_info=True,
        **context
    )
```

---

## 8. Request/Response Logging Middleware

Every HTTP request to your service should be logged with: method, path, status code, duration, and correlation ID. This gives you the traffic picture — how many requests, which endpoints, what status codes, how fast.

```python
import time
import uuid
from typing import Callable
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import structlog

log = structlog.get_logger()

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """
    Logs every HTTP request with:
    - Correlation ID (generated or extracted from X-Request-ID header)
    - Method, path, query string
    - Status code
    - Duration in milliseconds
    - Request/response size
    
    Also binds correlation_id to structlog context so all downstream
    logs within this request automatically include it.
    """
    
    # Paths to exclude from logging (health checks, etc.)
    EXCLUDED_PATHS = {"/health", "/healthz", "/ready", "/metrics", "/favicon.ico"}
    
    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Skip health checks — they're noisy and not useful
        if request.url.path in self.EXCLUDED_PATHS:
            return await call_next(request)
        
        # Generate or extract correlation ID
        correlation_id = (
            request.headers.get("X-Request-ID") or
            request.headers.get("X-Correlation-ID") or
            f"req-{uuid.uuid4()}"
        )
        
        # Bind to structlog context — all logs in this request get this
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            request_method=request.method,
            request_path=request.url.path,
        )
        
        # Also store in ContextVar for non-structlog code
        correlation_id_var.set(correlation_id)
        
        # Log request start (useful for long-running requests)
        request_start = time.monotonic()
        
        log.info(
            "request_started",
            query_params=str(request.query_params) if request.query_params else None,
            content_length=request.headers.get("content-length"),
            user_agent=request.headers.get("user-agent"),
        )
        
        # Process the request
        try:
            response = await call_next(request)
        except Exception as e:
            duration_ms = round((time.monotonic() - request_start) * 1000, 2)
            log.error(
                "request_unhandled_exception",
                error=str(e),
                error_type=type(e).__name__,
                duration_ms=duration_ms,
                exc_info=True,
            )
            raise
        
        # Calculate duration
        duration_ms = round((time.monotonic() - request_start) * 1000, 2)
        
        # Determine log level based on status code
        status_code = response.status_code
        if status_code >= 500:
            log_fn = log.error
            event = "request_failed_server_error"
        elif status_code >= 400:
            log_fn = log.warning
            event = "request_failed_client_error"
        else:
            log_fn = log.info
            event = "request_completed"
        
        log_fn(
            event,
            status_code=status_code,
            duration_ms=duration_ms,
            response_size=response.headers.get("content-length"),
        )
        
        # Add correlation ID to response headers
        response.headers["X-Request-ID"] = correlation_id
        
        return response


# Complete FastAPI app setup
def create_app() -> FastAPI:
    """Create and configure the FastAPI application with all middleware."""
    
    # Configure structlog at startup
    configure_structlog(environment=os.environ.get("ENVIRONMENT", "production"))
    
    app = FastAPI(
        title="Order Service",
        version="1.4.2",
    )
    
    # Add middleware (order matters — first added = outermost = first to run)
    app.add_middleware(RequestLoggingMiddleware)
    
    return app

app = create_app()
```

---

## 9. Full Working Example: Multi-Service Order Flow

This is the complete picture: an order comes in, hits the order service, which calls the 3PL connector, which calls the 3PL API. Every step is logged with correlation IDs.

### Project Structure

```
order-service/
├── main.py
├── config.py
├── middleware.py
├── models.py
├── services/
│   ├── order_service.py
│   └── tpl_connector.py
└── requirements.txt
```

### `config.py` — Structlog Configuration

```python
# config.py
import os
import sys
import logging
import structlog
from contextvars import ContextVar
from typing import Optional

# Global context variable for correlation ID
correlation_id_var: ContextVar[str] = ContextVar('correlation_id', default='no-correlation-id')

SERVICE_NAME = os.environ.get("SERVICE_NAME", "order-service")
SERVICE_VERSION = os.environ.get("SERVICE_VERSION", "unknown")
ENVIRONMENT = os.environ.get("ENVIRONMENT", "production")


def add_service_context(logger, method, event_dict):
    """Add service-level fields to every log entry."""
    event_dict.setdefault("service", SERVICE_NAME)
    event_dict.setdefault("version", SERVICE_VERSION)
    event_dict.setdefault("environment", ENVIRONMENT)
    return event_dict


def scrub_sensitive_fields(logger, method, event_dict):
    """Redact sensitive fields."""
    sensitive = {"password", "token", "api_key", "secret", "authorization"}
    for key in list(event_dict.keys()):
        if key.lower() in sensitive:
            event_dict[key] = "[REDACTED]"
    return event_dict


def configure_logging():
    """Configure structlog for the application. Call once at startup."""
    
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.stdlib.add_log_level,
        add_service_context,
        scrub_sensitive_fields,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
    ]
    
    if ENVIRONMENT in ("development", "local"):
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True, exception_formatter=structlog.dev.plain_traceback)
        ]
    else:
        processors = shared_processors + [
            structlog.processors.JSONRenderer()
        ]
    
    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(sys.stdout),
        cache_logger_on_first_use=True,
    )
    
    # Suppress noisy third-party loggers
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("httpcore").setLevel(logging.WARNING)
```

### `middleware.py` — Correlation ID + Request Logging

```python
# middleware.py
import time
import uuid
import structlog
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from config import correlation_id_var

log = structlog.get_logger()

EXCLUDED_PATHS = {"/health", "/metrics", "/favicon.ico"}

class CorrelationAndLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.url.path in EXCLUDED_PATHS:
            return await call_next(request)
        
        correlation_id = (
            request.headers.get("x-request-id") or
            request.headers.get("x-correlation-id") or
            f"req-{uuid.uuid4().hex[:16]}"
        )
        
        # Set in ContextVar (for non-structlog code)
        correlation_id_var.set(correlation_id)
        
        # Set in structlog context (for all log calls)
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            method=request.method,
            path=request.url.path,
        )
        
        start = time.monotonic()
        response = None
        
        try:
            response = await call_next(request)
            return response
        except Exception as exc:
            log.error("unhandled_exception", 
                     error=str(exc), 
                     error_type=type(exc).__name__,
                     exc_info=True)
            raise
        finally:
            duration_ms = round((time.monotonic() - start) * 1000, 2)
            status_code = response.status_code if response else 500
            
            log_method = (log.error if status_code >= 500 else
                         log.warning if status_code >= 400 else
                         log.info)
            
            log_method(
                "request_finished",
                status_code=status_code,
                duration_ms=duration_ms,
            )
            
            if response:
                response.headers["x-request-id"] = correlation_id
```

### `services/order_service.py` — Business Logic with Logging

```python
# services/order_service.py
import time
import structlog
from dataclasses import dataclass
from typing import List, Optional
from services.tpl_connector import TplConnector, ShipmentRequest

log = structlog.get_logger()

@dataclass
class OrderItem:
    sku: str
    quantity: int
    unit_price: float

@dataclass  
class Order:
    order_id: str
    customer_id: str
    items: List[OrderItem]
    shipping_address: dict

class OrderService:
    def __init__(self, tpl_connector: TplConnector):
        self.tpl = tpl_connector
    
    async def process_order(self, order: Order) -> dict:
        """
        Main order processing flow with full structured logging.
        
        Every step logs what happened. On failure, the correlation_id
        (set in middleware) links all these logs together.
        """
        # Bind order context — all subsequent logs include these fields
        bound_log = log.bind(order_id=order.order_id, customer_id=order.customer_id)
        
        bound_log.info("order_processing_started", item_count=len(order.items))
        start = time.monotonic()
        
        # Step 1: Validate
        try:
            self._validate_order(order)
            bound_log.info("order_validated")
        except ValueError as e:
            bound_log.warning(
                "order_validation_failed",
                error=str(e),
                duration_ms=round((time.monotonic() - start) * 1000, 2)
            )
            raise
        
        # Step 2: Calculate totals
        total_value = sum(item.quantity * item.unit_price for item in order.items)
        bound_log.info("order_totals_calculated", total_value=total_value, currency="USD")
        
        # Step 3: Create shipment with 3PL
        shipment_request = ShipmentRequest(
            reference=f"SHIP-{order.order_id}",
            customer_id=order.customer_id,
            items=[{"sku": i.sku, "qty": i.quantity} for i in order.items],
            address=order.shipping_address,
        )
        
        try:
            shipment = await self.tpl.create_shipment(shipment_request)
            bound_log = bound_log.bind(
                shipment_id=shipment["id"],
                shipment_ref=shipment["reference"],
                carrier=shipment.get("carrier")
            )
            bound_log.info("shipment_created")
        except Exception as e:
            bound_log.error(
                "shipment_creation_failed",
                error=str(e),
                error_type=type(e).__name__,
                duration_ms=round((time.monotonic() - start) * 1000, 2),
                exc_info=True,
            )
            raise
        
        # Step 4: Update order status
        total_duration_ms = round((time.monotonic() - start) * 1000, 2)
        bound_log.info(
            "order_processing_completed",
            status="awaiting_shipment",
            total_duration_ms=total_duration_ms,
        )
        
        return {
            "order_id": order.order_id,
            "status": "awaiting_shipment",
            "shipment_id": shipment["id"],
            "shipment_ref": shipment["reference"],
        }
    
    def _validate_order(self, order: Order) -> None:
        if not order.items:
            raise ValueError("Order must have at least one item")
        for item in order.items:
            if item.quantity <= 0:
                raise ValueError(f"Item {item.sku} has invalid quantity: {item.quantity}")
            if item.unit_price < 0:
                raise ValueError(f"Item {item.sku} has negative price")
```

### `services/tpl_connector.py` — Downstream Service Call

```python
# services/tpl_connector.py
import time
import structlog
import httpx
from dataclasses import dataclass
from config import correlation_id_var

log = structlog.get_logger()

@dataclass
class ShipmentRequest:
    reference: str
    customer_id: str
    items: list
    address: dict

class TplConnector:
    """
    Client for the 3PL (Third-Party Logistics) API.
    
    All requests include the current correlation ID so the 3PL's logs
    (if they support it) can be correlated with ours.
    """
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        # Don't log api_key — the scrubber handles it but let's be explicit
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers={"X-API-Key": api_key},  # api_key in header, not logged directly
            timeout=30.0,
        )
    
    def _get_headers(self) -> dict:
        return {
            "X-Request-ID": correlation_id_var.get(),
            "X-Source-Service": "order-service",
        }
    
    async def create_shipment(self, request: ShipmentRequest) -> dict:
        """Create a shipment in the 3PL system."""
        
        payload = {
            "reference": request.reference,
            "customer_id": request.customer_id,
            "items": request.items,
            "ship_to": request.address,
        }
        
        log.info(
            "tpl_create_shipment_started",
            reference=request.reference,
            item_count=len(request.items),
            target_service="tpl-api",
        )
        
        start = time.monotonic()
        
        try:
            response = await self._client.post(
                "/api/v2/shipments",
                json=payload,
                headers=self._get_headers(),
            )
            duration_ms = round((time.monotonic() - start) * 1000, 2)
            
            if response.status_code == 409:
                # Duplicate — idempotent, this is fine
                log.info(
                    "tpl_shipment_already_exists",
                    reference=request.reference,
                    status_code=409,
                    duration_ms=duration_ms,
                )
                return response.json()
            
            response.raise_for_status()
            
            result = response.json()
            log.info(
                "tpl_create_shipment_completed",
                reference=request.reference,
                shipment_id=result.get("id"),
                status_code=response.status_code,
                duration_ms=duration_ms,
            )
            return result
            
        except httpx.TimeoutException as e:
            duration_ms = round((time.monotonic() - start) * 1000, 2)
            log.error(
                "tpl_create_shipment_timeout",
                reference=request.reference,
                timeout_seconds=30,
                duration_ms=duration_ms,
                error_type="TimeoutException",
            )
            raise
        
        except httpx.HTTPStatusError as e:
            duration_ms = round((time.monotonic() - start) * 1000, 2)
            log.error(
                "tpl_create_shipment_http_error",
                reference=request.reference,
                status_code=e.response.status_code,
                response_body=e.response.text[:500],  # truncate
                duration_ms=duration_ms,
                error_type="HTTPStatusError",
            )
            raise
        
        except Exception as e:
            duration_ms = round((time.monotonic() - start) * 1000, 2)
            log.error(
                "tpl_create_shipment_unexpected_error",
                reference=request.reference,
                error=str(e),
                error_type=type(e).__name__,
                duration_ms=duration_ms,
                exc_info=True,
            )
            raise
```

### `main.py` — The FastAPI App

```python
# main.py
import os
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import List
import structlog

from config import configure_logging
from middleware import CorrelationAndLoggingMiddleware
from services.order_service import OrderService, Order, OrderItem
from services.tpl_connector import TplConnector

# Initialize logging FIRST before any other imports use loggers
configure_logging()

log = structlog.get_logger()
app = FastAPI()
app.add_middleware(CorrelationAndLoggingMiddleware)

# Initialize services
tpl = TplConnector(
    base_url=os.environ["TPL_API_URL"],
    api_key=os.environ["TPL_API_KEY"],
)
order_svc = OrderService(tpl_connector=tpl)


class CreateOrderRequest(BaseModel):
    order_id: str
    customer_id: str
    items: List[dict]
    shipping_address: dict


@app.post("/api/v1/orders")
async def create_order(request: CreateOrderRequest):
    order = Order(
        order_id=request.order_id,
        customer_id=request.customer_id,
        items=[OrderItem(**item) for item in request.items],
        shipping_address=request.shipping_address,
    )
    
    try:
        result = await order_svc.process_order(order)
        return result
    except ValueError as e:
        raise HTTPException(status_code=422, detail=str(e))
    except Exception as e:
        log.error("order_endpoint_error", error=str(e), exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")


@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## 10. FDE Context: Before and After

### The Customer Call

**Scenario**: It's 9am Monday. ACME Corp's operations manager calls: "We had a bunch of orders fail on Friday afternoon around 3pm EST. Order IDs: 12344, 12345, 12346, 12347. Can you find out what happened? We're getting calls from customers saying their orders weren't shipped."

### Without Structured Logging (The Old Way)

```bash
# SSH into the production server
ssh prod-server-01

# What even is the log format?
tail -100 /var/log/order-service/app.log
# 2024-01-12 15:01:23 INFO Processing order
# 2024-01-12 15:01:24 ERROR Failed to create shipment

# Try to grep for the order IDs
grep "12345" /var/log/order-service/app.log
# 2024-01-12 15:01:23 INFO Processing order 12345
# 2024-01-12 15:01:24 ERROR Failed
# (no context about WHY it failed)

# The TPL connector is a different service with different log format
ssh prod-server-02
grep "12345" /var/log/tpl-connector/app.log
# 2024-01-12 15:01:24 ERROR Connection refused to tpl.example.com

# But which order caused that? Can't tell.
# Try to manually correlate timestamps...
# This takes 2 hours.
```

**Result**: You spend 2 hours, find a vague "connection refused" error, still don't know which orders are in what state. Customer is frustrated.

### With Structured Logging (The New Way)

Open CloudWatch Logs Insights (or Datadog, or Grafana Loki). Run this query:

```sql
-- Find everything about these specific orders on Friday
filter order_id in ["12344", "12345", "12346", "12347"]
    and timestamp >= "2024-01-12T19:00:00Z"
    and timestamp <= "2024-01-12T21:00:00Z"
| sort @timestamp asc
| fields timestamp, service, event, order_id, error, error_type, duration_ms, correlation_id
```

Output (in 3 seconds):

```
timestamp              service           event                        order_id  error                  duration_ms
2024-01-12T20:01:23Z   order-service     order_processing_started     12344     -                      -
2024-01-12T20:01:23Z   order-service     order_validated              12344     -                      -
2024-01-12T20:01:23Z   tpl-connector     tpl_create_shipment_started  12344     -                      -
2024-01-12T20:01:55Z   tpl-connector     tpl_create_shipment_timeout  12344     TimeoutException       32004.2
2024-01-12T20:01:55Z   order-service     shipment_creation_failed     12344     TimeoutException       32105.1
2024-01-12T20:01:56Z   order-service     order_processing_started     12345     -                      -
...
```

You see immediately: **3PL API was timing out at ~32 seconds starting at 3pm EST Friday**. All 4 orders failed for the same reason.

Now drill in with the correlation ID from order 12344:

```sql
filter correlation_id = "req-a3f9b2c1"
| sort @timestamp asc
```

You see the complete journey of that single request across all services. You know:
- When it started
- Each step it took
- Where it failed
- How long each step took

**Total time to answer**: 4 minutes. **Customer impact**: You can tell them exactly what happened, why, which orders are affected, and what the remediation is.

---

## 11. Common Failure Modes

### Failure Mode 1: structlog Not Configured Before First Import

**Symptom**: Logs come out in unexpected format, or processors are missing.

**Cause**: A module-level `log = structlog.get_logger()` call caches the logger before `configure_logging()` is called.

```python
# WRONG — module-level logger initialized before configure_logging()
import structlog
log = structlog.get_logger()  # cached with default config

# Later in main.py:
configure_logging()  # too late, log above is already cached with defaults
```

**Fix**: Call `configure_logging()` as the very first thing in `main.py`, before any other imports that might create loggers.

```python
# main.py — CORRECT ORDER
from config import configure_logging
configure_logging()  # FIRST

# Now all subsequent imports get the configured processors
from services.order_service import OrderService
```

### Failure Mode 2: ContextVars Leaking Between Requests

**Symptom**: Logs for one request include `correlation_id` from a different request.

**Cause**: `structlog.contextvars.bind_contextvars()` was called but `clear_contextvars()` was never called at the start of each request.

**Fix**: Always `clear_contextvars()` at the start of the middleware, before binding new values.

```python
async def dispatch(self, request, call_next):
    structlog.contextvars.clear_contextvars()  # CLEAR FIRST
    structlog.contextvars.bind_contextvars(correlation_id=...)  # then bind
```

### Failure Mode 3: JSON Renderer Getting Non-Serializable Values

**Symptom**: `TypeError: Object of type datetime is not JSON serializable`

**Cause**: A log call includes a `datetime`, `Decimal`, `UUID`, or custom object.

**Fix**: Add a processor that coerces non-serializable types:

```python
import decimal
from datetime import datetime, date
from uuid import UUID

def coerce_to_serializable(logger, method, event_dict):
    """Convert types that aren't JSON-serializable to strings."""
    for key, value in list(event_dict.items()):
        if isinstance(value, datetime):
            event_dict[key] = value.isoformat()
        elif isinstance(value, (date,)):
            event_dict[key] = value.isoformat()
        elif isinstance(value, UUID):
            event_dict[key] = str(value)
        elif isinstance(value, decimal.Decimal):
            event_dict[key] = float(value)
        elif hasattr(value, '__dict__'):
            # Pydantic model or dataclass — use dict representation
            event_dict[key] = str(value)
    return event_dict
```

### Failure Mode 4: Logs Not Appearing in CloudWatch

**Symptom**: You write logs but they're not showing up in CloudWatch Insights.

**Causes and fixes**:
1. **Output to stderr instead of stdout**: ECS/Fargate typically captures stdout. Ensure `logger_factory=structlog.PrintLoggerFactory(sys.stdout)`.
2. **Buffer not flushed**: Add `PYTHONUNBUFFERED=1` as an environment variable to disable Python's output buffering.
3. **Wrong log group**: Verify the ECS task definition's log configuration points to the correct CloudWatch log group.
4. **Log level too high**: `LOG_LEVEL=WARNING` will suppress `INFO` logs — check the env var.

### Failure Mode 5: Correlation IDs Not Propagating

**Symptom**: Order service logs have `correlation_id`, but 3PL connector logs have a different one.

**Cause**: The outbound HTTP client isn't reading from the ContextVar.

**Debug**:
```python
# Add this temporarily to confirm what's in the context
log.debug("context_check", correlation_id_in_var=correlation_id_var.get())
```

**Fix**: Ensure the HTTP client reads the correlation ID from the ContextVar *at request time*, not at initialization time.

```python
# WRONG — reads correlation_id at class construction time (always empty)
class TplConnector:
    def __init__(self):
        self.correlation_id = correlation_id_var.get()  # wrong! always empty at init

# RIGHT — reads correlation_id when the request is made
class TplConnector:
    async def create_shipment(self, ...):
        headers = {"X-Request-ID": correlation_id_var.get()}  # correct
```

---

## 12. Interview Angle

### What Interviewers Ask

**Q: "How would you debug a customer issue where an order failed processing?"**

**Weak answer**: "I'd look at the logs and find the error."

**Strong answer**: "First, I'd ask the customer for the order ID and approximate time. If we've implemented structured logging with correlation IDs, I'd go to our log aggregation system — say CloudWatch Insights — and run a query filtered on that order ID to get the full event timeline across all services. I'd look for the first ERROR event, check the `error_type` and `error` fields, and if needed, filter by the `correlation_id` to see every service that touched this request. With this setup, I can usually give a root-cause answer within 5 minutes. If we don't have structured logging, it's a much harder process — which is why I make structured logging a day-one requirement for every service I build."

---

**Q: "What fields do you consider essential in every log entry?"**

**Strong answer**: "`timestamp` in UTC, `level`, `service` name, `correlation_id` for request tracing, and `event` as a snake_case description of what happened. For request logs: `duration_ms`, `status_code`, `method`, `path`. For error logs: `error`, `error_type`, and `exc_info` for the stack trace. The correlation ID is the most important non-obvious one — without it, you cannot connect logs across service boundaries."

---

**Q: "How do you handle sensitive data in logs?"**

**Strong answer**: "Defense in depth: first, we never log raw request bodies unless absolutely necessary. Second, we have a scrubber processor in the structlog pipeline that redacts fields matching a known sensitive list — password, token, api_key, authorization, etc. Third, for PII, we log IDs rather than values — `customer_id: 'cust-456'` instead of `email: 'john@example.com'`. Fourth, we use pre-commit hooks that scan for patterns like credit card numbers or common secret formats. The structlog processor approach is particularly powerful because it's applied uniformly to every single log entry, so you can't accidentally miss one."

---

**Q: "Describe the difference between structured logging, distributed tracing, and metrics."**

**Strong answer**: "They're complementary. Structured logs are your detailed narrative — they tell you what happened, for which entity, in what sequence. Distributed tracing shows you the call graph — which service called which, how long each hop took, where in the chain something failed. Metrics aggregate signals over time — request rate, error rate, p99 latency. In practice: you use metrics for alerting (alert when error rate > 5%), traces for debugging where in the system something is slow, and logs for the exact details of what went wrong. The three connect through correlation IDs: a trace ID is also injected into your structured logs, so you can jump from a trace to the relevant log entries."

---

## 13. Practice Exercise

### Exercise: Build a Logging-First FastAPI Service

**Goal**: Build a working order validation service that demonstrates all the concepts in this file.

**Requirements**:

1. **Structlog configured** for production (JSON) with these processors:
   - `TimeStamper(fmt="iso", utc=True)`
   - `add_log_level`
   - `add_service_context` (adds `service`, `version`, `environment`)
   - `scrub_sensitive_fields`
   - `format_exc_info`
   - `JSONRenderer`

2. **Middleware**:
   - Extracts or generates correlation ID from `X-Request-ID` header
   - Binds correlation ID to structlog context
   - Logs every request with `method`, `path`, `status_code`, `duration_ms`
   - Returns correlation ID in response `X-Request-ID` header

3. **Endpoints**:
   - `POST /orders` — accepts `{"order_id": str, "items": [...], "customer_id": str}`
   - Logs: `order_received`, `order_validated` or `order_validation_failed`, `order_created`
   - Validates: all items have positive quantity, customer_id is not empty
   - Returns correlation ID in error responses: `{"error": "...", "request_id": "..."}`

4. **Sensitive data**: The `POST /orders` request includes `"payment_token": "tok_abc123"` — confirm it's scrubbed from logs.

5. **Exception handling**: If any unexpected error occurs, log with `exc_info=True` and return a 500 with the request_id.

**Test the exercise**:

```bash
# Start the service
ENVIRONMENT=development SERVICE_NAME=order-service uvicorn main:app --port 8000

# Valid order — should see structured logs
curl -X POST http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: test-req-001" \
  -d '{"order_id": "ORD-1", "customer_id": "acme", "items": [{"sku": "A", "quantity": 2}], "payment_token": "tok_secret123"}'

# Should see in logs: payment_token is [REDACTED]
# Should see X-Request-ID: test-req-001 in response headers

# Invalid order — negative quantity
curl -X POST http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ORD-2", "customer_id": "acme", "items": [{"sku": "A", "quantity": -1}]}'

# Should see order_validation_failed in logs at WARNING level
# Should see request_id in error response body

# Simulate multiple concurrent orders — verify correlation IDs don't bleed between requests
for i in {1..10}; do
  curl -s -X POST http://localhost:8000/orders \
    -H "Content-Type: application/json" \
    -d "{\"order_id\": \"ORD-$i\", \"customer_id\": \"acme\", \"items\": [{\"sku\": \"A\", \"quantity\": 1}]}" &
done
wait

# Check logs: each request should have a unique correlation_id
# and no correlation_id should appear in a request that didn't have it
```

**Stretch goal**: Add a second service (a "fulfillment service") that the order service calls via httpx. Verify the correlation ID from the order service appears in the fulfillment service's logs by propagating the `X-Request-ID` header.

---

**Next**: [02-distributed-tracing-otel.md](./02-distributed-tracing-otel.md) — Distributed tracing builds on correlation IDs to give you full call graph visibility across services.
