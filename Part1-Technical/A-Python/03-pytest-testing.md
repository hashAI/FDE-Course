# 03 — pytest: Testing Integration Code for FDE Work

> **Related files:** [01-async-await.md](./01-async-await.md) | [02-pydantic-type-hints.md](./02-pydantic-type-hints.md) | [04-error-handling.md](./04-error-handling.md) | [../B-FastAPI/01-async-endpoints-di.md](../B-FastAPI/01-async-endpoints-di.md)

---

## Why Testing Is Different for FDE Work

A typical software project tests its own logic. FDE work tests the **contract between your code and external systems** — carrier APIs, customer ERPs, warehouse management systems, message queues.

Your tests need to answer:
- Does your integration handle the customer's malformed payload gracefully?
- Does your retry logic actually retry, and stop after N attempts?
- When the carrier API returns a 429, does your service back off correctly?
- When the database is unavailable, does the error message tell the customer something useful?

These scenarios happen at 2am on a go-live weekend. The test suite you write during development is what prevents the 2am call.

---

## Part 1: pytest Fundamentals

### Test Discovery

pytest finds tests by convention:
- Files: `test_*.py` or `*_test.py`
- Functions: `test_*`
- Classes: `Test*` (no `__init__`)
- Methods in test classes: `test_*`

```
project/
├── src/
│   └── integration/
│       ├── __init__.py
│       ├── carrier_client.py
│       └── order_processor.py
└── tests/
    ├── conftest.py          ← shared fixtures (pytest finds this automatically)
    ├── test_carrier_client.py
    ├── test_order_processor.py
    └── integration/
        ├── conftest.py      ← integration-specific fixtures
        └── test_end_to_end.py
```

Run tests:
```bash
pytest                          # all tests
pytest tests/test_carrier_client.py  # specific file
pytest tests/ -v                # verbose (show test names)
pytest tests/ -k "carrier"      # filter by keyword
pytest tests/ -m "not slow"     # run tests not marked 'slow'
pytest tests/ --tb=short        # shorter traceback format
pytest tests/ -x               # stop on first failure
pytest tests/ -s               # print stdout (don't capture)
pytest tests/ --co              # collect only (show test names without running)
```

### Assertions

pytest's assert introspection gives you much better failure messages than `unittest`:

```python
# pytest rewrites assert statements to give rich failure output

def test_carrier_normalization():
    from integration.carrier_client import normalize_carrier_name
    
    # Simple equality
    assert normalize_carrier_name("Federal Express") == "FEDEX"
    
    # Container checks
    result = get_supported_carriers()
    assert "FEDEX" in result
    assert len(result) >= 4
    
    # With custom message (shown on failure)
    actual = normalize_carrier_name("unknown carrier")
    assert actual == "UNKNOWN", f"Expected 'UNKNOWN' but got '{actual}'"
    
    # Approximate float equality
    import math
    assert math.isclose(calculate_dimensional_weight(12, 10, 8), 6.9, rel_tol=0.01)
    
    # Exception raising — the idiomatic pytest way
    import pytest
    with pytest.raises(ValueError, match="Cannot parse weight"):
        parse_weight("not a number at all")
    
    # Check exception attributes
    with pytest.raises(ValueError) as exc_info:
        parse_weight("xyz")
    assert "Cannot parse weight" in str(exc_info.value)
    assert exc_info.value.field_name == "weight"  # if it's a custom exception
```

### Markers

```python
import pytest

@pytest.mark.slow
def test_bulk_shipment_processing():
    """Takes 30 seconds — skip in fast feedback loops."""
    ...

@pytest.mark.integration
def test_carrier_api_real_call():
    """Requires real carrier API credentials — only in integration environment."""
    ...

@pytest.mark.skip(reason="Known flaky — being investigated")
def test_webhook_retry():
    ...

@pytest.mark.skipif(
    condition=not os.getenv("CARRIER_API_KEY"),
    reason="CARRIER_API_KEY not set"
)
def test_live_carrier_call():
    ...

@pytest.mark.xfail(reason="Bug #456 — carrier returns wrong format")
def test_carrier_date_format():
    """We know this fails — tracking it until the bug is fixed."""
    ...
```

Register custom markers in `pyproject.toml` to avoid warnings:
```toml
[tool.pytest.ini_options]
markers = [
    "slow: tests that take >5 seconds",
    "integration: tests that hit real external services",
    "unit: fast, isolated unit tests",
]
```

---

## Part 2: Fixtures — The Core of Maintainable Tests

Fixtures are the mechanism for **setting up test state** and **cleaning it up**. They replace setUp/tearDown and are more composable.

### Basic Fixtures

```python
# conftest.py

import pytest
from datetime import date

@pytest.fixture
def sample_shipment() -> dict:
    """A valid shipment payload for testing."""
    return {
        "shipment_id": "SHIP-TEST-001",
        "tracking_number": "1Z999AA1012345678",
        "carrier": "UPS",
        "status": "IN_TRANSIT",
        "ship_date": "2024-01-15",
        "weight_lbs": 12.5,
        "origin": {
            "street1": "100 Main St",
            "city": "Chicago",
            "state": "IL",
            "postal_code": "60601"
        },
        "destination": {
            "street1": "200 Oak Ave",
            "city": "Dallas",
            "state": "TX",
            "postal_code": "75201"
        }
    }

@pytest.fixture
def malformed_shipment() -> dict:
    """A payload that should fail validation."""
    return {
        "shipment_id": "",          # empty string
        "tracking_number": "BAD",   # too short
        "carrier": "TELEPORTATION", # invalid carrier
        "weight_lbs": -5,           # negative weight
        "ship_date": "not-a-date",  # invalid date
    }

# Using fixtures in tests
def test_valid_shipment_parses(sample_shipment):
    from integration.models import Shipment
    model = Shipment.model_validate(sample_shipment)
    assert model.shipment_id == "SHIP-TEST-001"
    assert model.carrier == "UPS"

def test_malformed_shipment_raises(malformed_shipment):
    from pydantic import ValidationError
    from integration.models import Shipment
    with pytest.raises(ValidationError):
        Shipment.model_validate(malformed_shipment)
```

