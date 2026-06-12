# 04 — Error Handling: Graceful Failure for Customer-Facing Integrations

> **Related files:** [01-async-await.md](./01-async-await.md) | [02-pydantic-type-hints.md](./02-pydantic-type-hints.md) | [03-pytest-testing.md](./03-pytest-testing.md) | [../B-FastAPI/05-middleware-exception-handlers.md](../B-FastAPI/05-middleware-exception-handlers.md)

---

## The Core Principle: Error Messages Are Customer Communication

When your integration service returns an error, it will be seen by:
- A developer at the customer's company trying to integrate
- An ops person at 2am trying to figure out why orders aren't shipping
- Potentially the customer's end users

A raw Python traceback like this is a **failed deployment**:

```
Internal Server Error: 'NoneType' object has no attribute 'get' (integration.carrier_client, line 142)
```

A structured error response like this is a **professional product**:

```json
{
  "error": "carrier_api_unavailable",
  "message": "The FedEx API is temporarily unavailable. Your request has been queued and will be retried automatically. If this persists after 15 minutes, contact support with reference ID: req_8f3k2p9x.",
  "request_id": "req_8f3k2p9x",
  "retryable": true,
  "retry_after_seconds": 30,
  "timestamp": "2024-01-15T10:32:45Z"
}
```

This module teaches you how to build the second kind of error handling.

---

## Part 1: Python's Exception Hierarchy

Understanding the hierarchy helps you catch the right exceptions.

```
BaseException
├── SystemExit                    ← sys.exit() — don't catch unless you're cleaning up
├── KeyboardInterrupt             ← Ctrl+C — don't catch unless you're cleaning up
├── GeneratorExit                 ← generator.close() — don't catch
└── Exception                    ← EVERYTHING you should normally care about
    ├── StopIteration             ← for loop exhausted — usually implicit
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── AttributeError            ← None.something — extremely common
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError            ← list[99] when len < 99
    │   └── KeyError              ← dict["missing"]
    ├── OSError (IOError)
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── ConnectionError
    │       ├── ConnectionRefusedError
    │       └── ConnectionResetError
    ├── RuntimeError
    ├── TypeError                 ← wrong type passed
    ├── ValueError                ← right type, wrong value
    └── Warning (subclass hierarchy)
```

### Critical rules:
1. **Never catch `BaseException`** in application code — it swallows `KeyboardInterrupt` and `SystemExit`
2. **Never use bare `except:`** — same problem as catching `BaseException`
3. **Catch the most specific exception first** — `except ConnectionError` before `except OSError`
4. **Re-raise when you can't handle it** — don't swallow exceptions you don't understand

```python
# ❌ WRONG: catches too broadly
try:
    result = process_order(order)
except:
    print("Something went wrong")  # swallows KeyboardInterrupt, SystemExit, EVERYTHING

# ❌ WRONG: still too broad
try:
    result = process_order(order)
except Exception:
    pass  # silently discards all errors — worst thing you can do

# ✅ CORRECT: specific, actionable
try:
    result = process_order(order)
except ValidationError as e:
    raise InvalidOrderError(f"Order payload failed validation: {e}") from e
except ConnectionError as e:
    raise CarrierAPIUnavailableError("Carrier API unreachable") from e
except Exception as e:
    # Last resort — log it fully, re-raise it
    logger.error(f"Unexpected error processing order: {e}", exc_info=True)
    raise  # re-raise original exception, don't wrap in generic message
```

---

## Part 2: Custom Exception Classes

### Why Custom Exceptions?

1. **Semantic clarity**: `CarrierAPIError` tells you instantly what went wrong vs `Exception`
2. **Structured context**: carry error codes, retryability flags, customer-facing messages
3. **Catch precisely**: catch `ValidationError` without accidentally catching `DBError`
4. **Error reporting**: different exception types route to different monitoring alerts

### Building an Exception Hierarchy for an Integration Service

