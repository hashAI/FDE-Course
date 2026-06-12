# Distributed Tracing with OpenTelemetry

> **FDE Principle**: When a customer says "shipment creation is slow — it takes 10 seconds", a trace waterfall shows you in 30 seconds that 8 of those 10 seconds are spent waiting for the carrier's rate API. That's a conversation-changing superpower.

**Related files:**
- [01-structured-logging.md](./01-structured-logging.md) — Logs provide the detail; tracing provides the structure
- [03-metrics-prometheus-grafana.md](./03-metrics-prometheus-grafana.md) — Metrics tell you there's a problem; traces tell you where
- [04-log-aggregation.md](./04-log-aggregation.md) — Where traces are stored and queried

---

## Table of Contents

1. [Why Distributed Tracing](#1-why-distributed-tracing)
2. [Core Concepts](#2-core-concepts)
3. [OpenTelemetry Overview](#3-opentelemetry-overview)
4. [Auto-Instrumentation](#4-auto-instrumentation)
5. [Manual Instrumentation](#5-manual-instrumentation)
6. [Trace Propagation](#6-trace-propagation)
7. [The OTel Collector](#7-the-otel-collector)
8. [Backend Options](#8-backend-options)
9. [Sampling Strategies](#9-sampling-strategies)
10. [Connecting Tracing to Logging](#10-connecting-tracing-to-logging)
11. [Full Working Example: FastAPI with OTel](#11-full-working-example-fastapi-with-otel)
12. [FDE Context: Debugging Slow Shipments](#12-fde-context-debugging-slow-shipments)
13. [Common Failure Modes](#13-common-failure-modes)
14. [Interview Angle](#14-interview-angle)
15. [Practice Exercise](#15-practice-exercise)

---

## 1. Why Distributed Tracing

### The Problem: You Have Logs But Still Can't See the Picture

Even with excellent structured logging (see [01-structured-logging.md](./01-structured-logging.md)), logs tell you *what* each service did — they don't show you the *relationship* between those actions.

Consider this scenario: a shipment creation request takes 10 seconds. Your order service logs show `duration_ms: 9823`. But the order service calls 4 downstream services:
- Carrier rate API
- Address validation API
- Inventory check service
- 3PL fulfillment API

Which one is slow? Log entries don't show you the dependency graph or which call is on the critical path.

### Without Tracing: The Blind Spots

```
Request arrives at API Gateway (T+0ms)
  → Order Service receives request (T+12ms)
    → Something happens for 9800ms
  → Order Service returns response (T+9823ms)
```

You know something is slow. You don't know where. You'd need to manually correlate timestamps across log entries in 4 different services and reconstruct the call graph by hand.

### With Tracing: The Waterfall Diagram

```
[api-gateway          ] ████████████████████████████████████  9823ms
  [order-service      ]   ██████████████████████████████████  9801ms
    [validate_order   ]   ██  12ms
    [check_inventory  ]     ████  48ms
    [get_carrier_rates]       ████████████████████████  8012ms  ← HERE
    [create_shipment  ]                                 ██  229ms
    [notify_customer  ]                                   █  91ms
```

In 5 seconds you see: **the carrier rate API is taking 8 seconds**. That's 82% of total latency. Now you can have a productive conversation with the customer about whether to cache carrier rates, use a different carrier, or set a timeout.

---

## 2. Core Concepts

### Trace

A **trace** represents the complete journey of a single request as it travels through your distributed system. It has:
- A globally unique **Trace ID** (16-byte hex string, e.g., `4bf92f3577b34da6a3ce929d0e0e4736`)
- A collection of **spans**
- Start and end time (derived from spans)

Think of a trace as a tree: the root is the initial request, branches are service calls, leaves are individual operations.

### Span

A **span** represents a single unit of work. It has:
- **Span ID**: uniquely identifies this span within the trace
- **Parent Span ID**: which span created this one (null for root span)
- **Trace ID**: which trace this belongs to
- **Name**: what operation this represents (e.g., `"GET /api/v1/orders"`, `"create_shipment"`)
- **Start time** and **end time**
- **Status**: `OK`, `ERROR`, or `UNSET`
- **Attributes**: key-value metadata (`http.method`, `db.statement`, `order_id`)
- **Events**: timestamped annotations within the span (`"retry_attempt_1"`)
- **Links**: connections to other traces (useful for async operations)

```python
# Conceptually, a span looks like this:
{
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id": "00f067aa0ba902b7",
    "parent_span_id": "a2fb4a1d1a96d312",  # null if root span
    "name": "create_shipment",
    "start_time": "2024-01-15T14:23:10.100Z",
    "end_time": "2024-01-15T14:23:10.442Z",
    "duration_ms": 342,
    "status": "OK",
    "attributes": {
        "order.id": "ORD-12345",
        "customer.id": "acme-corp",
        "shipment.carrier": "fedex",
        "http.method": "POST",
        "http.url": "https://tpl.example.com/shipments",
        "http.status_code": 201,
    },
    "events": [
        {
            "timestamp": "2024-01-15T14:23:10.200Z",
            "name": "retry_attempt",
            "attributes": {"attempt": 1, "reason": "timeout"}
        }
    ]
}
```

### Parent/Child Relationship: The Call Tree

Spans form a tree through parent-child relationships:

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736
│
└── [ROOT] POST /api/v1/orders (span: a2fb4a1d) — order-service — 9823ms
    ├── validate_order (span: b3fc5b2e) — order-service — 12ms
    ├── check_inventory (span: c4gd6c3f) — order-service — 48ms
    │   └── SELECT * FROM inventory (span: d5he7d4g) — postgres — 45ms
    ├── GET https://carrier-api.com/rates (span: e6if8e5h) — httpx — 8012ms
    └── POST https://tpl.example.com/shipments (span: f7jg9f6i) — httpx — 229ms
        └── [ROOT] POST /shipments (span: g8kh0g7j) — tpl-connector — 220ms
            └── INSERT INTO shipments (span: h9li1h8k) — postgres — 18ms
```

### Span Attributes vs Events

**Attributes** describe static properties of the work being done:
```python
span.set_attribute("order.id", "ORD-12345")
span.set_attribute("http.status_code", 200)
span.set_attribute("db.system", "postgresql")
```

**Events** are point-in-time occurrences within a span:
```python
span.add_event("cache_miss", attributes={"cache_key": "order:12345"})
span.add_event("retry_attempt", attributes={"attempt": 2, "delay_ms": 500})
span.add_event("rate_limit_hit", attributes={"retry_after_seconds": 60})
```

### Span Status

```python
from opentelemetry.trace import StatusCode

# Default — not explicitly set
span.set_status(StatusCode.UNSET)

# Successful completion
span.set_status(StatusCode.OK)

# Error occurred
span.set_status(StatusCode.ERROR, description="Connection refused to carrier API")
```

---

## 3. OpenTelemetry Overview

### Why OpenTelemetry

Before OTel, every observability vendor (Datadog, Jaeger, Zipkin, New Relic, etc.) had their own SDK. If you instrumented with the Datadog SDK and later wanted to switch to Honeycomb, you had to re-instrument your entire codebase.

OpenTelemetry solves this with a **vendor-neutral standard**:
- **Instrument once** using the OTel SDK
- **Export anywhere** — change backends without changing code
- **Industry standard** — backed by CNCF, supported by all major observability vendors

The OTel architecture has three layers:

```
Your Code
    ↓ uses
OTel API (opentelemetry-api)     ← thin interface, safe to depend on
    ↓ implements
OTel SDK (opentelemetry-sdk)     ← actual implementation, configure at startup
    ↓ exports to
OTel Collector / Backends        ← Jaeger, Tempo, Datadog, X-Ray, etc.
```

### Installation

```bash
# Core OTel packages
pip install opentelemetry-api opentelemetry-sdk

# OTLP exporter (sends traces to collector or backend)
pip install opentelemetry-exporter-otlp-proto-grpc
# or HTTP:
pip install opentelemetry-exporter-otlp-proto-http

# Auto-instrumentation for common frameworks
pip install opentelemetry-instrumentation-fastapi
pip install opentelemetry-instrumentation-httpx
pip install opentelemetry-instrumentation-sqlalchemy
pip install opentelemetry-instrumentation-redis

# Convenience wrapper
pip install opentelemetry-distro
```

### The Three Signals: Traces, Metrics, Logs

OpenTelemetry covers all three observability signals:
- **Traces**: call graphs, spans, latency — covered in this file
- **Metrics**: counters, gauges, histograms — see [03-metrics-prometheus-grafana.md](./03-metrics-prometheus-grafana.md)
- **Logs**: structured log records with trace correlation — see [01-structured-logging.md](./01-structured-logging.md)

This file focuses on traces.

---

## 4. Auto-Instrumentation

Auto-instrumentation wraps common libraries so they automatically create spans without you changing application code.

### What Auto-Instrumentation Covers

| Library | What it traces |
|---------|----------------|
| `opentelemetry-instrumentation-fastapi` | Incoming HTTP requests (one span per request) |
| `opentelemetry-instrumentation-httpx` | Outgoing HTTP requests (one span per call) |
| `opentelemetry-instrumentation-sqlalchemy` | Database queries (one span per query) |
| `opentelemetry-instrumentation-redis` | Redis operations |
| `opentelemetry-instrumentation-celery` | Celery task execution |
| `opentelemetry-instrumentation-boto3sqs` | SQS send/receive |

### Setting Up Auto-Instrumentation

```python
# tracing.py — Configure OTel tracing at application startup
import os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased


def configure_tracing(
    service_name: str,
    service_version: str,
    environment: str,
    otlp_endpoint: str = None,
    sample_rate: float = 1.0,
) -> trace.Tracer:
    """
    Configure OpenTelemetry tracing.
    
    Args:
        service_name: Your service name (e.g., "order-service")
        service_version: Your service version (e.g., "1.4.2")
        environment: "production", "staging", "development"
        otlp_endpoint: Where to send traces (e.g., "http://otel-collector:4317")
        sample_rate: 1.0 = trace everything, 0.1 = trace 10% of requests
    
    Call this once at application startup, before creating FastAPI app.
    """
    
    # Resource: metadata about the service
    # This appears on every span from this service
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": environment,
        "host.name": os.environ.get("HOSTNAME", "unknown"),
    })
    
    # Sampler: which traces to record
    # ParentBased: respect the parent's sampling decision (important for distributed tracing)
    if sample_rate >= 1.0:
        from opentelemetry.sdk.trace.sampling import ALWAYS_ON
        sampler = ALWAYS_ON
    else:
        sampler = ParentBased(root=TraceIdRatioBased(sample_rate))
    
    # Tracer Provider: the entry point for creating tracers
    provider = TracerProvider(resource=resource, sampler=sampler)
    
    # Exporters: where to send spans
    if otlp_endpoint:
        # Send to OTel Collector or directly to a backend
        otlp_exporter = OTLPSpanExporter(
            endpoint=otlp_endpoint,
            # For gRPC, endpoint is "host:port" (no scheme)
            # For HTTP, use OTLPSpanExporter from otlp.proto.http
        )
        provider.add_span_processor(
            BatchSpanProcessor(
                otlp_exporter,
                max_queue_size=2048,
                max_export_batch_size=512,
                export_timeout_millis=30_000,
            )
        )
    
    if environment in ("development", "local"):
        # Also print traces to console in development
        provider.add_span_processor(
            BatchSpanProcessor(ConsoleSpanExporter())
        )
    
    # Register globally — makes trace.get_tracer() work anywhere
    trace.set_tracer_provider(provider)
    
    # Auto-instrument libraries
    # These wrap the libraries so they automatically create spans
    FastAPIInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()
    # SQLAlchemyInstrumentor().instrument(engine=engine)  # call after engine creation
    
    return trace.get_tracer(service_name, service_version)


# Get a tracer for manual instrumentation
# (auto-instrumentation handles FastAPI and httpx automatically)
tracer = configure_tracing(
    service_name=os.environ.get("SERVICE_NAME", "order-service"),
    service_version=os.environ.get("SERVICE_VERSION", "unknown"),
    environment=os.environ.get("ENVIRONMENT", "production"),
    otlp_endpoint=os.environ.get("OTLP_ENDPOINT"),
    sample_rate=float(os.environ.get("TRACE_SAMPLE_RATE", "1.0")),
)
```

### What Auto-Instrumentation Creates

After calling `FastAPIInstrumentor().instrument()`, every HTTP request to FastAPI automatically creates a span like:

```json
{
  "name": "POST /api/v1/orders",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "a2fb4a1d1a96d312",
  "start_time": "2024-01-15T14:23:10.000Z",
  "duration_ms": 342,
  "attributes": {
    "http.method": "POST",
    "http.url": "http://localhost:8000/api/v1/orders",
    "http.scheme": "http",
    "http.host": "localhost:8000",
    "http.target": "/api/v1/orders",
    "http.status_code": 201,
    "http.user_agent": "python-httpx/0.25.0",
    "net.host.port": 8000
  }
}
```

And `HTTPXClientInstrumentor().instrument()` creates a child span for every outbound HTTP call:

```json
{
  "name": "POST",
  "parent_span_id": "a2fb4a1d1a96d312",
  "attributes": {
    "http.method": "POST",
    "http.url": "https://tpl.example.com/api/v2/shipments",
    "http.status_code": 201
  }
}
```

---

## 5. Manual Instrumentation

Auto-instrumentation gives you framework-level spans. Manual instrumentation adds business-context spans within your application code.

### Creating Spans

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def process_order(order_id: str, customer_id: str, items: list) -> dict:
    """Process an order with full tracing."""
    
    # Context manager creates a span for this function
    # The span automatically becomes the parent for any spans created inside
    with tracer.start_as_current_span("process_order") as span:
        # Set attributes — metadata about this operation
        span.set_attribute("order.id", order_id)
        span.set_attribute("customer.id", customer_id)
        span.set_attribute("order.item_count", len(items))
        
        # Validate order
        with tracer.start_as_current_span("validate_order") as validate_span:
            try:
                validate_order_items(items)
                validate_span.set_attribute("validation.result", "pass")
                validate_span.set_status(trace.StatusCode.OK)
            except ValueError as e:
                validate_span.set_attribute("validation.result", "fail")
                validate_span.set_attribute("validation.error", str(e))
                validate_span.set_status(trace.StatusCode.ERROR, str(e))
                validate_span.record_exception(e)
                raise
        
        # Check inventory
        with tracer.start_as_current_span("check_inventory") as inv_span:
            inventory_results = []
            for item in items:
                available = await check_item_inventory(item["sku"])
                inventory_results.append({"sku": item["sku"], "available": available})
                
            all_available = all(r["available"] for r in inventory_results)
            inv_span.set_attribute("inventory.all_available", all_available)
            
            if not all_available:
                out_of_stock = [r["sku"] for r in inventory_results if not r["available"]]
                inv_span.add_event("inventory_shortage", {
                    "out_of_stock_skus": str(out_of_stock)
                })
        
        # Add a span event for a significant moment
        span.add_event("validation_and_inventory_complete")
        
        return {"order_id": order_id, "status": "processed"}
```

### The Context Manager vs. start_span

There are two ways to create spans:

```python
# Option 1: Context manager (recommended for most cases)
# Automatically sets the span as "current" so child spans know their parent
with tracer.start_as_current_span("my_operation") as span:
    span.set_attribute("foo", "bar")
    do_work()

# Option 2: Explicit start/end (for async or complex flow control)
span = tracer.start_span("my_operation")
try:
    with trace.use_span(span, end_on_exit=True):
        span.set_attribute("foo", "bar")
        await do_async_work()
except Exception as e:
    span.record_exception(e)
    span.set_status(trace.StatusCode.ERROR, str(e))
    raise
```

### Recording Exceptions Properly

```python
async def call_carrier_api(shipment_data: dict) -> dict:
    with tracer.start_as_current_span("call_carrier_rate_api") as span:
        span.set_attribute("carrier.name", shipment_data.get("carrier"))
        span.set_attribute("shipment.weight_kg", shipment_data.get("weight"))
        
        try:
            response = await carrier_client.get_rates(shipment_data)
            span.set_attribute("carrier.rate_count", len(response["rates"]))
            span.set_attribute("carrier.cheapest_rate_usd", response["rates"][0]["price"])
            span.set_status(trace.StatusCode.OK)
            return response
            
        except httpx.TimeoutException as e:
            # Record the exception — this adds stack trace to the span
            span.record_exception(e)
            span.set_status(trace.StatusCode.ERROR, "Carrier API timeout")
            span.set_attribute("error.type", "timeout")
            span.set_attribute("timeout.seconds", 30)
            raise
            
        except httpx.HTTPStatusError as e:
            span.record_exception(e)
            span.set_status(trace.StatusCode.ERROR, f"HTTP {e.response.status_code}")
            span.set_attribute("error.type", "http_error")
            span.set_attribute("http.status_code", e.response.status_code)
            # Include response body for debugging (truncated)
            span.set_attribute("error.response_body", e.response.text[:500])
            raise
```

### Adding Business-Context Attributes

The most valuable traces include business context, not just technical metadata:

```python
# Technical attributes only — less useful for FDE debugging
span.set_attribute("http.method", "POST")
span.set_attribute("http.status_code", 200)

# Business context — much more useful when debugging with customer
span.set_attribute("order.id", order_id)
span.set_attribute("order.customer_id", customer_id)
span.set_attribute("order.total_value_usd", total_value)
span.set_attribute("order.item_count", len(items))
span.set_attribute("shipment.carrier", carrier)
span.set_attribute("shipment.service_level", "express")
span.set_attribute("edi.transaction_set", "856")  # for logistics/EDI contexts
span.set_attribute("wms.warehouse_id", warehouse_id)
```

When you sit with a customer and filter traces by `order.customer_id = "acme-corp"` and `order.total_value_usd > 1000`, you see all high-value orders for that customer. That's a conversation you can't have with technical spans alone.

### Span Links: Connecting Async Operations

When a message is put on a queue and processed later, the processing span can't have the producer span as its parent (they run at different times). Span **links** solve this:

```python
import json
from opentelemetry import trace
from opentelemetry.propagate import extract

# Producer: put a message on SQS
async def enqueue_order(order_id: str, queue_url: str):
    with tracer.start_as_current_span("enqueue_order") as span:
        # Inject current trace context into the message
        carrier = {}
        from opentelemetry.propagate import inject
        inject(carrier)  # adds traceparent, tracestate
        
        message_body = json.dumps({
            "order_id": order_id,
            "trace_context": carrier,  # embed trace context in message
        })
        
        await sqs.send_message(QueueUrl=queue_url, MessageBody=message_body)
        span.set_attribute("order.id", order_id)
        span.set_attribute("messaging.destination", queue_url)


# Consumer: process the message — link back to the producer's trace
async def process_queued_order(message: dict):
    message_body = json.loads(message["Body"])
    
    # Extract the trace context from the message
    parent_context = extract(message_body.get("trace_context", {}))
    
    # Create a new root span for the consumer, with a link to the producer
    with tracer.start_as_current_span(
        "process_queued_order",
        links=[trace.Link(
            trace.get_current_span(parent_context).get_span_context()
        )]
    ) as span:
        span.set_attribute("order.id", message_body["order_id"])
        span.set_attribute("messaging.message_id", message.get("MessageId"))
        
        # Process the order...
        await order_service.process(message_body["order_id"])
```

---

## 6. Trace Propagation

### The W3C traceparent Header

When service A calls service B, it needs to tell service B "you're part of trace X, your parent span is Y." This is done via the `traceparent` HTTP header, standardized by the W3C.

Format: `00-{trace_id}-{parent_span_id}-{flags}`

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ↑  ↑                                ↑                ↑
             version  trace ID (16 bytes hex)   parent span ID   flags (01=sampled)
```

### How Propagation Works

With `HTTPXClientInstrumentor().instrument()`, propagation is **automatic** — the OTel SDK injects `traceparent` into every outbound httpx request:

```python
# You write this:
response = await httpx_client.post("/shipments", json=payload)

# OTel automatically adds this header:
# traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-a2fb4a1d1a96d312-01
```

And when the receiving service also uses OTel with `FastAPIInstrumentor().instrument()`, it automatically extracts the `traceparent` header and makes the incoming span a child of the caller's span.

### Manual Propagation (When Not Using Auto-Instrumentation)

```python
from opentelemetry.propagate import inject, extract
from opentelemetry import context

# INJECTING: add trace context to outbound request headers
def get_trace_headers() -> dict:
    """Get headers with current trace context for outbound requests."""
    headers = {}
    inject(headers)  # adds traceparent and tracestate
    return headers

# Usage:
response = await httpx_client.post(
    "/shipments",
    json=payload,
    headers=get_trace_headers()
)


# EXTRACTING: restore trace context from inbound request headers
def extract_trace_context(headers: dict):
    """Extract trace context from inbound request headers."""
    return extract(headers)

# In a background worker (not using FastAPI auto-instrumentation):
async def handle_webhook(request_headers: dict, body: dict):
    parent_context = extract_trace_context(request_headers)
    
    with tracer.start_as_current_span(
        "handle_inbound_webhook",
        context=parent_context  # sets this as the parent
    ) as span:
        span.set_attribute("webhook.type", body.get("event"))
        await process_webhook(body)
```

---

## 7. The OTel Collector

### What Is the OTel Collector?

The OTel Collector is a standalone process that:
1. **Receives** telemetry from your services (via OTLP protocol)
2. **Processes** it (filter, sample, add attributes, batch)
3. **Exports** to one or more backends (Jaeger, Tempo, Datadog, X-Ray, etc.)

```
Your Services
    ↓ OTLP (gRPC or HTTP)
[OTel Collector]
    ├── → Jaeger (self-hosted, development)
    ├── → Grafana Tempo (production)
    └── → Datadog (if customer uses it)
```

### Why Use the Collector

**Without collector**: Each service SDK exports directly to the backend. Changing backends requires code changes everywhere.

**With collector**: Change the collector config to change backends — zero code changes in your services.

Additional benefits:
- **Buffering**: Collector buffers spans if the backend is temporarily unavailable
- **Transformation**: Rename attributes, filter sensitive data, add resource attributes
- **Fan-out**: Send to multiple backends simultaneously (e.g., Jaeger for debug + Datadog for production)
- **Sampling**: Do tail-based sampling at the collector (see Sampling section)

### Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Batch spans before exporting (more efficient)
  batch:
    timeout: 5s
    send_batch_size: 1024
  
  # Add resource attributes to all spans
  resource:
    attributes:
      - key: deployment.environment
        value: production
        action: upsert
  
  # Filter out health check spans (noise)
  filter:
    spans:
      exclude:
        match_type: strict
        attributes:
          - key: http.target
            value: /health
  
  # Memory limiter — prevent OOM
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s

exporters:
  # Jaeger for development/debugging
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  # Grafana Tempo for production
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  
  # Datadog (if needed)
  datadog:
    api:
      key: ${DATADOG_API_KEY}
  
  # Debug: log spans to stdout
  logging:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource, filter]
      exporters: [jaeger, otlp/tempo]
```

### Docker Compose Setup for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  order-service:
    build: .
    environment:
      - OTLP_ENDPOINT=otel-collector:4317
      - ENVIRONMENT=development
    depends_on:
      - otel-collector

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    command: ["--config=/etc/otel-collector-config.yaml"]
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "14250:14250"   # gRPC collector
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

---

## 8. Backend Options

### Jaeger (Development / Self-Hosted)

**Best for**: Local development, staging environments, teams that need self-hosted.

**Setup**: Single Docker container, in-memory storage, Jaeger UI at port 16686.

**UI Features**: Trace search, waterfall view, service dependency graph, compare traces.

**Limitations**: In-memory storage (lost on restart), no alerting, no metrics.

```python
# For Jaeger, use the OTLP gRPC exporter pointing to jaeger:4317
# Jaeger supports OTLP natively since v1.35
exporter = OTLPSpanExporter(endpoint="jaeger:4317", insecure=True)
```

### Grafana Tempo (Production, Open Source)

**Best for**: Teams already using Grafana + Prometheus. Cost-effective object storage backend.

**Storage**: S3, GCS, Azure Blob — very cheap at scale.

**Integration**: Deep Grafana integration — jump from a log line to its trace, from a metric spike to a trace.

**Limitations**: Query-heavy workloads require careful resource planning.

### AWS X-Ray (AWS-Native)

**Best for**: Teams running entirely on AWS who want native integration with CloudWatch, Lambda, ECS.

**Setup**: Use the AWS Distro for OpenTelemetry (ADOT) to send OTel traces to X-Ray.

```python
# Use the AWS OTel exporter
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Point to AWS Distro for OpenTelemetry (ADOT) collector
exporter = OTLPSpanExporter(endpoint="http://localhost:4317")  # ADOT sidecar
```

**X-Ray Concepts**: Service maps, trace groups, insights, CloudWatch integration.

### Datadog APM

**Best for**: Customers who already pay for Datadog (unified platform: metrics + logs + traces + profiling).

**Setup**: Use the Datadog exporter in OTel Collector, or use `ddtrace` library.

```python
# Via OTel Collector with Datadog exporter (preferred approach)
# In collector config:
exporters:
  datadog:
    api:
      key: ${DATADOG_API_KEY}
      site: datadoghq.com
```

**Datadog-specific value**: Flame graphs, service maps, correlate with logs and metrics, set up SLOs.

### Honeycomb

**Best for**: Teams that want best-in-class trace analysis, high cardinality queries.

**Differentiator**: Can query on any attribute at high cardinality (query by `customer_id` across millions of spans — other backends struggle with this).

**Pricing**: Expensive at high volume, but the query capability is genuinely superior.

---

## 9. Sampling Strategies

### Why Sampling?

At scale, tracing every single request is expensive:
- 1000 RPS × 10 spans/request × 1KB/span = 10MB/second of trace data
- At $0.10/GB that's $864/month just for traces

Sampling discards some traces to reduce cost while preserving signal.

### Head-Based Sampling (Simple)

The sampling decision is made at the root span, before the trace is complete.

```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased

# Sample 10% of all traces
sampler = ParentBased(root=TraceIdRatioBased(0.10))

# Always trace errors (combine with custom sampler)
class AlwaysSampleErrors:
    """Custom sampler: always sample, but mark non-error traces for 10% sampling."""
    pass
```

**Problem with head-based sampling**: You don't know if a trace will be interesting (contain an error) until it's complete. Head-based sampling might discard error traces.

### Tail-Based Sampling (Better)

The sampling decision is made after the trace is complete, based on its contents. Implemented in the OTel Collector:

```yaml
# otel-collector-config.yaml — tail-based sampling
processors:
  tail_sampling:
    decision_wait: 10s     # wait up to 10 seconds for trace to complete
    num_traces: 100000     # max traces held in memory
    
    policies:
      # Always keep error traces
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      
      # Always keep slow traces (>2 seconds)
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 2000}
      
      # Keep 5% of everything else (healthy fast traces)
      - name: random-sample
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
```

**Result**: Every error trace is captured. Every slow trace is captured. 5% of normal traces are captured. Dramatically lower cost, higher signal quality.

### Sampling Strategy by Environment

```python
# Development: trace everything
TRACE_SAMPLE_RATE = 1.0

# Staging: trace everything  
TRACE_SAMPLE_RATE = 1.0

# Production low-traffic service (<100 RPS): trace everything
TRACE_SAMPLE_RATE = 1.0

# Production medium-traffic (100-1000 RPS): 20% head-based + tail sampling for errors
TRACE_SAMPLE_RATE = 0.20  # with tail sampling in collector

# Production high-traffic (>1000 RPS): 5% head-based + tail sampling
TRACE_SAMPLE_RATE = 0.05  # with tail sampling in collector
```

---

## 10. Connecting Tracing to Logging

The most powerful debugging workflow is jumping between traces and logs. To enable this, **inject the current trace ID into every log record**.

```python
import structlog
from opentelemetry import trace

def inject_trace_context(logger, method, event_dict):
    """
    structlog processor that adds the current OTel trace/span IDs to every log entry.
    
    This allows you to:
    1. See a log entry and immediately click through to its trace
    2. See a slow trace and immediately find all log entries from that request
    """
    current_span = trace.get_current_span()
    
    if current_span and current_span.is_recording():
        ctx = current_span.get_span_context()
        if ctx.is_valid:
            # Format trace_id as 32-char hex (standard format)
            event_dict["trace_id"] = format(ctx.trace_id, '032x')
            event_dict["span_id"] = format(ctx.span_id, '016x')
            # trace_flags: 1 = sampled, 0 = not sampled
            event_dict["trace_sampled"] = bool(ctx.trace_flags & 0x01)
    
    return event_dict


# Add this processor to your structlog configuration
shared_processors = [
    structlog.contextvars.merge_contextvars,
    structlog.processors.TimeStamper(fmt="iso", utc=True),
    structlog.stdlib.add_log_level,
    inject_trace_context,  # ← add this
    add_service_context,
    structlog.processors.format_exc_info,
    structlog.processors.JSONRenderer(),
]
```

Now every log entry includes `trace_id` and `span_id`:

```json
{
  "timestamp": "2024-01-15T14:23:11.432Z",
  "level": "error",
  "service": "order-service",
  "event": "shipment_creation_failed",
  "order_id": "ORD-12345",
  "error": "Connection refused",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "a2fb4a1d1a96d312",
  "correlation_id": "req-7f3a9b2c"
}
```

**In Grafana**: If you're using Tempo + Loki, you can configure a "derived field" in Loki that turns `trace_id` values into clickable links that jump directly to the trace in Tempo. One click from a log error to the full trace waterfall.

**In Datadog**: Automatic — Datadog correlates log entries with traces when they share the same trace ID format.

---

## 11. Full Working Example: FastAPI with OTel

### Complete Project Setup

```
order-service/
├── tracing.py          # OTel configuration
├── logging_config.py   # structlog configuration  
├── middleware.py       # Request logging + correlation
├── main.py             # FastAPI app
├── services/
│   ├── order_service.py
│   └── carrier_service.py
└── docker-compose.yml  # Jaeger for local dev
```

### `tracing.py`

```python
# tracing.py
import os
import logging
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

def configure_tracing() -> trace.Tracer:
    service_name = os.environ.get("SERVICE_NAME", "order-service")
    environment = os.environ.get("ENVIRONMENT", "production")
    otlp_endpoint = os.environ.get("OTLP_ENDPOINT")
    
    resource = Resource.create({
        "service.name": service_name,
        "service.version": os.environ.get("SERVICE_VERSION", "unknown"),
        "deployment.environment": environment,
    })
    
    provider = TracerProvider(resource=resource)
    
    if otlp_endpoint:
        from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
        exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
        provider.add_span_processor(BatchSpanProcessor(exporter))
    
    if environment in ("development", "local") and not otlp_endpoint:
        provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
    
    trace.set_tracer_provider(provider)
    FastAPIInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()
    
    return trace.get_tracer(service_name)
```

### `services/order_service.py` — Full Manual Instrumentation

```python
# services/order_service.py
import time
import structlog
from opentelemetry import trace
from typing import List, Optional
from dataclasses import dataclass

log = structlog.get_logger()
tracer = trace.get_tracer(__name__)


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
    def __init__(self, carrier_service, tpl_service):
        self.carrier = carrier_service
        self.tpl = tpl_service

    async def process_order(self, order: Order) -> dict:
        """
        Process an order with full OTel tracing.
        
        The FastAPI auto-instrumentation creates the root span for the HTTP request.
        We create child spans for each business step.
        """
        # FastAPI auto-instrumentation already created the parent span.
        # Our child spans will automatically be children of it.
        
        with tracer.start_as_current_span("order.process") as span:
            span.set_attribute("order.id", order.order_id)
            span.set_attribute("order.customer_id", order.customer_id)
            span.set_attribute("order.item_count", len(order.items))
            span.set_attribute("order.total_value", 
                               sum(i.quantity * i.unit_price for i in order.items))
            
            # Step 1: Validate
            await self._validate(order, span)
            
            # Step 2: Get carrier rates
            best_rate = await self._get_carrier_rates(order, span)
            
            # Step 3: Create shipment
            shipment = await self._create_shipment(order, best_rate, span)
            
            span.set_status(trace.StatusCode.OK)
            return shipment

    async def _validate(self, order: Order, parent_span) -> None:
        with tracer.start_as_current_span("order.validate") as span:
            errors = []
            for item in order.items:
                if item.quantity <= 0:
                    errors.append(f"{item.sku}: invalid quantity {item.quantity}")
                if item.unit_price < 0:
                    errors.append(f"{item.sku}: negative price")
            
            span.set_attribute("validation.error_count", len(errors))
            
            if errors:
                error_msg = "; ".join(errors)
                span.set_attribute("validation.errors", error_msg)
                span.set_status(trace.StatusCode.ERROR, error_msg)
                log.warning("order_validation_failed", 
                           order_id=order.order_id, 
                           errors=errors)
                raise ValueError(f"Order validation failed: {error_msg}")
            
            span.set_status(trace.StatusCode.OK)
            log.info("order_validated", order_id=order.order_id)

    async def _get_carrier_rates(self, order: Order, parent_span) -> dict:
        with tracer.start_as_current_span("order.get_carrier_rates") as span:
            span.set_attribute("shipping.destination_country", 
                               order.shipping_address.get("country"))
            span.set_attribute("shipping.destination_zip", 
                               order.shipping_address.get("zip"))
            
            start = time.monotonic()
            
            try:
                rates = await self.carrier.get_rates(
                    items=order.items,
                    destination=order.shipping_address
                )
                duration_ms = (time.monotonic() - start) * 1000
                
                span.set_attribute("carrier.rate_count", len(rates))
                span.set_attribute("carrier.cheapest_rate_usd", rates[0]["price"])
                span.set_attribute("carrier.cheapest_carrier", rates[0]["carrier"])
                
                log.info("carrier_rates_retrieved", 
                         order_id=order.order_id,
                         rate_count=len(rates),
                         cheapest_rate=rates[0]["price"],
                         duration_ms=round(duration_ms, 2))
                
                return rates[0]
                
            except Exception as e:
                duration_ms = (time.monotonic() - start) * 1000
                span.record_exception(e)
                span.set_status(trace.StatusCode.ERROR, str(e))
                span.set_attribute("error.type", type(e).__name__)
                log.error("carrier_rates_failed", 
                          order_id=order.order_id,
                          error=str(e),
                          duration_ms=round(duration_ms, 2),
                          exc_info=True)
                raise

    async def _create_shipment(self, order: Order, rate: dict, parent_span) -> dict:
        with tracer.start_as_current_span("order.create_shipment") as span:
            span.set_attribute("shipment.carrier", rate["carrier"])
            span.set_attribute("shipment.rate_usd", rate["price"])
            span.set_attribute("shipment.service_level", rate.get("service_level"))
            
            try:
                shipment = await self.tpl.create_shipment(
                    order_id=order.order_id,
                    carrier=rate["carrier"],
                    items=order.items,
                    address=order.shipping_address,
                )
                
                span.set_attribute("shipment.id", shipment["id"])
                span.set_attribute("shipment.tracking_number", 
                                   shipment.get("tracking_number", ""))
                span.set_status(trace.StatusCode.OK)
                
                log.info("shipment_created",
                         order_id=order.order_id,
                         shipment_id=shipment["id"],
                         carrier=rate["carrier"])
                
                return shipment
                
            except Exception as e:
                span.record_exception(e)
                span.set_status(trace.StatusCode.ERROR, str(e))
                log.error("shipment_creation_failed",
                          order_id=order.order_id,
                          carrier=rate["carrier"],
                          error=str(e),
                          exc_info=True)
                raise
```

### `main.py` — Wiring It All Together

```python
# main.py
import os
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List
import structlog

# IMPORTANT: configure tracing BEFORE creating the FastAPI app
# because FastAPIInstrumentor().instrument() needs to be called first
from tracing import configure_tracing
from logging_config import configure_logging

configure_logging()
tracer = configure_tracing()

from middleware import CorrelationAndLoggingMiddleware
from services.order_service import OrderService, Order, OrderItem

log = structlog.get_logger()
app = FastAPI()
app.add_middleware(CorrelationAndLoggingMiddleware)

order_svc = OrderService(
    carrier_service=...,
    tpl_service=...,
)


class CreateOrderRequest(BaseModel):
    order_id: str
    customer_id: str
    items: List[dict]
    shipping_address: dict


@app.post("/api/v1/orders")
async def create_order(request: CreateOrderRequest):
    # Get the current span created by FastAPIInstrumentor
    # Add business context to it
    current_span = trace.get_current_span()
    current_span.set_attribute("customer.id", request.customer_id)
    current_span.set_attribute("order.id", request.order_id)
    
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
        log.error("order_endpoint_error", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

## 12. FDE Context: Debugging Slow Shipments

### The Customer Call

**Scenario**: Tuesday morning. Global Logistics Corp's CTO is on the call: "Our shipment creation is taking 10-12 seconds. It was fast last week — under 2 seconds. We've got a demo for a big client on Friday and this is embarrassing. Can you look into it?"

### Without Distributed Tracing (The Dark Age)

```
CTO: "It's slow."
You: "Let me check... [grepping logs for 30 minutes]"
You: "I see the total request duration is 10 seconds."
CTO: "Yes. Why?"
You: "I'm... not sure yet. Let me check the 3PL connector logs..."
[30 more minutes later]
You: "The carrier API seems slow?"
CTO: "Can you be more specific?"
You: "I need more time to trace this."
CTO: [hangs up]
```

### With Distributed Tracing

**Step 1**: Open Jaeger/Tempo UI, filter by `service=order-service` and `duration>5000ms`. See all slow traces from the last 24 hours.

**Step 2**: Click on a representative slow trace. You see:

```
POST /api/v1/orders                            [order-service]    11,234ms
  ├── order.process                            [order-service]    11,201ms
  │   ├── order.validate                       [order-service]       14ms
  │   ├── order.get_carrier_rates              [order-service]    9,987ms  ← !!!
  │   │   └── GET https://carrier.api/rates    [httpx]            9,972ms
  │   └── order.create_shipment                [order-service]     1,200ms
  │       └── POST https://tpl.example.com     [httpx]            1,185ms
```

**Step 3**: Click on the `GET https://carrier.api/rates` span. See the attributes:
```
http.url: https://carrier.api/v3/rates?origin=90210&destination=10001
http.status_code: 200
http.request_content_length: 1284
http.response_content_length: 48291  ← 48KB response? That's new.
```

**Step 4**: Check a trace from last week:
```
order.get_carrier_rates    [order-service]    412ms
└── GET https://carrier.api/rates  [httpx]   398ms
   http.response_content_length: 1247  ← 1.2KB last week vs 48KB this week
```

**Finding**: The carrier API response size grew from ~1.2KB to ~48KB. They probably added fields to their response. Your code is downloading 40x more data than before, which causes the 10-second delay.

**Conversation with the CTO**:
> "I found it. The carrier rate API response size grew from 1.2KB to 48KB between last Monday and last Wednesday. They likely added new fields to their response. Our integration is downloading 40x more data on each request. I can see this clearly in the trace — the carrier API call went from 400ms to 10 seconds. The fix is to request only the fields we need from the carrier API, or to cache carrier rates for short periods. I can have this fixed today."

**Total time to diagnosis**: 4 minutes. That's the distributed tracing FDE superpower.

---

## 13. Common Failure Modes

### Failure Mode 1: Spans Not Connected (Broken Trace Tree)

**Symptom**: Traces show two separate, disconnected traces instead of one connected trace. Or downstream service spans appear as root spans, not as children.

**Cause**: The `traceparent` header is not being propagated.

**Debug**:
```python
# In the upstream service, log the headers you're sending
log.debug("outbound_headers", headers=dict(httpx_request.headers))

# In the downstream service, log the headers you received
log.debug("inbound_headers", headers=dict(request.headers))

# Check if traceparent is present in both
```

**Fix**: Ensure `HTTPXClientInstrumentor().instrument()` is called before creating the httpx client. Or manually inject headers:

```python
from opentelemetry.propagate import inject
headers = {}
inject(headers)
response = await client.post(url, headers=headers)
```

### Failure Mode 2: All Spans Show as Root Spans

**Symptom**: No parent-child relationships in Jaeger. Every span is a root span.

**Cause**: `trace.set_tracer_provider(provider)` was not called before `FastAPIInstrumentor().instrument()`.

**Fix**: Configure OTel **before** creating the FastAPI app and before calling any instrumentors.

```python
# main.py — CORRECT ORDER
configure_tracing()  # FIRST — sets global tracer provider
app = FastAPI()      # SECOND — FastAPIInstrumentor sees the configured provider
```

### Failure Mode 3: Traces Not Appearing in Backend

**Symptom**: You instrument your service but no traces appear in Jaeger/Tempo.

**Causes**:
1. **OTLP endpoint wrong**: Check the endpoint URL and port (gRPC: 4317, HTTP: 4318)
2. **Batch processor not flushed**: BatchSpanProcessor queues spans — if the service shuts down before it flushes, spans are lost. Add a shutdown hook:

```python
import atexit
from opentelemetry.sdk.trace import TracerProvider

provider = TracerProvider(...)
atexit.register(provider.shutdown)
```

3. **Spans being dropped by sampler**: Log sampler decisions during debugging:

```python
from opentelemetry.sdk.trace.sampling import ALWAYS_ON
provider = TracerProvider(sampler=ALWAYS_ON)  # temporarily force all traces
```

### Failure Mode 4: Missing Business Context in Spans

**Symptom**: Traces show timing and HTTP info but no business context (no order_id, customer_id, etc.).

**Cause**: `set_attribute()` calls are missing from manual instrumentation.

**Fix**: Establish a convention: every span that represents a business operation must include the relevant business entity IDs.

```python
# In code review checklist:
# - Does this span include the business entity ID (order_id, shipment_id, etc.)?
# - Does this span set status (OK or ERROR)?
# - Are exceptions recorded with span.record_exception()?
```

### Failure Mode 5: OTel Impacting Application Performance

**Symptom**: After adding OTel instrumentation, service latency increases by 50-100ms.

**Cause**: The `SimpleSpanProcessor` is synchronous — it blocks the request thread while exporting spans.

**Fix**: Always use `BatchSpanProcessor` in production:

```python
# WRONG — blocks every request
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
provider.add_span_processor(SimpleSpanProcessor(exporter))

# RIGHT — exports in background batch
from opentelemetry.sdk.trace.export import BatchSpanProcessor
provider.add_span_processor(BatchSpanProcessor(exporter))
```

---

## 14. Interview Angle

### What Interviewers Ask

**Q: "Explain the difference between a trace, a span, and a log."**

**Strong answer**: "A log is a discrete event record — it tells you something happened at a specific time with some context. A span is a unit of work within a distributed request — it has a start time, end time, and attributes, and it knows its relationship to other spans (parent/child). A trace is the complete collection of spans for a single request as it travels through multiple services. The relationship is: one trace contains many spans (one per operation), and each operation may emit many log entries. The trace gives you the call graph and timing; the logs give you the detail. They're connected by injecting the trace ID into log records."

---

**Q: "How would you explain trace propagation to a customer who's integrating a new system?"**

**Strong answer**: "When your system sends an HTTP request to our service, we include a special header called `traceparent`. That header carries the trace ID — a unique identifier for this request — and the ID of the span that made the call. When your service receives the request, if you're also instrumented with OpenTelemetry, you can extract that header and continue the trace. Your spans become children of our span in the same trace. This means when we look at the trace in our monitoring system, we see the complete call graph — your system calling ours, and everything our system did in response. It helps us debug issues much faster because we can see exactly where time was spent across both systems."

---

**Q: "When would you use tail-based sampling vs head-based sampling?"**

**Strong answer**: "Head-based sampling makes the 'keep or drop' decision at the start of a trace, before you know if it will be interesting. This is simple but means you might discard error traces because you don't know yet they'll be errors. Tail-based sampling waits until the trace is complete and then makes the decision based on what actually happened — keep all error traces, keep all traces over 2 seconds, sample 5% of everything else. The tradeoff is that tail-based sampling requires the OTel Collector to buffer all in-flight traces, which uses memory. For most services, I use head-based sampling at 10-20% for cost control, plus tail-based sampling in the Collector to ensure errors are always captured. The result is: you capture all the interesting traces (errors, slow) and a representative sample of healthy traces."

---

## 15. Practice Exercise

### Exercise: Instrument a Multi-Service Order Pipeline

**Goal**: Set up a two-service system with distributed tracing and observe the complete call graph in Jaeger.

**Services to build**:

**Service A** (`order-service`, port 8001):
- `POST /orders` endpoint
- Creates an order, then calls Service B to check inventory
- If inventory is available, calls Service B again to reserve it
- Manual spans for: `order.validate`, `order.check_inventory`, `order.reserve_inventory`
- Add business attributes: `order.id`, `customer.id`, `order.total_value`

**Service B** (`inventory-service`, port 8002):
- `GET /inventory/{sku}` — returns `{"sku": "...", "available": 50}`
- `POST /inventory/reserve` — reserves inventory, occasionally returns 409 Conflict
- Uses FastAPI auto-instrumentation (no manual spans needed for this exercise)

**Requirements**:

1. Both services use OTel with `OTLP_ENDPOINT=localhost:4317`
2. Start a Jaeger container: `docker run -p 16686:16686 -p 4317:4317 jaegertracing/all-in-one`
3. Send a test order and observe in Jaeger UI (http://localhost:16686):
   - One trace spans both services
   - Service A spans are parents of Service B spans
   - Business attributes visible on `order.process` span

4. **Simulate a slow inventory check**:
   - Add `await asyncio.sleep(2)` to inventory service
   - Observe in Jaeger: the `order.check_inventory` span takes 2 seconds
   - The waterfall clearly shows which call is the bottleneck

5. **Simulate an error**:
   - Make inventory service return 503 randomly (10% of requests)
   - Observe: error traces captured with `StatusCode.ERROR`
   - `span.record_exception()` shows the full exception in Jaeger

6. **Connect to logs**:
   - Add `inject_trace_context` to structlog processors
   - Observe: log entries include `trace_id`
   - In terminal, find a log entry with an error, copy its `trace_id`, search for it in Jaeger

**Stretch goal**: Add the OTel Collector between your services and Jaeger. Configure tail-based sampling to always capture error traces and sample 20% of normal traces. Observe that a forced error always appears in Jaeger even at low traffic.

---

**Next**: [03-metrics-prometheus-grafana.md](./03-metrics-prometheus-grafana.md) — Metrics give you the aggregate view that complements per-request tracing.