### Fixture Scopes

```python
import pytest
import asyncpg  # async postgres driver

# ─── Function scope (default) ────────────────────────────────────────────────
# Fresh fixture for every test. Safest.

@pytest.fixture
def empty_order_list():
    return []  # new list for each test

# ─── Module scope ────────────────────────────────────────────────────────────
# One fixture instance shared by all tests in the module.
# Good for expensive setup that you don't want to repeat.

@pytest.fixture(scope="module")
def carrier_rate_table():
    """Load rate tables from file — expensive, share across module."""
    import json
    from pathlib import Path
    with open("tests/fixtures/carrier_rates.json") as f:
        return json.load(f)

# ─── Session scope ───────────────────────────────────────────────────────────
# One fixture instance for the entire test session.
# Good for: database connection pool, HTTP client pool.

@pytest.fixture(scope="session")
def db_pool():
    """
    WARNING: session-scoped fixtures with yield need special handling in async.
    """
    import asyncio
    pool = asyncio.get_event_loop().run_until_complete(
        asyncpg.create_pool(dsn="postgresql://test:test@localhost/testdb")
    )
    yield pool
    asyncio.get_event_loop().run_until_complete(pool.close())

# ─── Class scope ─────────────────────────────────────────────────────────────
# Shared within a test class.

@pytest.fixture(scope="class")
def registered_user(self):
    return {"user_id": "TEST-USER-001", "api_key": "test-key"}
```

### Yield Fixtures — Setup and Teardown

```python
import pytest
import tempfile
import os

@pytest.fixture
def temp_csv_file():
    """
    Creates a temporary CSV file, yields the path, then cleans up.
    The code after yield is the teardown.
    """
    content = """shipment_id,carrier,weight
SHIP-001,FEDEX,12.5
SHIP-002,UPS,8.2
SHIP-003,USPS,1.1
"""
    with tempfile.NamedTemporaryFile(
        mode="w",
        suffix=".csv",
        delete=False,
        encoding="utf-8"
    ) as f:
        f.write(content)
        temp_path = f.name
    
    yield temp_path  # test runs here
    
    # Cleanup — runs even if the test fails
    os.unlink(temp_path)

@pytest.fixture
def mock_redis():
    """
    In-memory redis replacement for tests.
    Using fakeredis to avoid requiring a real Redis server.
    """
    import fakeredis
    server = fakeredis.FakeServer()
    client = fakeredis.FakeRedis(server=server)
    
    yield client  # tests use this
    
    client.flushall()
    client.close()

# Using both in one test
def test_shipment_csv_processing(temp_csv_file, mock_redis):
    from integration.csv_processor import process_shipment_csv
    
    processed = process_shipment_csv(temp_csv_file)
    assert len(processed) == 3
    assert processed[0].shipment_id == "SHIP-001"
```

### autouse Fixtures

```python
import pytest

@pytest.fixture(autouse=True)
def reset_singleton_state():
    """
    Automatically runs for every test.
    Useful for resetting global state (metrics, caches, etc.)
    """
    from integration.metrics import reset_counters
    reset_counters()
    yield
    # Optionally assert state was clean at end too

@pytest.fixture(autouse=True, scope="session")
def configure_test_logging():
    """Set up logging configuration once for all tests."""
    import logging
    logging.basicConfig(level=logging.WARNING)  # suppress info logs in tests
```

### Fixture Factories

When you need multiple variations of a fixture:

```python
import pytest

@pytest.fixture
def make_shipment():
    """
    Returns a factory function for creating shipments.
    Tests can call it multiple times with different params.
    """
    def _factory(
        carrier: str = "FEDEX",
        status: str = "IN_TRANSIT",
        weight_lbs: float = 12.5,
        **overrides
    ) -> dict:
        base = {
            "shipment_id": f"SHIP-{carrier}-001",
            "tracking_number": "1Z999AA1012345678",
            "carrier": carrier,
            "status": status,
            "ship_date": "2024-01-15",
            "weight_lbs": weight_lbs,
        }
        base.update(overrides)
        return base
    
    return _factory

def test_multiple_carriers(make_shipment):
    fedex = make_shipment(carrier="FEDEX", weight_lbs=15.0)
    ups = make_shipment(carrier="UPS", status="DELIVERED")
    
    assert fedex["carrier"] == "FEDEX"
    assert ups["status"] == "DELIVERED"

def test_overweight_shipment(make_shipment):
    heavy = make_shipment(weight_lbs=150.0, carrier="FREIGHT")
    # Test that freight routing kicks in above 100lbs
    ...
```

---

## Part 3: Parametrize — Testing Multiple Inputs