```python
"""
exceptions.py

Complete exception hierarchy for a logistics integration service.
Design principles:
- Every exception carries enough info to produce a useful error response
- Exceptions are categorized by retryability (important for retry logic)
- HTTP status code is part of the exception (for FastAPI exception handlers)
- Error codes are stable strings (not changing between versions)
"""

from __future__ import annotations
from typing import Any
import uuid


# ─── Base Exception ───────────────────────────────────────────────────────────

class IntegrationError(Exception):
    """
    Base for all integration service errors.
    Provides: error_code, http_status, retryable flag, request_id, context.
    """
    
    http_status: int = 500
    error_code: str = "internal_error"
    retryable: bool = False
    
    def __init__(
        self,
        message: str,
        *,
        context: dict[str, Any] | None = None,
        request_id: str | None = None,
        cause: Exception | None = None,
    ):
        super().__init__(message)
        self.message = message
        self.context = context or {}
        self.request_id = request_id or str(uuid.uuid4())[:8]
        self.cause = cause
    
    def to_response_dict(self, include_debug: bool = False) -> dict[str, Any]:
        """Convert to a structured HTTP response body."""
        response = {
            "error": self.error_code,
            "message": self.message,
            "request_id": self.request_id,
            "retryable": self.retryable,
        }
        
        if include_debug and self.context:
            response["debug"] = self.context
        
        return response
    
    def __repr__(self) -> str:
        return (
            f"{self.__class__.__name__}("
            f"message={self.message!r}, "
            f"error_code={self.error_code!r}, "
            f"request_id={self.request_id!r}, "
            f"context={self.context!r}"
            f")"
        )


# ─── Validation Errors (4xx) ──────────────────────────────────────────────────

class ValidationError(IntegrationError):
    """Request payload failed validation."""
    http_status = 422
    error_code = "validation_error"
    retryable = False  # won't succeed without fixing the payload

class InvalidPayloadError(ValidationError):
    """The payload structure is wrong (missing fields, wrong types)."""
    error_code = "invalid_payload"
    
    def __init__(self, message: str, field_errors: list[dict] | None = None, **kwargs):
        super().__init__(message, **kwargs)
        self.field_errors = field_errors or []
    
    def to_response_dict(self, include_debug: bool = False) -> dict:
        response = super().to_response_dict(include_debug)
        if self.field_errors:
            response["field_errors"] = self.field_errors
        return response

class InvalidTrackingNumberError(ValidationError):
    """Tracking number format doesn't match the carrier's expected format."""
    error_code = "invalid_tracking_number"
    
    def __init__(self, tracking_number: str, carrier: str, **kwargs):
        super().__init__(
            f"Tracking number '{tracking_number}' is not a valid format for carrier '{carrier}'",
            context={"tracking_number": tracking_number, "carrier": carrier},
            **kwargs
        )

class UnsupportedCarrierError(ValidationError):
    """Carrier code is not supported."""
    error_code = "unsupported_carrier"
    
    def __init__(self, carrier: str, supported_carriers: list[str] | None = None, **kwargs):
        supported = supported_carriers or ["FEDEX", "UPS", "USPS", "DHL"]
        super().__init__(
            f"Carrier '{carrier}' is not supported. "
            f"Supported carriers: {', '.join(supported)}",
            context={"carrier": carrier, "supported": supported},
            **kwargs
        )


# ─── Authentication Errors (4xx) ─────────────────────────────────────────────

class AuthenticationError(IntegrationError):
    """Authentication failed."""
    http_status = 401
    error_code = "authentication_failed"
    retryable = False

class AuthorizationError(IntegrationError):
    """Authenticated but not authorized for this resource."""
    http_status = 403
    error_code = "authorization_failed"
    retryable = False

class APIKeyExpiredError(AuthenticationError):
    """API key is valid format but has expired."""
    error_code = "api_key_expired"


# ─── Resource Errors (4xx) ───────────────────────────────────────────────────

class NotFoundError(IntegrationError):
    """Resource doesn't exist."""
    http_status = 404
    error_code = "not_found"
    retryable = False
    
    def __init__(self, resource_type: str, resource_id: str, **kwargs):
        super().__init__(
            f"{resource_type} with ID '{resource_id}' not found",
            context={"resource_type": resource_type, "resource_id": resource_id},
            **kwargs
        )

class ShipmentNotFoundError(NotFoundError):
    def __init__(self, shipment_id: str, **kwargs):
        super().__init__("Shipment", shipment_id, **kwargs)
        self.error_code = "shipment_not_found"

class ConflictError(IntegrationError):
    """Resource already exists / state conflict."""
    http_status = 409
    error_code = "conflict"
    retryable = False

class RateLimitError(IntegrationError):
    """Too many requests."""
    http_status = 429
    error_code = "rate_limit_exceeded"
    retryable = True
    
    def __init__(self, message: str, retry_after_seconds: int | None = None, **kwargs):
        super().__init__(message, **kwargs)
        self.retry_after_seconds = retry_after_seconds


# ─── External Service Errors (5xx, retryable) ────────────────────────────────

class ExternalServiceError(IntegrationError):
    """An upstream service we depend on failed."""
    http_status = 502
    error_code = "external_service_error"
    retryable = True
    
    def __init__(self, service_name: str, message: str, status_code: int | None = None, **kwargs):
        super().__init__(
            f"{service_name}: {message}",
            context={"service_name": service_name, "upstream_status": status_code},
            **kwargs
        )
        self.service_name = service_name
        self.upstream_status = status_code

class CarrierAPIError(ExternalServiceError):
    """Carrier API (FedEx/UPS/etc.) returned an error."""
    error_code = "carrier_api_error"
    
    def __init__(self, carrier: str, message: str, **kwargs):
        super().__init__(service_name=f"{carrier} API", message=message, **kwargs)
        self.carrier = carrier

class CarrierAPITimeoutError(CarrierAPIError):
    """Carrier API request timed out."""
    error_code = "carrier_api_timeout"
    
    def __init__(self, carrier: str, timeout_seconds: float, **kwargs):
        super().__init__(
            carrier=carrier,
            message=f"Request timed out after {timeout_seconds}s",
            **kwargs
        )

class ERPConnectionError(ExternalServiceError):
    """Cannot connect to customer's ERP system."""
    error_code = "erp_connection_error"


# ─── Internal Service Errors (5xx) ───────────────────────────────────────────

class DatabaseError(IntegrationError):
    """Database operation failed."""
    http_status = 500
    error_code = "database_error"
    retryable = True  # transient DB errors are often retryable

class ConfigurationError(IntegrationError):
    """Service is misconfigured."""
    http_status = 500
    error_code = "configuration_error"
    retryable = False  # won't fix itself without deployment


# ─── Helper Functions ─────────────────────────────────────────────────────────

def wrap_external_error(
    service_name: str,
    exc: Exception,
    error_class: type[ExternalServiceError] = ExternalServiceError
) -> ExternalServiceError:
    """
    Convert a third-party exception into our exception hierarchy.
    Preserves the original exception as the cause.
    """
    import httpx
    
    if isinstance(exc, httpx.TimeoutException):
        if hasattr(error_class, '__init__'):
            # Try to use carrier-specific error if available
            pass
        return error_class(
            service_name=service_name,
            message=f"Request timed out: {exc}",
            cause=exc
        )
    
    if isinstance(exc, httpx.HTTPStatusError):
        return error_class(
            service_name=service_name,
            message=f"HTTP {exc.response.status_code}: {exc.response.text[:200]}",
            status_code=exc.response.status_code,
            cause=exc
        )
    
    if isinstance(exc, httpx.ConnectError):
        return error_class(
            service_name=service_name,
            message=f"Could not connect: {exc}",
            cause=exc
        )
    
    return error_class(
        service_name=service_name,
        message=str(exc),
        cause=exc
    )
```

---

## Part 3: Try/Except/Else/Finally Patterns

```python
import logging
from typing import TypeVar, Callable, Any
import functools

logger = logging.getLogger(__name__)

# ─── The four clauses ─────────────────────────────────────────────────────────

async def fetch_order_with_full_handling(order_id: str) -> dict:
    """Demonstrates all four clauses and when to use each."""
    
    connection = None
    
    try:
        # Code that MIGHT raise exceptions
        connection = await acquire_db_connection()
        order = await connection.fetch_one("SELECT * FROM orders WHERE id = $1", order_id)
        
        if order is None:
            raise OrderNotFoundError(order_id)
        
        return dict(order)
    
    except OrderNotFoundError:
        # Re-raise our own exceptions without modification
        raise
    
    except asyncpg.PostgresError as e:
        # Convert third-party exception to our hierarchy
        logger.error(f"Database error fetching order {order_id}: {e}", exc_info=True)
        raise DatabaseError(
            f"Database error retrieving order '{order_id}'"
        ) from e  # chain the exception — preserves original traceback
    
    except Exception as e:
        # Catch-all: log with full traceback, re-raise
        logger.critical(
            f"Unexpected error fetching order {order_id}: {type(e).__name__}: {e}",
            exc_info=True,
            extra={"order_id": order_id}
        )
        raise  # re-raise without wrapping — don't lose information
    
    else:
        # Runs ONLY if no exception was raised
        # Good for "success" side effects
        logger.info(f"Successfully fetched order {order_id}")
        # Note: the return value already happened in try block
    
    finally:
        # Runs ALWAYS — exception or not
        # Perfect for cleanup
        if connection is not None:
            await release_db_connection(connection)

# ─── Exception Chaining ───────────────────────────────────────────────────────

# raise X from Y — explicit chaining
# The original exception Y is stored as X.__cause__
# Python shows both when printing the traceback

try:
    import json
    data = json.loads("not valid json {")
except json.JSONDecodeError as e:
    raise InvalidPayloadError(
        "Request body is not valid JSON. Check for trailing commas, unquoted strings, or encoding issues."
    ) from e
# Output:
# json.JSONDecodeError: Expecting property name ... (cause)
# The above exception was the direct cause of the following exception:
# InvalidPayloadError: Request body is not valid JSON. ...

# raise X — implicit chaining (Python records __context__ automatically)
# Less clear, but sometimes you want to suppress the context:
try:
    sensitive_operation()
except SensitiveInternalError as e:
    raise PublicError("Operation failed") from None  # suppress internal cause
```

---

## Part 4: Logging Exceptions Properly