```python
import pytest

# ─── Basic parametrize ───────────────────────────────────────────────────────

@pytest.mark.parametrize("input_value,expected", [
    ("Federal Express", "FEDEX"),
    ("fedex", "FEDEX"),
    ("FDX", "FEDEX"),
    ("United Parcel Service", "UPS"),
    ("ups", "UPS"),
    ("U.P.S.", "UPS"),
    ("USPS", "USPS"),
    ("United States Postal Service", "USPS"),
    ("DHL Express", "DHL"),
    ("dhl", "DHL"),
])
def test_carrier_normalization(input_value, expected):
    from integration.utils import normalize_carrier_name
    assert normalize_carrier_name(input_value) == expected

# ─── Parametrize with expected exceptions ────────────────────────────────────

@pytest.mark.parametrize("weight_str,expected", [
    ("12.5 lbs", 12.5),
    ("12.5", 12.5),
    ("5.6 kg", pytest.approx(12.35, rel=0.01)),  # 5.6 * 2.205
    ("100g", pytest.approx(0.22, rel=0.01)),      # 100g * 0.0022046
    ("1.0 oz", pytest.approx(0.0625, rel=0.01)), # 1 oz = 0.0625 lbs
])
def test_weight_parsing(weight_str, expected):
    from integration.utils import parse_weight_lbs
    result = parse_weight_lbs(weight_str)
    assert result == expected

@pytest.mark.parametrize("weight_str", [
    "abc",
    "",
    "???",
    None,
])
def test_weight_parsing_invalid(weight_str):
    from integration.utils import parse_weight_lbs
    with pytest.raises((ValueError, TypeError)):
        parse_weight_lbs(weight_str)

# ─── Parametrize with IDs for readable output ─────────────────────────────────

@pytest.mark.parametrize("date_str,expected", [
    ("2024-01-15", date(2024, 1, 15)),
    ("01/15/2024", date(2024, 1, 15)),
    ("15/01/2024", date(2024, 1, 15)),
    ("Jan 15, 2024", date(2024, 1, 15)),
    ("20240115", date(2024, 1, 15)),
], ids=[
    "iso-format",
    "us-format",
    "eu-format",
    "written-format",
    "compact-format",
])
def test_date_parsing(date_str, expected):
    from datetime import date
    from integration.utils import parse_flexible_date
    assert parse_flexible_date(date_str) == expected

# ─── Nested parametrize (all combinations) ───────────────────────────────────

@pytest.mark.parametrize("carrier", ["FEDEX", "UPS", "USPS", "DHL"])
@pytest.mark.parametrize("service_level", ["GROUND", "2DAY", "OVERNIGHT"])
def test_rate_calculation_all_combinations(carrier, service_level):
    """Generates 4 × 3 = 12 test cases automatically."""
    from integration.rates import calculate_rate
    rate = calculate_rate(carrier=carrier, service_level=service_level, weight_lbs=10.0)
    assert rate > 0.0
    assert rate < 1000.0
```

---

## Part 4: Mocking — Isolating External Systems

### unittest.mock Basics

```python
from unittest.mock import patch, MagicMock, call
import pytest

# ─── Patch a module-level attribute ──────────────────────────────────────────

def test_carrier_api_success():
    """Test without hitting the real carrier API."""
    
    # IMPORTANT: patch where the name is USED, not where it's defined
    with patch("integration.carrier_client.httpx.get") as mock_get:
        mock_get.return_value = MagicMock(
            status_code=200,
            json=lambda: {
                "tracking_id": "1Z999AA1012345678",
                "status": "IN_TRANSIT",
                "estimated_delivery": "2024-01-18"
            }
        )
        
        from integration.carrier_client import get_tracking_status
        result = get_tracking_status("1Z999AA1012345678")
        
        assert result["status"] == "IN_TRANSIT"
        mock_get.assert_called_once_with(
            "https://carrier-api.internal/track/1Z999AA1012345678",
            headers=ANY,
            timeout=10
        )

# ─── Patch as decorator ───────────────────────────────────────────────────────

@patch("integration.email_service.send_email")
@patch("integration.carrier_client.get_tracking_status")
def test_shipment_notification(mock_tracking, mock_email):
    """Multiple patches — order is bottom-up in function params."""
    
    mock_tracking.return_value = {
        "status": "DELIVERED",
        "delivered_at": "2024-01-18T14:22:00"
    }
    
    from integration.notifications import send_delivery_notification
    send_delivery_notification("1Z999AA1012345678", "customer@example.com")
    
    # Verify email was sent with correct content
    mock_email.assert_called_once()
    call_args = mock_email.call_args
    assert "DELIVERED" in call_args.kwargs["body"]
    assert call_args.kwargs["to"] == "customer@example.com"

# ─── Side effects — different return values per call ──────────────────────────

def test_retry_logic():
    """Test that retry logic works: first two calls fail, third succeeds."""
    
    from requests.exceptions import ConnectionError, Timeout
    
    with patch("integration.carrier_client.requests.get") as mock_get:
        # First call: connection error
        # Second call: timeout
        # Third call: success
        mock_get.side_effect = [
            ConnectionError("Connection refused"),
            Timeout("Request timed out"),
            MagicMock(status_code=200, json=lambda: {"status": "OK"})
        ]
        
        from integration.carrier_client import get_tracking_with_retry
        result = get_tracking_with_retry("1Z999AA1012345678", max_retries=3)
        
        assert result["status"] == "OK"
        assert mock_get.call_count == 3  # was called 3 times

# ─── MagicMock inspection ─────────────────────────────────────────────────────

def test_erp_update_called_correctly():
    with patch("integration.erp_client.ERPClient") as MockERPClient:
        mock_erp = MagicMock()
        MockERPClient.return_value = mock_erp
        
        from integration.order_processor import update_order_status
        update_order_status("ORD-001", "SHIPPED", tracking="1Z999AA1012345678")
        
        # Verify the method was called
        mock_erp.update_order.assert_called_once()
        
        # Inspect the call arguments
        args, kwargs = mock_erp.update_order.call_args
        assert kwargs["order_id"] == "ORD-001"
        assert kwargs["status"] == "SHIPPED"
        assert kwargs["tracking_number"] == "1Z999AA1012345678"
        
        # Check call count
        assert mock_erp.update_order.call_count == 1
        
        # Verify calls in order
        mock_erp.update_order.assert_has_calls([
            call(order_id="ORD-001", status="SHIPPED", tracking_number="1Z999AA1012345678")
        ])
```

### AsyncMock for Async Functions

```python
from unittest.mock import AsyncMock, patch, MagicMock
import pytest
import pytest_asyncio

@pytest.mark.asyncio
async def test_async_carrier_call():
    """Test an async function that calls an async carrier API."""
    
    with patch("integration.carrier_client.httpx.AsyncClient") as MockClient:
        # Set up the mock context manager
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "tracking_id": "1Z999AA1012345678",
            "status": "IN_TRANSIT"
        }
        mock_response.raise_for_status = MagicMock()  # doesn't raise
        
        mock_client_instance = AsyncMock()
        mock_client_instance.get = AsyncMock(return_value=mock_response)
        mock_client_instance.__aenter__ = AsyncMock(return_value=mock_client_instance)
        mock_client_instance.__aexit__ = AsyncMock(return_value=None)
        MockClient.return_value = mock_client_instance
        
        from integration.carrier_client import async_get_tracking
        result = await async_get_tracking("1Z999AA1012345678")
        
        assert result["status"] == "IN_TRANSIT"
        mock_client_instance.get.assert_awaited_once()

@pytest.mark.asyncio
async def test_gather_with_partial_failure():
    """Test that partial failure in gather is handled correctly."""
    
    async def mock_tracking(*args, **kwargs):
        return {"status": "IN_TRANSIT"}
    
    async def mock_order_fails(*args, **kwargs):
        raise ConnectionError("ERP is down")
    
    with patch("integration.enrichment.fetch_tracking", mock_tracking), \
         patch("integration.enrichment.fetch_order", mock_order_fails):
        
        from integration.enrichment import enrich_shipment
        result = await enrich_shipment("SHIP-001", "1Z999AA1012345678", "ORD-001")
        
        assert result.tracking is not None
        assert result.order is None
        assert any("ERP is down" in e for e in result.errors)
```

---

## Part 5: Testing HTTP Clients with respx and httpx

For async code using `httpx`, `respx` is the right tool:

```python
import pytest
import respx
import httpx
import pytest_asyncio

# ─── respx as context manager ─────────────────────────────────────────────────

@pytest.mark.asyncio
async def test_carrier_tracking_success():
    with respx.mock as mock:
        mock.get("https://carrier-api.internal/track/1Z999AA1012345678").mock(
            return_value=httpx.Response(
                200,
                json={
                    "tracking_id": "1Z999AA1012345678",
                    "status": "IN_TRANSIT",
                    "estimated_delivery": "2024-01-18"
                }
            )
        )
        
        from integration.carrier_client import CarrierAPIClient
        async with CarrierAPIClient("https://carrier-api.internal") as client:
            result = await client.get_tracking("1Z999AA1012345678")
        
        assert result.status == "IN_TRANSIT"
        assert mock.called

# ─── respx as decorator ───────────────────────────────────────────────────────

@pytest.mark.asyncio
@respx.mock
async def test_carrier_tracking_404():
    respx.get("https://carrier-api.internal/track/UNKNOWN123").mock(
        return_value=httpx.Response(404, json={"error": "Tracking number not found"})
    )
    
    from integration.carrier_client import CarrierAPIClient, TrackingNotFoundError
    async with CarrierAPIClient("https://carrier-api.internal") as client:
        with pytest.raises(TrackingNotFoundError) as exc_info:
            await client.get_tracking("UNKNOWN123")
        
        assert "UNKNOWN123" in str(exc_info.value)

# ─── Mocking multiple endpoints in one test ───────────────────────────────────

@pytest.mark.asyncio
@respx.mock
async def test_shipment_enrichment_concurrent():
    """Verify that enrichment makes all 3 API calls and returns combined result."""
    
    respx.get("https://carrier-api.internal/track/TRACK123").mock(
        return_value=httpx.Response(200, json={"status": "DELIVERED"})
    )
    respx.get("https://erp.internal/orders/ORD-001").mock(
        return_value=httpx.Response(200, json={"customer": "Acme Corp"})
    )
    respx.get("https://wms.internal/inventory/SKU-001").mock(
        return_value=httpx.Response(200, json={"available": 50})
    )
    
    from integration.enrichment import enrich_shipment
    result = await enrich_shipment("SHIP-001", "TRACK123", "ORD-001", "SKU-001")
    
    assert result.tracking["status"] == "DELIVERED"
    assert result.order["customer"] == "Acme Corp"
    assert result.inventory["available"] == 50

# ─── Testing timeout behavior ─────────────────────────────────────────────────

@pytest.mark.asyncio
@respx.mock
async def test_carrier_api_timeout_handling():
    """Verify that timeout exceptions are caught and handled gracefully."""
    
    respx.get("https://carrier-api.internal/track/TIMEOUT_TEST").mock(
        side_effect=httpx.TimeoutException("Request timed out")
    )
    
    from integration.carrier_client import CarrierAPIClient, CarrierAPIError
    async with CarrierAPIClient("https://carrier-api.internal", timeout=5.0) as client:
        with pytest.raises(CarrierAPIError) as exc_info:
            await client.get_tracking("TIMEOUT_TEST")
        
        assert "timed out" in str(exc_info.value).lower()
        assert exc_info.value.is_retryable is True  # timeouts should be retried

# ─── Testing retry logic with respx ──────────────────────────────────────────

@pytest.mark.asyncio
@respx.mock
async def test_retry_on_503():
    """Service should retry 503 responses up to 3 times."""
    
    route = respx.get("https://carrier-api.internal/track/RETRY_TEST")
    route.side_effect = [
        httpx.Response(503, json={"error": "Service temporarily unavailable"}),
        httpx.Response(503, json={"error": "Service temporarily unavailable"}),
        httpx.Response(200, json={"status": "IN_TRANSIT"}),  # succeeds on third try
    ]
    
    from integration.carrier_client import CarrierAPIClient
    async with CarrierAPIClient("https://carrier-api.internal") as client:
        result = await client.get_tracking_with_retry("RETRY_TEST", max_retries=3)
    
    assert result["status"] == "IN_TRANSIT"
    assert route.call_count == 3
```

---

## Part 6: Testing FastAPI Endpoints