```python
import logging
import traceback
import sys
from typing import Any

# ─── Logger setup ─────────────────────────────────────────────────────────────

def setup_logging():
    """Configure structured logging for production."""
    import json
    
    class JSONFormatter(logging.Formatter):
        """Formats log records as JSON for log aggregation systems."""
        
        def format(self, record: logging.LogRecord) -> str:
            log_dict: dict[str, Any] = {
                "timestamp": self.formatTime(record),
                "level": record.levelname,
                "logger": record.name,
                "message": record.getMessage(),
            }
            
            # Add request context if available
            from contextvars import ContextVar
            # (These would be your actual context vars from middleware)
            
            # Include exception info if present
            if record.exc_info:
                exc_type, exc_value, exc_tb = record.exc_info
                log_dict["exception"] = {
                    "type": exc_type.__name__ if exc_type else None,
                    "message": str(exc_value),
                    "traceback": traceback.format_exception(exc_type, exc_value, exc_tb),
                }
            
            # Include extra fields set by callers
            for key in ("order_id", "shipment_id", "carrier", "request_id", "customer_id"):
                if hasattr(record, key):
                    log_dict[key] = getattr(record, key)
            
            return json.dumps(log_dict)
    
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    
    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(logging.INFO)

logger = logging.getLogger(__name__)

# ─── Logging exception patterns ───────────────────────────────────────────────

# ✅ CORRECT: exc_info=True includes the full traceback
def process_shipment(shipment_id: str):
    try:
        do_something_risky()
    except Exception as e:
        logger.error(
            f"Failed to process shipment {shipment_id}",
            exc_info=True,                    # include traceback
            extra={"shipment_id": shipment_id}  # structured context
        )
        raise

# ✅ CORRECT: logger.exception() is shorthand for error + exc_info=True
def process_order(order_id: str):
    try:
        do_something_risky()
    except Exception:
        logger.exception(
            f"Unexpected error processing order {order_id}",
            extra={"order_id": order_id}
        )
        raise

# ❌ WRONG: using str(e) loses the traceback
def bad_logging(order_id: str):
    try:
        do_something_risky()
    except Exception as e:
        logger.error(f"Error: {str(e)}")  # NO TRACEBACK — impossible to debug

# ❌ WRONG: logging sensitive data
def very_bad_logging(api_request: dict):
    try:
        call_api(api_request)
    except Exception as e:
        logger.error(f"API call failed with payload: {api_request}")  # MAY LOG API KEYS

# ✅ CORRECT: log safe subset of request
def good_logging(api_request: dict):
    # Define what's safe to log
    safe_fields = {"order_id", "carrier", "service_level"}
    safe_context = {k: v for k, v in api_request.items() if k in safe_fields}
    
    try:
        call_api(api_request)
    except Exception as e:
        logger.error(
            f"API call failed",
            exc_info=True,
            extra=safe_context
        )

# ─── Never swallow exceptions silently ───────────────────────────────────────

# ❌ WRONG: exception vanishes without a trace
def terrible_pattern():
    try:
        critical_operation()
    except Exception:
        pass  # someone will spend hours debugging why nothing happened

# ✅ If you genuinely need to continue on error, at least log it
def acceptable_fallback():
    try:
        result = optional_enrichment()
        return result
    except Exception as e:
        logger.warning(
            f"Optional enrichment failed, continuing without it: {e}",
            exc_info=True
        )
        return None  # explicit fallback
```

---

## Part 5: Graceful Degradation Patterns

### Pattern 1: Partial Success

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class EnrichmentResult:
    """
    Result of enriching a shipment from multiple sources.
    Some may succeed, some may fail — we return whatever we got.
    """
    shipment_id: str
    tracking_info: dict | None = None
    order_info: dict | None = None
    warehouse_info: dict | None = None
    errors: list[str] = field(default_factory=list)
    
    @property
    def is_complete(self) -> bool:
        return all([self.tracking_info, self.order_info, self.warehouse_info])
    
    @property
    def has_minimum_viable_data(self) -> bool:
        """Minimum needed for shipping label generation."""
        return self.tracking_info is not None and self.order_info is not None

async def enrich_with_graceful_degradation(shipment_id: str) -> EnrichmentResult:
    """
    Collect data from 3 sources.
    If any fail, continue with what we have.
    The caller decides if partial data is acceptable.
    """
    result = EnrichmentResult(shipment_id=shipment_id)
    
    async with httpx.AsyncClient(timeout=5.0) as client:
        # Run all fetches concurrently, capture all exceptions
        tracking_result, order_result, warehouse_result = await asyncio.gather(
            fetch_tracking_info(client, shipment_id),
            fetch_order_info(client, shipment_id),
            fetch_warehouse_instructions(client, shipment_id),
            return_exceptions=True
        )
    
    if isinstance(tracking_result, Exception):
        logger.warning(f"Tracking fetch failed for {shipment_id}: {tracking_result}")
        result.errors.append(f"Tracking unavailable: {type(tracking_result).__name__}")
    else:
        result.tracking_info = tracking_result
    
    if isinstance(order_result, Exception):
        logger.error(f"Order fetch failed for {shipment_id}: {order_result}", exc_info=True)
        result.errors.append(f"Order info unavailable: {type(order_result).__name__}")
    else:
        result.order_info = order_result
    
    if isinstance(warehouse_result, Exception):
        logger.warning(f"Warehouse fetch failed for {shipment_id}: {warehouse_result}")
        result.errors.append("Warehouse instructions unavailable (using defaults)")
    else:
        result.warehouse_info = warehouse_result
    
    return result
```

### Pattern 2: Retry with Exponential Backoff

```python
import asyncio
import random
import logging
from functools import wraps
from typing import TypeVar, Callable, Any

F = TypeVar("F", bound=Callable[..., Any])
logger = logging.getLogger(__name__)