### Synchronous TestClient

```python
import pytest
from fastapi.testclient import TestClient
from unittest.mock import patch, MagicMock

# ─── Basic endpoint testing ───────────────────────────────────────────────────

@pytest.fixture
def app():
    from integration.api import create_app
    return create_app()

@pytest.fixture
def client(app):
    return TestClient(app)

def test_health_check(client):
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"
    assert "version" in data

def test_create_shipment_success(client):
    with patch("integration.api.routes.shipments.create_shipment_in_db") as mock_create:
        mock_create.return_value = {
            "shipment_id": "SHIP-001",
            "status": "CREATED"
        }
        
        response = client.post(
            "/v1/shipments",
            json={
                "tracking_number": "1Z999AA1012345678",
                "carrier": "UPS",
                "weight_lbs": 12.5
            },
            headers={"Authorization": "Bearer test-api-key"}
        )
        
        assert response.status_code == 201
        data = response.json()
        assert data["shipment_id"] == "SHIP-001"

def test_create_shipment_validation_error(client):
    """Test that validation errors return 422 with clear messages."""
    
    response = client.post(
        "/v1/shipments",
        json={
            "tracking_number": "BAD",  # too short
            "carrier": "",             # empty
            "weight_lbs": -5,          # negative
        },
        headers={"Authorization": "Bearer test-api-key"}
    )
    
    assert response.status_code == 422
    data = response.json()
    
    # FastAPI returns validation errors in a specific format
    assert "detail" in data
    errors = data["detail"]
    assert len(errors) > 0
    
    # Check that each error has location and message
    for error in errors:
        assert "loc" in error
        assert "msg" in error
        assert "type" in error

def test_unauthorized_request(client):
    """Test that missing auth returns 401, not 500."""
    
    response = client.post(
        "/v1/shipments",
        json={"tracking_number": "1Z999AA1012345678", "carrier": "UPS", "weight_lbs": 12.5}
        # No Authorization header
    )
    
    assert response.status_code == 401

def test_get_nonexistent_shipment(client):
    """Test that 404 is returned for unknown shipment IDs."""
    
    with patch("integration.api.routes.shipments.get_shipment_from_db") as mock_get:
        mock_get.return_value = None
        
        response = client.get(
            "/v1/shipments/NONEXISTENT",
            headers={"Authorization": "Bearer test-api-key"}
        )
        
        assert response.status_code == 404
        data = response.json()
        assert "detail" in data
        assert "NONEXISTENT" in data["detail"]
```

### Async TestClient with httpx

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport

@pytest_asyncio.fixture
async def async_client():
    from integration.api import create_app
    app = create_app()
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client

@pytest.mark.asyncio
async def test_async_webhook_endpoint(async_client):
    """Test a webhook endpoint that processes asynchronously."""
    
    with patch("integration.api.routes.webhooks.process_webhook_async") as mock_process:
        mock_process.return_value = None
        
        response = await async_client.post(
            "/webhooks/carrier",
            json={
                "carrier": "FEDEX",
                "event_type": "DL",
                "tracking_number": "123456789012",
                "timestamp": "2024-01-18T14:22:00Z"
            },
            headers={"X-Carrier-Signature": "valid-signature"}
        )
        
        # Webhook endpoint should return 202 immediately
        # (processing happens in background)
        assert response.status_code == 202

@pytest.mark.asyncio
async def test_streaming_endpoint(async_client):
    """Test that streaming endpoint yields chunks."""
    
    async def mock_token_stream():
        for token in ["Hello", " world", "!"]:
            yield token
    
    with patch("integration.api.routes.llm.generate_stream", mock_token_stream):
        async with async_client.stream(
            "POST",
            "/v1/generate/stream",
            json={"prompt": "test prompt"},
            headers={"Authorization": "Bearer test-key"}
        ) as response:
            assert response.status_code == 200
            
            chunks = []
            async for chunk in response.aiter_text():
                if chunk.strip():
                    chunks.append(chunk)
            
            assert len(chunks) > 0
```

---

## Part 7: Testing with External Dependencies (AWS, Databases)

### Mocking AWS Services with moto

```python
import pytest
import boto3
from moto import mock_sqs, mock_s3

# ─── Mock SQS ─────────────────────────────────────────────────────────────────

@pytest.fixture
def sqs_queue(aws_credentials):
    with mock_sqs():
        sqs = boto3.client("sqs", region_name="us-east-1")
        queue = sqs.create_queue(QueueName="test-shipment-events")
        queue_url = queue["QueueUrl"]
        yield {"sqs": sqs, "queue_url": queue_url}

@pytest.fixture
def aws_credentials(monkeypatch):
    """Set fake AWS credentials for moto."""
    monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_SECURITY_TOKEN", "testing")
    monkeypatch.setenv("AWS_SESSION_TOKEN", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")

@mock_sqs
def test_shipment_event_published(aws_credentials):
    """Test that processing a shipment publishes an SQS event."""
    
    sqs = boto3.client("sqs", region_name="us-east-1")
    queue = sqs.create_queue(QueueName="shipment-events")
    queue_url = queue["QueueUrl"]
    
    from integration.events import publish_shipment_event
    publish_shipment_event(
        queue_url=queue_url,
        shipment_id="SHIP-001",
        event_type="SHIPMENT_CREATED"
    )
    
    # Read the message from the mock queue
    messages = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=1
    )
    
    assert len(messages["Messages"]) == 1
    import json
    body = json.loads(messages["Messages"][0]["Body"])
    assert body["shipment_id"] == "SHIP-001"
    assert body["event_type"] == "SHIPMENT_CREATED"

# ─── Mock S3 ──────────────────────────────────────────────────────────────────