def retry_async(
    max_attempts: int = 3,
    backoff_base: float = 1.0,
    backoff_max: float = 30.0,
    jitter: bool = True,
    retryable_exceptions: tuple[type[Exception], ...] = (Exception,),
    retryable_status_codes: set[int] | None = None,
):
    """
    Decorator for retrying async functions with exponential backoff.
    
    Args:
        max_attempts: Total attempts (including first try)
        backoff_base: Initial wait time in seconds
        backoff_max: Maximum wait time in seconds
        jitter: Add randomness to prevent thundering herd
        retryable_exceptions: Only retry these exception types
        retryable_status_codes: For HTTP errors, only retry these status codes
    """
    retryable_status_codes = retryable_status_codes or {429, 500, 502, 503, 504}
    
    def decorator(func: F) -> F:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                
                except retryable_exceptions as e:
                    last_exception = e
                    
                    # Check if this specific exception is retryable
                    import httpx
                    if isinstance(e, httpx.HTTPStatusError):
                        if e.response.status_code not in retryable_status_codes:
                            logger.debug(
                                f"{func.__name__}: HTTP {e.response.status_code} "
                                f"is not retryable — failing immediately"
                            )
                            raise
                    
                    if attempt == max_attempts - 1:
                        # Last attempt failed
                        logger.error(
                            f"{func.__name__} failed after {max_attempts} attempts: {e}",
                            exc_info=True
                        )
                        raise
                    
                    # Calculate backoff
                    backoff = min(backoff_base * (2 ** attempt), backoff_max)
                    if jitter:
                        backoff = backoff * (0.5 + random.random() * 0.5)
                    
                    logger.warning(
                        f"{func.__name__} attempt {attempt + 1}/{max_attempts} failed: "
                        f"{type(e).__name__}: {e}. "
                        f"Retrying in {backoff:.1f}s..."
                    )
                    
                    await asyncio.sleep(backoff)
            
            raise last_exception  # should not reach here
        
        return wrapper  # type: ignore
    
    return decorator