@mock_s3
def test_shipment_report_upload(aws_credentials):
    """Test that daily report is uploaded to S3."""
    
    s3 = boto3.client("s3", region_name="us-east-1")
    s3.create_bucket(Bucket="shipment-reports")
    
    from integration.reporting import generate_and_upload_daily_report
    generate_and_upload_daily_report(
        bucket="shipment-reports",
        date="2024-01-15",
        shipments=[{"id": "SHIP-001"}, {"id": "SHIP-002"}]
    )
    
    # Verify the file was created
    response = s3.get_object(Bucket="shipment-reports", Key="reports/2024-01-15.csv")
    content = response["Body"].read().decode("utf-8")
    assert "SHIP-001" in content
    assert "SHIP-002" in content
```

### Testing with a Real Test Database

```python
import pytest
import asyncpg

# ─── Database fixtures ────────────────────────────────────────────────────────

@pytest.fixture(scope="session")
def db_url():
    """Use a real test database. Set up in docker-compose.test.yml."""
    import os
    return os.environ.get(
        "TEST_DATABASE_URL",
        "postgresql://test:test@localhost:5433/integration_test"
    )

@pytest_asyncio.fixture(scope="session")
async def db_pool(db_url):
    """Session-scoped connection pool."""
    pool = await asyncpg.create_pool(db_url, min_size=2, max_size=5)
    yield pool
    await pool.close()

@pytest_asyncio.fixture
async def db_transaction(db_pool):
    """
    Each test gets a transaction that's rolled back after.
    This keeps tests isolated without needing to reset the database.
    """
    async with db_pool.acquire() as connection:
        transaction = connection.transaction()
        await transaction.start()
        yield connection
        await transaction.rollback()  # always roll back — tests don't affect each other

@pytest.mark.asyncio
@pytest.mark.integration
async def test_create_and_retrieve_shipment(db_transaction):
    """Test database round-trip — only runs with integration marker."""
    
    from integration.repository import ShipmentRepository
    repo = ShipmentRepository(db_transaction)
    
    shipment_id = await repo.create_shipment({
        "tracking_number": "1Z999AA1012345678",
        "carrier": "UPS",
        "status": "CREATED",
        "weight_lbs": 12.5
    })
    
    retrieved = await repo.get_shipment(shipment_id)
    
    assert retrieved is not None
    assert retrieved["carrier"] == "UPS"
    assert retrieved["status"] == "CREATED"
    # After the test, the transaction is rolled back — DB is clean
```

---

## Part 8: Testing Webhook Handlers — A Complete Example

```python
"""
test_webhook_handler.py

Full test suite for a carrier webhook endpoint.
This is the pattern you'll use for every customer integration.
"""

import pytest
import json
import hmac
import hashlib
from unittest.mock import patch, AsyncMock
from fastapi.testclient import TestClient
from datetime import datetime

# ─── Fixtures ────────────────────────────────────────────────────────────────

@pytest.fixture
def webhook_secret():
    return "test-webhook-secret-12345"

@pytest.fixture
def app(webhook_secret):
    from integration.api import create_app
    app = create_app(webhook_secret=webhook_secret)
    return app

@pytest.fixture
def client(app):
    return TestClient(app, raise_server_exceptions=False)