# Usage
@retry_async(
    max_attempts=3,
    backoff_base=1.0,
    retryable_exceptions=(httpx.HTTPStatusError, httpx.TimeoutException, httpx.ConnectError),
    retryable_status_codes={429, 500, 502, 503, 504}
)
async def call_carrier_api(tracking_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://carrier.api/track/{tracking_id}")
        response.raise_for_status()
        return response.json()
```

### Pattern 3: Circuit Breaker

For when a downstream service is persistently failing — don't keep hammering it:

```python
import asyncio
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"       # normal operation — requests pass through
    OPEN = "open"           # service is broken — fail fast without calling
    HALF_OPEN = "half_open" # testing if service recovered

class CircuitBreaker:
    """
    Circuit breaker for external service calls.
    
    CLOSED → OPEN: after `failure_threshold` failures in `window_seconds`
    OPEN → HALF_OPEN: after `recovery_timeout` seconds
    HALF_OPEN → CLOSED: if probe request succeeds
    HALF_OPEN → OPEN: if probe request fails
    """
    
    def __init__(
        self,
        name: str,
        failure_threshold: int = 5,
        recovery_timeout: float = 60.0,
        window_seconds: float = 60.0,
    ):
        self.name = name
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.window_seconds = window_seconds
        
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._failure_window_start = time.time()
        self._opened_at: float | None = None
        self._lock = asyncio.Lock()
    
    @property
    def state(self) -> CircuitState:
        if self._state == CircuitState.OPEN:
            if time.time() - self._opened_at > self.recovery_timeout:
                return CircuitState.HALF_OPEN
        return self._state
    
    async def call(self, func, *args, **kwargs):
        """Execute func through the circuit breaker."""
        async with self._lock:
            state = self.state
        
        if state == CircuitState.OPEN:
            raise ExternalServiceError(
                service_name=self.name,
                message=(
                    f"Circuit breaker is OPEN — {self.name} is experiencing issues. "
                    f"Failing fast to protect the system. "
                    f"Will attempt recovery in {self.recovery_timeout:.0f}s."
                )
            )
        
        try:
            result = await func(*args, **kwargs)
            await self._on_success(state)
            return result
        
        except Exception as e:
            await self._on_failure(state, e)
            raise
    
    async def _on_success(self, state: CircuitState) -> None:
        async with self._lock:
            if state == CircuitState.HALF_OPEN:
                logger.info(f"Circuit breaker {self.name}: HALF_OPEN → CLOSED (probe succeeded)")
                self._state = CircuitState.CLOSED
                self._failure_count = 0
    
    async def _on_failure(self, state: CircuitState, exc: Exception) -> None:
        async with self._lock:
            now = time.time()
            
            # Reset window if expired
            if now - self._failure_window_start > self.window_seconds:
                self._failure_count = 0
                self._failure_window_start = now
            
            self._failure_count += 1
            
            if state == CircuitState.HALF_OPEN or self._failure_count >= self.failure_threshold:
                logger.error(
                    f"Circuit breaker {self.name}: opening after "
                    f"{self._failure_count} failures. Error: {exc}"
                )
                self._state = CircuitState.OPEN
                self._opened_at = now

# ─── Global circuit breakers ─────────────────────────────────────────────────

_circuit_breakers: dict[str, CircuitBreaker] = {}

def get_circuit_breaker(service_name: str) -> CircuitBreaker:
    if service_name not in _circuit_breakers:
        _circuit_breakers[service_name] = CircuitBreaker(
            name=service_name,
            failure_threshold=5,
            recovery_timeout=60.0
        )
    return _circuit_breakers[service_name]

# Usage
async def call_erp_with_circuit_breaker(order_id: str) -> dict:
    cb = get_circuit_breaker("customer-erp")
    return await cb.call(fetch_order_from_erp, order_id)
```

---

## Part 6: Structured Error Responses for HTTP APIs

This is what your FastAPI exception handlers should produce. Covered in detail in [../B-FastAPI/05-middleware-exception-handlers.md](../B-FastAPI/05-middleware-exception-handlers.md), but the model is defined here.

```python
from pydantic import BaseModel, Field
from typing import Any
from datetime import datetime, timezone

class ErrorDetail(BaseModel):
    """Individual field validation error."""
    field: str
    message: str
    received: str | None = None

class ErrorResponse(BaseModel):
    """
    Standardized error response for all API errors.
    Design principles:
    - error: stable machine-readable code (don't change between versions)
    - message: human-readable, actionable, no internal details
    - request_id: for support tickets ("I got error, ref: req_8f3k2p9x")
    - retryable: client can decide whether to retry automatically
    - timestamp: for correlation with logs
    """
    error: str = Field(description="Stable error code, e.g. 'validation_error'")
    message: str = Field(description="Human-readable explanation and suggested action")
    request_id: str = Field(description="Unique request identifier for support tracing")
    retryable: bool = Field(description="Whether retrying the same request may succeed")
    timestamp: str = Field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    field_errors: list[ErrorDetail] | None = Field(None, description="Per-field validation errors")
    retry_after_seconds: int | None = Field(None, description="For 429: when to retry")

# Examples of what good error responses look like:

good_validation_error = ErrorResponse(
    error="invalid_payload",
    message=(
        "Your request payload contains 3 validation errors. "
        "Please check the 'field_errors' array for details on each invalid field."
    ),
    request_id="req_8f3k2p9x",
    retryable=False,
    field_errors=[
        ErrorDetail(
            field="shipTo.postalCode",
            message="Value must be a valid US ZIP code (5 digits or ZIP+4 format)",
            received="ABCDE"
        ),
        ErrorDetail(
            field="weight_lbs",
            message="Value must be greater than 0",
            received="-5.0"
        ),
        ErrorDetail(
            field="carrier",
            message="Value must be one of: FEDEX, UPS, USPS, DHL, ONTRAC",
            received="TELEPORTATION"
        )
    ]
)

good_service_error = ErrorResponse(
    error="carrier_api_unavailable",
    message=(
        "The FedEx API is temporarily unavailable. "
        "Your request has been queued and will be retried automatically within 30 seconds. "
        "If this persists, contact support with reference ID: req_9z2m4k8p."
    ),
    request_id="req_9z2m4k8p",
    retryable=True,
    retry_after_seconds=30
)

good_not_found_error = ErrorResponse(
    error="shipment_not_found",
    message=(
        "No shipment found with ID 'SHIP-99999'. "
        "Please verify the shipment ID and try again. "
        "Note: shipments older than 90 days may have been archived."
    ),
    request_id="req_7x1n5p3q",
    retryable=False
)
```

---

## Part 7: Global Exception Handler in FastAPI

```python
"""
exception_handlers.py

FastAPI exception handlers that convert our exception hierarchy
to consistent HTTP responses.
"""

import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError as PydanticValidationError

from .exceptions import (
    IntegrationError,
    ValidationError as OurValidationError,
    AuthenticationError,
    AuthorizationError,
    NotFoundError,
    RateLimitError,
    ExternalServiceError,
    DatabaseError,
)
from .models import ErrorResponse, ErrorDetail

logger = logging.getLogger(__name__)

def register_exception_handlers(app: FastAPI) -> None:
    """Register all exception handlers. Call this in create_app()."""
    
    app.add_exception_handler(IntegrationError, handle_integration_error)
    app.add_exception_handler(RequestValidationError, handle_fastapi_validation_error)
    app.add_exception_handler(PydanticValidationError, handle_pydantic_validation_error)
    app.add_exception_handler(Exception, handle_unexpected_error)

async def handle_integration_error(
    request: Request,
    exc: IntegrationError
) -> JSONResponse:
    """Handle our custom exceptions."""
    
    # Log level based on severity
    if exc.http_status >= 500:
        logger.error(
            f"Integration error: {exc.error_code}: {exc.message}",
            exc_info=exc.cause,
            extra={
                "error_code": exc.error_code,
                "request_id": exc.request_id,
                "path": str(request.url),
                **exc.context
            }
        )
    elif exc.http_status >= 400:
        logger.warning(
            f"Client error: {exc.error_code}: {exc.message}",
            extra={
                "error_code": exc.error_code,
                "request_id": exc.request_id,
                "path": str(request.url),
            }
        )
    
    # Build response
    response_data = exc.to_response_dict(
        include_debug=request.headers.get("X-Debug-Mode") == "true"
    )
    
    headers = {}
    if isinstance(exc, RateLimitError) and exc.retry_after_seconds:
        headers["Retry-After"] = str(exc.retry_after_seconds)
    
    return JSONResponse(
        status_code=exc.http_status,
        content=response_data,
        headers=headers
    )

async def handle_fastapi_validation_error(
    request: Request,
    exc: RequestValidationError
) -> JSONResponse:
    """
    Handle FastAPI/Pydantic validation errors from request models.
    Convert from FastAPI's format to our error format.
    """
    import uuid
    request_id = str(uuid.uuid4())[:8]
    
    # Convert Pydantic errors to our format
    field_errors = []
    for error in exc.errors():
        field_path = " → ".join(str(loc) for loc in error["loc"] if loc != "body")
        field_errors.append(ErrorDetail(
            field=field_path,
            message=error["msg"],
            received=str(error.get("input", ""))[:100]  # truncate long values
        ))
    
    logger.info(
        f"Request validation error: {len(field_errors)} field(s) invalid",
        extra={"request_id": request_id, "path": str(request.url)}
    )
    
    return JSONResponse(
        status_code=422,
        content={
            "error": "validation_error",
            "message": (
                f"Request validation failed with {len(field_errors)} error(s). "
                f"See 'field_errors' for details on each invalid field."
            ),
            "request_id": request_id,
            "retryable": False,
            "field_errors": [e.model_dump() for e in field_errors]
        }
    )

async def handle_pydantic_validation_error(
    request: Request,
    exc: PydanticValidationError
) -> JSONResponse:
    """Handle Pydantic validation errors that escape endpoint handling."""
    return await handle_fastapi_validation_error(
        request,
        RequestValidationError(errors=exc.errors())
    )

async def handle_unexpected_error(
    request: Request,
    exc: Exception
) -> JSONResponse:
    """
    Catch-all for any unhandled exception.
    
    CRITICAL: never expose internal details to the caller.
    Log everything internally, return minimal info externally.
    """
    import uuid
    request_id = str(uuid.uuid4())[:8]
    
    logger.critical(
        f"Unhandled exception: {type(exc).__name__}: {exc}",
        exc_info=True,
        extra={
            "request_id": request_id,
            "path": str(request.url),
            "method": request.method,
            "exception_type": type(exc).__name__,
        }
    )
    
    # Return MINIMAL info — no internal details
    return JSONResponse(
        status_code=500,
        content={
            "error": "internal_error",
            "message": (
                "An unexpected error occurred. Our team has been notified. "
                f"Please contact support with reference ID: {request_id} "
                "if this persists."
            ),
            "request_id": request_id,
            "retryable": False,
        }
    )

# ─── Wiring it all together ───────────────────────────────────────────────────

def create_app() -> FastAPI:
    app = FastAPI(
        title="Logistics Integration API",
        version="1.0.0",
    )
    
    register_exception_handlers(app)
    
    # Include routers...
    
    return app
```

---

## Part 8: Wrapping Third-Party API Errors

Real-world pattern: you're calling a carrier SDK or raw HTTP API. Their errors are opaque. Here's how to convert them:

```python
"""
carrier_client.py

Wraps the carrier API, converting all third-party errors
into our exception hierarchy.
"""

import httpx
import logging
from typing import Any

from .exceptions import (
    CarrierAPIError,
    CarrierAPITimeoutError,
    InvalidTrackingNumberError,
    ExternalServiceError,
    RateLimitError,
    NotFoundError,
)

logger = logging.getLogger(__name__)

class CarrierAPIClient:
    """
    HTTP client for carrier APIs.
    All errors are converted to our exception hierarchy.
    Callers never deal with httpx exceptions.
    """
    
    def __init__(self, carrier: str, base_url: str, api_key: str, timeout: float = 10.0):
        self.carrier = carrier
        self.base_url = base_url
        self.api_key = api_key
        self.timeout = timeout
        self._client: httpx.AsyncClient | None = None
    
    async def __aenter__(self):
        self._client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"X-API-Key": self.api_key},
            timeout=httpx.Timeout(
                connect=2.0,
                read=self.timeout,
                write=2.0,
                pool=1.0,
            )
        )
        return self
    
    async def __aexit__(self, *args):
        if self._client:
            await self._client.aclose()
    
    async def _make_request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> dict[str, Any]:
        """
        Central request method — all HTTP calls go through here.
        Converts ALL httpx/HTTP errors to our exceptions.
        """
        assert self._client is not None, "Must use as context manager"
        
        url = endpoint  # base_url is set on client
        
        try:
            response = await self._client.request(method, url, **kwargs)
        
        except httpx.TimeoutException as e:
            raise CarrierAPITimeoutError(
                carrier=self.carrier,
                timeout_seconds=self.timeout
            ) from e
        
        except httpx.ConnectError as e:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"Cannot connect to {self.carrier} API. Check network connectivity."
            ) from e
        
        except httpx.RequestError as e:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"Request error: {e}"
            ) from e
        
        # Handle HTTP error responses
        try:
            response.raise_for_status()
        except httpx.HTTPStatusError as e:
            await self._handle_http_error(response, e)
        
        # Parse response
        try:
            return response.json()
        except Exception as e:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"Could not parse {self.carrier} API response as JSON: {e}",
                status_code=response.status_code
            ) from e
    
    async def _handle_http_error(
        self,
        response: httpx.Response,
        exc: httpx.HTTPStatusError
    ) -> None:
        """Convert HTTP error responses to specific exceptions."""
        
        status = response.status_code
        
        # Try to get error details from response body
        try:
            error_body = response.json()
            api_error_message = error_body.get("message") or error_body.get("error") or ""
        except Exception:
            api_error_message = response.text[:200]
        
        if status == 400:
            # Carrier rejected the request — usually a bad tracking number
            raise InvalidTrackingNumberError(
                tracking_number=self._extract_tracking_from_url(response.url),
                carrier=self.carrier,
            ) from exc
        
        elif status == 401:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"{self.carrier} API authentication failed. Check your API credentials.",
                status_code=status
            ) from exc
        
        elif status == 404:
            # Tracking number not found
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"Tracking number not found in {self.carrier} system. It may be too new.",
                status_code=status
            ) from exc
        
        elif status == 429:
            retry_after = int(response.headers.get("Retry-After", 30))
            raise RateLimitError(
                f"{self.carrier} API rate limit exceeded.",
                retry_after_seconds=retry_after
            ) from exc
        
        elif status in {500, 502, 503, 504}:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"{self.carrier} API is experiencing issues (HTTP {status}). Try again in a few minutes.",
                status_code=status
            ) from exc
        
        else:
            raise CarrierAPIError(
                carrier=self.carrier,
                message=f"{self.carrier} API returned unexpected status {status}: {api_error_message}",
                status_code=status
            ) from exc
    
    def _extract_tracking_from_url(self, url) -> str:
        """Try to get tracking number from URL path."""
        parts = str(url).split("/")
        return parts[-1] if parts else "unknown"
    
    async def get_tracking(self, tracking_number: str) -> dict:
        """Get tracking status for a shipment."""
        return await self._make_request("GET", f"/track/{tracking_number}")
    
    async def create_shipment(self, shipment_data: dict) -> dict:
        """Create a new shipment and get a label."""
        return await self._make_request("POST", "/shipments", json=shipment_data)