def make_signature(payload: dict, secret: str) -> str:
    """Helper: generate the HMAC-SHA256 signature that carriers use."""
    body = json.dumps(payload, separators=(",", ":")).encode()
    return hmac.new(
        secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()

@pytest.fixture
def fedex_delivery_payload():
    return {
        "carrier": "FEDEX",
        "event_type": "DL",
        "tracking_number": "123456789012",
        "timestamp": "2024-01-18T14:22:00Z",
        "location": "Dallas, TX",
        "delivery_details": {
            "signed_by": "J. SMITH",
            "delivery_location": "FRONT_DOOR"
        }
    }

# ─── Happy path ───────────────────────────────────────────────────────────────

def test_webhook_valid_delivery_event(client, fedex_delivery_payload, webhook_secret):
    """Valid webhook event is processed and 202 returned."""
    
    signature = make_signature(fedex_delivery_payload, webhook_secret)
    
    with patch("integration.api.routes.webhooks.update_shipment_status") as mock_update:
        mock_update.return_value = True
        
        response = client.post(
            "/webhooks/fedex",
            json=fedex_delivery_payload,
            headers={
                "X-FedEx-Signature": signature,
                "Content-Type": "application/json"
            }
        )
    
    assert response.status_code == 202
    data = response.json()
    assert data["status"] == "accepted"
    assert "event_id" in data  # should return a reference ID

# ─── Authentication ───────────────────────────────────────────────────────────

def test_webhook_invalid_signature(client, fedex_delivery_payload):
    """Invalid signature should return 401, not 500."""
    
    response = client.post(
        "/webhooks/fedex",
        json=fedex_delivery_payload,
        headers={
            "X-FedEx-Signature": "invalid-signature-abcdef",
            "Content-Type": "application/json"
        }
    )
    
    assert response.status_code == 401
    data = response.json()
    assert "signature" in data["detail"].lower()

def test_webhook_missing_signature(client, fedex_delivery_payload):
    """Missing signature header should return 401."""
    
    response = client.post(
        "/webhooks/fedex",
        json=fedex_delivery_payload
        # No signature header
    )
    
    assert response.status_code == 401

# ─── Validation ───────────────────────────────────────────────────────────────

def test_webhook_malformed_json(client, webhook_secret):
    """Malformed JSON should return 400, not 500."""
    
    body = b"this is not json {"
    signature = hmac.new(webhook_secret.encode(), body, hashlib.sha256).hexdigest()
    
    response = client.post(
        "/webhooks/fedex",
        content=body,
        headers={
            "X-FedEx-Signature": signature,
            "Content-Type": "application/json"
        }
    )
    
    assert response.status_code == 400
    assert "parse" in response.json()["detail"].lower()

def test_webhook_missing_required_fields(client, webhook_secret):
    """Payload missing required fields should return 422."""
    
    incomplete_payload = {
        "carrier": "FEDEX",
        # missing event_type, tracking_number, timestamp
    }
    signature = make_signature(incomplete_payload, webhook_secret)
    
    response = client.post(
        "/webhooks/fedex",
        json=incomplete_payload,
        headers={"X-FedEx-Signature": signature}
    )
    
    assert response.status_code == 422
    errors = response.json()["detail"]
    # Verify all missing fields are named in the error
    error_fields = {str(e["loc"][-1]) for e in errors}
    assert "event_type" in error_fields
    assert "tracking_number" in error_fields

# ─── Idempotency ──────────────────────────────────────────────────────────────

def test_webhook_duplicate_event_idempotent(client, fedex_delivery_payload, webhook_secret):
    """Processing the same event twice should be safe (idempotent)."""
    
    signature = make_signature(fedex_delivery_payload, webhook_secret)
    
    with patch("integration.api.routes.webhooks.is_duplicate_event") as mock_dup, \
         patch("integration.api.routes.webhooks.update_shipment_status") as mock_update:
        
        # First call: not a duplicate
        mock_dup.return_value = False
        response1 = client.post(
            "/webhooks/fedex",
            json=fedex_delivery_payload,
            headers={"X-FedEx-Signature": signature}
        )
        assert response1.status_code == 202
        assert mock_update.call_count == 1
        
        # Second call: duplicate
        mock_dup.return_value = True
        response2 = client.post(
            "/webhooks/fedex",
            json=fedex_delivery_payload,
            headers={"X-FedEx-Signature": signature}
        )
        # Should still return 202 (not an error), but not process again
        assert response2.status_code == 202
        assert mock_update.call_count == 1  # NOT called again

# ─── Error handling ───────────────────────────────────────────────────────────

def test_webhook_downstream_failure_does_not_expose_error(
    client, fedex_delivery_payload, webhook_secret
):
    """If downstream processing fails, return 500 WITHOUT exposing internal details."""
    
    signature = make_signature(fedex_delivery_payload, webhook_secret)
    
    with patch("integration.api.routes.webhooks.update_shipment_status") as mock_update:
        mock_update.side_effect = Exception("DB connection pool exhausted at line 42 of db.py")
        
        response = client.post(
            "/webhooks/fedex",
            json=fedex_delivery_payload,
            headers={"X-FedEx-Signature": signature}
        )
    
    assert response.status_code == 500
    data = response.json()
    
    # Should have a correlation ID for support tracing
    assert "correlation_id" in data or "request_id" in data
    
    # Should NOT expose internal implementation details
    assert "db.py" not in str(data)
    assert "line 42" not in str(data)
    assert "pool exhausted" not in str(data)

# ─── Parametrized happy path for all carriers ─────────────────────────────────

@pytest.mark.parametrize("carrier,event_header", [
    ("FEDEX", "X-FedEx-Signature"),
    ("UPS", "X-UPS-Signature"),
    ("DHL", "X-DHL-Signature"),
])
def test_webhook_all_carriers(client, webhook_secret, carrier, event_header):
    payload = {
        "carrier": carrier,
        "event_type": "STATUS_UPDATE",
        "tracking_number": "TRACK123",
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "status": "IN_TRANSIT"
    }
    signature = make_signature(payload, webhook_secret)
    
    with patch("integration.api.routes.webhooks.update_shipment_status"):
        response = client.post(
            f"/webhooks/{carrier.lower()}",
            json=payload,
            headers={event_header: signature}
        )
    
    assert response.status_code == 202
```

---

## Part 9: pytest-asyncio Configuration

```python
# conftest.py — session-level async config

import pytest
import asyncio

# ─── Configure asyncio mode ───────────────────────────────────────────────────
# In pyproject.toml:
# [tool.pytest.ini_options]
# asyncio_mode = "auto"  # all async tests are automatically treated as asyncio tests

# OR per-file:
pytestmark = pytest.mark.asyncio  # marks all tests in this module as async

# ─── Custom event loop for session-scoped async fixtures ──────────────────────

@pytest.fixture(scope="session")
def event_loop():
    """
    Create a session-scoped event loop.
    Required when you have session-scoped async fixtures.
    """
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

# ─── Async fixture example ───────────────────────────────────────────────────

@pytest_asyncio.fixture
async def async_http_client():
    """Async fixture — create client, yield, close."""
    import httpx
    async with httpx.AsyncClient() as client:
        yield client

@pytest.mark.asyncio
async def test_something_async(async_http_client):
    # async_http_client is an open httpx.AsyncClient
    ...
```

---

## Part 10: Testing Pydantic Validation Errors

```python
import pytest
from pydantic import ValidationError

def test_validation_error_messages_are_customer_friendly():
    """
    FDE priority: validation errors must be human-readable.
    This test documents our error message contract.
    """
    from integration.models import InboundOrder
    
    with pytest.raises(ValidationError) as exc_info:
        InboundOrder.model_validate({
            "orderId": "ORD-001",
            "serviceLevel": "TELEPORTATION",  # invalid
            "shipFrom": {
                "name": "Warehouse",
                "addressLine1": "100 Main St",
                "city": "Chicago",
                "stateProvince": "IL",
                "postalCode": "ABC",  # invalid ZIP
            },
            "shipTo": {
                "name": "Customer",
                "addressLine1": "200 Oak Ave",
                "city": "Dallas",
                "stateProvince": "TX",
                "postalCode": "75201",
            },
            "lineItems": []  # empty — invalid
        })
    
    errors = exc_info.value.errors()
    error_locs = {".".join(str(l) for l in e["loc"]) for e in errors}
    
    # Verify all invalid fields are reported
    assert any("serviceLevel" in loc or "service_level" in loc for loc in error_locs), \
        f"Expected serviceLevel error, got: {error_locs}"
    assert any("postalCode" in loc or "postal_code" in loc for loc in error_locs), \
        f"Expected postalCode error, got: {error_locs}"
    assert any("lineItems" in loc or "line_items" in loc for loc in error_locs), \
        f"Expected lineItems error, got: {error_locs}"

@pytest.mark.parametrize("field,value,expected_error_contains", [
    ("weight_lbs", -5, "greater than 0"),
    ("carrier", "", "at least 1"),
    ("tracking_number", "AB", "too short"),
    ("ship_date", "not-a-date", "date"),
])
def test_shipment_field_validation_messages(field, value, expected_error_contains):
    """Each invalid field produces a useful error message."""
    from integration.models import Shipment
    
    valid_base = {
        "shipment_id": "SHIP-001",
        "tracking_number": "1Z999AA1012345678",
        "carrier": "UPS",
        "weight_lbs": 12.5,
        "ship_date": "2024-01-15",
    }
    valid_base[field] = value
    
    with pytest.raises(ValidationError) as exc_info:
        Shipment.model_validate(valid_base)
    
    error_messages = " ".join(e["msg"] for e in exc_info.value.errors())
    assert expected_error_contains.lower() in error_messages.lower(), \
        f"Expected '{expected_error_contains}' in error messages: {error_messages}"
```

---

## FDE Context Callouts

> **FDE Context: Tests Document Customer Assumptions**
>
> Every Pydantic validator, every date parsing workaround, every null-handling branch exists because a specific customer sent you specific bad data. Write a test for each one. When a new engineer joins, those tests explain "why does this code handle the string 'NULL' as None?" The answer is in the test name: `test_legacy_erp_sends_null_string_for_missing_fields`.

> **FDE Context: Test the Error Messages, Not Just the Errors**
>
> It's not enough to test that a 422 is returned — test the body. When a customer's developer is integrating, they'll see your error messages in their logs. `"value is not a valid integer"` is unhelpful. `"Field 'weight_lbs': expected a number (e.g. 12.5), received 'heavy'"` is actionable. Write tests that assert on error message content.

> **FDE Context: Always Test the Unhappy Path First**
>
> In a customer deployment, the most expensive bugs are the ones where your service crashes on unexpected input. Before writing a test for the happy path, write tests for: empty payload, wrong content type, auth missing, rate limit exceeded, downstream service down, malformed JSON, and Unicode in unexpected fields. Then write the happy path.

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| `ScopeMismatch` error with async fixtures | Mixing session-scope with function-scope async | Ensure session-scope event loop fixture is defined |
| `RuntimeError: Event loop is closed` | Fixture cleanup runs after event loop closes | Use `pytest_asyncio.fixture` not `pytest.fixture` for async fixtures |
| Mock not being called | Patching wrong import path | Patch where name is **used**, not where it's **defined** |
| `respx` requests passing through to real APIs | `respx.mock` context not applied | Check that `@respx.mock` or `with respx.mock:` wraps the call |
| Tests pass locally, fail in CI | Hardcoded port/host in test | Use environment variables for test DB URLs |
| Pydantic validation tests pass but behavior is wrong | Testing model parse, not the validator | Add `mode="before"` test to ensure coercion happens |
| `moto` mock not intercepting boto3 calls | boto3 client created before `@mock_sqs` activates | Create boto3 clients inside the mock context |

---

## Interview Angle

**Q: "What's a fixture factory and when would you use one?"**

Great answer: "A fixture factory is a fixture that returns a callable instead of a value. Instead of one hardcoded test object, you get a function you can call multiple times in the same test with different parameters. I use it when tests need variations of the same object — for example, a `make_shipment()` factory where some tests need a heavy shipment, others need an international shipment, and others need a shipment with missing fields. It avoids having a dozen separate `sample_shipment_*` fixtures."

**Q: "How do you test retry logic?"**

Great answer: "I use `MagicMock.side_effect` with a list of exceptions and success responses. First call raises `ConnectionError`, second raises `Timeout`, third returns a success response. Then I assert `mock.call_count == 3` to verify all retries happened, and assert the final result is correct. I also test the failure case: when all retries are exhausted, the right exception is raised. And I test the exponential backoff delays by patching `time.sleep` and asserting it was called with increasing values."

**Q: "How do you test code that calls multiple external APIs concurrently?"**

Great answer: "I use `respx` for httpx-based async clients, which intercepts HTTP calls at the transport level — no real network requests happen. I set up mock routes for each API endpoint with specific responses. Then in the test, I verify the business logic worked correctly and that all expected API calls were made. For testing partial failure, I configure one route to raise a `TimeoutException` and verify the response still includes the data from the other two successful calls."

---

## Practice Exercise

**Scenario:** You have an `OrderProcessor` class that:
1. Validates an incoming order with Pydantic
2. Calls a carrier API to get shipping rates (async, httpx)
3. Calls an ERP API to reserve inventory (async, httpx)
4. Saves the order to a database
5. Publishes an SQS message

**Task 1:** Write a test suite using `pytest` and `respx` that covers:
- Happy path: valid order processed successfully
- Carrier API returns 429: verify retry happens
- ERP API times out: verify order is rejected with a meaningful error
- Database save fails: verify the error is logged and the SQS message is NOT published

**Task 2:** Write parametrized tests for all date formats your customer's ERP sends (you've identified 4 formats from their docs).

**Task 3:** Write a fixture that creates a realistic messy payload with 10 line items, some with null SKUs, mixed-case carrier names, and dates in multiple formats. Use this fixture across 3+ different test cases.

**Task 4:** Add a test that verifies that when Pydantic rejects a payload, the error response body contains all invalid field names (not just the first one), and none of them expose internal Python module paths.

---

*Previous: [02-pydantic-type-hints.md](./02-pydantic-type-hints.md) | Next: [04-error-handling.md](./04-error-handling.md)*