```

---

## FDE Context Callouts

> **FDE Context: Error Messages as Customer Documentation**
>
> Think of every error message as documentation. When a developer at the customer's company sees your error, they should know: (1) what went wrong, (2) whose fault it is (their data, your service, or a downstream service), and (3) what to do next. "Internal Server Error" fails all three. "Field 'carrier': value 'TELEPORTATION' is not a valid carrier code. Valid values: FEDEX, UPS, USPS, DHL" passes all three.

> **FDE Context: The Request ID Is Your Lifeline**
>
> At a customer deployment, you will get this message: "Something failed around 3pm yesterday, order ORD-78234." Without a request ID tied to every error response AND logged on your end, this is a needle-in-a-haystack investigation. With a request ID, it's a one-second log query. Make the request ID part of every error response, every log line, and train customers to include it in support tickets.

> **FDE Context: Don't Retry Non-Retryable Errors**
>
> If you get a 400 Bad Request from a carrier API, retrying the same payload will get you the same 400. If you get a 503 Service Unavailable, retrying makes sense. The `retryable` flag on your exception model is not just documentation — it controls whether your retry middleware actually retries. Getting this wrong burns your customer's API rate limits.

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| Customers see stack traces in API responses | No global exception handler, or handler re-raises | Add `handle_unexpected_error` that never exposes internals |
| 5xx errors are impossible to trace in logs | Exceptions logged without `exc_info=True` | Always use `logger.exception()` or `logger.error(..., exc_info=True)` |
| Retry logic retries 400 responses | `retryable_status_codes` includes 400 | 400 = bad request = won't improve, don't retry |
| Circuit breaker never closes | HALF_OPEN probe fails on first try, reopens | Add multiple probe successes required before closing |
| Custom exceptions swallowed at boundary | Bare `except Exception: pass` somewhere | Grep for `except.*pass` and `except.*:` followed by only logging |
| Error context missing from logs | Context in exception not extracted in handler | In exception handler, call `exc.context` and add to `extra={}` |

---

## Interview Angle

**Q: "How do you ensure that internal implementation details don't leak to API consumers?"**

Great answer: "Three-layer defense. First, all exceptions inherit from a base class that has a `to_response_dict()` method — it controls exactly what goes into the response and never includes stack traces, module paths, or raw database errors. Second, there's a catch-all exception handler for unhandled exceptions that returns a generic '500 internal error with request_id' and logs the full traceback internally. Third, I run the error handler in tests with assertions that check the response body doesn't contain file paths, line numbers, or internal function names."

**Q: "What's exception chaining and when do you use it?"**

Great answer: "`raise NewException() from original_exception` creates a chain where the original is stored as `__cause__`. Python shows both in tracebacks. I use it whenever I'm converting a third-party exception to my own — `raise CarrierAPIError('FedEx timeout') from httpx.TimeoutException`. This preserves the full context for debugging while giving callers a clean, typed exception to catch. The key is `from e` not just `from None` — you always want the cause in your logs."

---

## Practice Exercise

**Scenario:** You're integrating with three systems. Design the complete error handling for this service method:

```python
async def book_shipment(order: InboundOrder) -> ShipmentConfirmation:
    """
    1. Validate the order
    2. Get rates from carrier API
    3. Reserve inventory in WMS
    4. Create shipment in carrier system
    5. Update order status in ERP
    6. Return confirmation
    """
```

**Task 1:** Define a custom exception hierarchy with at least 8 exception types covering: validation failure, carrier rate fetch failure, inventory shortage, carrier booking failure, ERP update failure. Each must have: error_code, http_status, retryable flag, structured context.

**Task 2:** Implement `book_shipment()` with proper exception handling at each step — some failures should stop the process, some should retry, some should proceed with fallback behavior.

**Task 3:** Write a `to_response_dict()` method on your base exception that produces a customer-friendly JSON response. Write 3 tests that verify: (a) internal file paths never appear in the response, (b) the request_id is always present, (c) the error_code is stable (test a specific value, not just "exists").

**Task 4:** Implement a `retry_async` decorator that works for the carrier API call — 3 attempts, exponential backoff starting at 1s, only retries on 429/500/502/503/504.

---

*Previous: [03-pytest-testing.md](./03-pytest-testing.md) | Next: [05-packaging-uv-poetry.md](./05-packaging-uv-poetry.md)*
