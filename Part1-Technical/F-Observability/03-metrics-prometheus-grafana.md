# Metrics, Prometheus, and Grafana for FDEs

> **FDE Principle**: Logs tell you what happened to one request. Metrics tell you what's happening to all requests. A Grafana dashboard where a customer's CTO can see "orders processed per minute" and "error rate by customer" in real time is a trust-building artifact that pays dividends for months.

**Related files:**
- [01-structured-logging.md](./01-structured-logging.md) — Logs for request-level detail
- [02-distributed-tracing-otel.md](./02-distributed-tracing-otel.md) — Traces for request-path analysis
- [04-log-aggregation.md](./04-log-aggregation.md) — CloudWatch as an alternative metrics source

---

## Table of Contents

1. [Why Metrics](#1-why-metrics)
2. [Metric Types](#2-metric-types)
3. [The RED Method](#3-the-red-method)
4. [The USE Method](#4-the-use-method)
5. [Prometheus Basics](#5-prometheus-basics)
6. [prometheus-client in Python](#6-prometheus-client-in-python)
7. [Exposing Metrics from FastAPI](#7-exposing-metrics-from-fastapi)
8. [PromQL: Querying Metrics](#8-promql-querying-metrics)
9. [Prometheus Alerting Rules](#9-prometheus-alerting-rules)
10. [Grafana Dashboards](#10-grafana-dashboards)
11. [Custom Business Metrics](#11-custom-business-metrics)
12. [CloudWatch Metrics (AWS-Native)](#12-cloudwatch-metrics-aws-native)
13. [Full Working Example](#13-full-working-example)
14. [FDE Context: The CTO Dashboard](#14-fde-context-the-cto-dashboard)
15. [Common Failure Modes](#15-common-failure-modes)
16. [Interview Angle](#16-interview-angle)
17. [Practice Exercise](#17-practice-exercise)

---

## 1. Why Metrics

### The Three Pillars of Observability

- **Logs** answer: "What exactly happened to request #X?" (high detail, low aggregation)
- **Traces** answer: "Why was request #X slow?" (causal chain, timing)
- **Metrics** answer: "How is the system doing right now and over time?" (aggregate, trends)

Metrics are the foundation of alerting. You don't alert on individual log entries (too noisy) — you alert when a metric crosses a threshold (e.g., "error rate > 5% for 5 minutes").

### What Metrics Enable

| Use Case | How Metrics Help |
|----------|-----------------|
| Capacity planning | "We're handling 200 orders/hour today, up 40% from last month — time to scale" |
| SLA compliance | "Our p99 latency is 1.8s vs our 2s SLA — we have headroom" |
| Incident detection | "Error rate spiked at 3pm, correlates with a deploy" |
| Business reporting | "We processed 14,000 EDI files yesterday with a 99.7% success rate" |
| Cost attribution | "Customer ACME generates 60% of our API calls" |
| Customer QBRs | Show a customer their own integration metrics over the quarter |

### Metrics vs Logs: The Cost Comparison

At 1000 requests/second:
- **Logs**: 1 log entry per request × 500 bytes = 500 bytes/sec × 86,400 seconds = ~43GB/day
- **Metrics**: Prometheus scrapes every 15 seconds, aggregating to ~50 time series = ~50KB/day

Metrics are **100,000x cheaper** than logs at this scale. You can afford to keep 1 year of metrics; you might only keep 30 days of logs.

---

## 2. Metric Types

### Counter

A counter **only goes up** (or resets on service restart). Use for things you count cumulatively.

```python
# What counters are used for:
requests_total          # total HTTP requests received
errors_total            # total errors
orders_processed_total  # total orders processed
edi_files_ingested_total  # total EDI files ingested
messages_consumed_total   # total SQS messages consumed
```

**Key insight**: Counters are not useful as raw values — you query their **rate of change** per second.
- `requests_total` → boring (always going up)
- `rate(requests_total[5m])` → interesting (current request rate)

### Gauge

A gauge can go up or down. Use for "current value of something."

```python
# What gauges are used for:
active_connections          # number of current DB connections
queue_depth                 # messages currently in SQS queue
memory_usage_bytes          # current memory usage
active_worker_threads       # workers currently processing
pending_orders              # orders waiting for processing
```

### Histogram

A histogram samples observations and counts them in configurable **buckets**. Use for durations, sizes, or any value where you care about distribution.

```python
# What histograms are used for:
request_duration_seconds    # HTTP request latency
db_query_duration_seconds   # Database query latency  
payload_size_bytes          # Request/response sizes
shipment_processing_seconds # Time to process a shipment
```

A histogram with buckets `[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]` tracks how many observations fell into each bucket. From this you can calculate:
- p50 (median): 50% of requests are faster than X
- p95: 95% of requests are faster than X
- p99: 99% of requests are faster than X

**Why percentiles over averages**: An average of 500ms might hide that 1% of requests take 10 seconds. p99 captures those outliers. For customer SLAs, you care about p99, not average.

### Summary

Similar to histogram, but computes percentiles **client-side** (in the application process). Disadvantage: cannot be aggregated across multiple instances. Generally prefer histograms unless you have very specific requirements.

---

## 3. The RED Method

Every service you build should have RED metrics. This is the baseline for service health.

### R — Rate

**What it is**: Requests per second (RPS)

**Why it matters**: Baseline for "is the service receiving traffic?" and for normalizing errors.

**PromQL**:
```promql
# Request rate over last 5 minutes (requests/second)
rate(http_requests_total{service="order-service"}[5m])

# Broken down by endpoint
rate(http_requests_total{service="order-service"}[5m]) by (path)
```

### E — Errors

**What it is**: Error rate (fraction of requests that result in errors)

**Why it matters**: The primary signal for "is something broken?"

**PromQL**:
```promql
# Error rate as a fraction (0-1)
rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
/
rate(http_requests_total{service="order-service"}[5m])

# Error rate as a percentage
(
  rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
  /
  rate(http_requests_total{service="order-service"}[5m])
) * 100
```

### D — Duration

**What it is**: Request latency distribution (p50, p95, p99)

**Why it matters**: Your customers experience p99, not average. "But average latency is fine" is not a valid excuse when 1% of shipment creation requests take 30 seconds.

**PromQL**:
```promql
# p50 latency
histogram_quantile(0.50, 
  rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
)

# p95 latency  
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
)

# p99 latency
histogram_quantile(0.99, 
  rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
)
```

### Applying RED

Every service should have a Grafana dashboard with:
1. **Rate panel**: Time series of requests/second
2. **Error rate panel**: Time series of error rate %, alert when > 5%
3. **Duration panel**: p50, p95, p99 time series on one graph

---

## 4. The USE Method

USE applies to **resources** (CPU, memory, disk, network, DB connections). Different from RED (which is service-level).

### U — Utilization

How much of the available capacity is being used?

```promql
# CPU utilization
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory utilization  
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# DB connection pool utilization
db_connection_pool_used / db_connection_pool_max
```

**Rule of thumb**: Alert at 80% sustained utilization (not spikes).

### S — Saturation

How much work is waiting/queued?

```promql
# SQS queue depth (how many messages waiting to be processed)
aws_sqs_approximate_number_of_messages_visible

# DB connection waiters (connections waiting for a free slot)
db_connection_pool_waiting_count
```

**Why saturation matters more than utilization**: A system at 99% CPU but with an empty queue is fine. A system at 60% CPU with a growing queue is on its way to an incident.

### E — Errors

Resource-level errors (distinct from service-level errors):

```promql
# Disk I/O errors
rate(node_disk_io_time_weighted_seconds_total[5m])

# Network errors
rate(node_network_receive_errs_total[5m])
```

---

## 5. Prometheus Basics

### How Prometheus Works

Prometheus is a **pull-based** monitoring system. Instead of your services pushing metrics to Prometheus, **Prometheus scrapes** a `/metrics` HTTP endpoint on each service at regular intervals (default: every 15 seconds).

```
Prometheus Server
    ↓ scrapes every 15s
Service A :8001/metrics
Service B :8002/metrics
Service C :8003/metrics
```

### The /metrics Endpoint Format

Prometheus uses a plain-text format called **Exposition Format**:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="POST",path="/api/v1/orders",status_code="200"} 1234
http_requests_total{method="POST",path="/api/v1/orders",status_code="500"} 12
http_requests_total{method="GET",path="/api/v1/orders",status_code="200"} 5678

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 892
http_request_duration_seconds_bucket{le="0.25"} 1143
http_request_duration_seconds_bucket{le="0.5"} 1234
http_request_duration_seconds_bucket{le="1.0"} 1240
http_request_duration_seconds_bucket{le="+Inf"} 1246
http_request_duration_seconds_sum 187.3
http_request_duration_seconds_count 1246
```

### Labels: The Core Concept

Labels are key-value pairs that add dimensions to metrics. They allow you to slice and dice metrics:

```
http_requests_total{method="POST", path="/orders", status_code="200", service="order-service"}
http_requests_total{method="POST", path="/orders", status_code="500", service="order-service"}
http_requests_total{method="POST", path="/orders", status_code="200", service="order-service", customer="acme"}
```

**Cardinal sin**: High-cardinality labels. Never use `user_id` or `order_id` as Prometheus labels — each unique value creates a new time series, and Prometheus doesn't scale to millions of time series. Use `customer_id` (you might have 100 customers) but never `order_id` (you might have millions of orders).

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s       # How often to scrape
  evaluation_interval: 15s   # How often to evaluate alert rules

scrape_configs:
  # Scrape the order service
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8001']
    metrics_path: '/metrics'
  
  # Scrape multiple instances (for load-balanced services)
  - job_name: 'order-service-cluster'
    static_configs:
      - targets: 
          - 'order-service-1:8001'
          - 'order-service-2:8001'
          - 'order-service-3:8001'
  
  # For containerized environments, use service discovery
  - job_name: 'ecs-services'
    ec2_sd_configs:
      - region: us-east-1
        filters:
          - name: tag:prometheus-scrape
            values: ["true"]
```

---

## 6. prometheus-client in Python

### Installation

```bash
pip install prometheus-client
```

### Defining Metrics

Metrics are defined at the **module level** — not inside functions or classes. This is important: Prometheus metrics are global singletons.

```python
# metrics.py — define all metrics for the service here
from prometheus_client import Counter, Gauge, Histogram, Summary

# ---- HTTP Metrics (for every service) ----

HTTP_REQUESTS_TOTAL = Counter(
    name='http_requests_total',
    documentation='Total number of HTTP requests received',
    labelnames=['method', 'path', 'status_code'],
)

HTTP_REQUEST_DURATION_SECONDS = Histogram(
    name='http_request_duration_seconds',
    documentation='HTTP request duration in seconds',
    labelnames=['method', 'path', 'status_code'],
    # Buckets optimized for API latency (in seconds)
    # 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)


# ---- Business Metrics ----

ORDERS_PROCESSED_TOTAL = Counter(
    name='orders_processed_total',
    documentation='Total orders processed',
    labelnames=['customer_id', 'status'],  # status: success, failed, validation_error
)

SHIPMENTS_CREATED_TOTAL = Counter(
    name='shipments_created_total',
    documentation='Total shipments created',
    labelnames=['carrier', 'service_level', 'customer_id'],
)

SHIPMENT_CREATION_DURATION_SECONDS = Histogram(
    name='shipment_creation_duration_seconds',
    documentation='Time to create a shipment including all downstream calls',
    labelnames=['carrier', 'customer_id'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0],
)

EDI_FILES_PROCESSED_TOTAL = Counter(
    name='edi_files_processed_total',
    documentation='Total EDI files processed',
    labelnames=['transaction_set', 'customer_id', 'status'],
)

MAPPING_ERRORS_TOTAL = Counter(
    name='mapping_errors_total',
    documentation='Data mapping errors by type and customer',
    labelnames=['error_type', 'customer_id', 'field_name'],
)


# ---- Queue Metrics ----

QUEUE_DEPTH = Gauge(
    name='order_queue_depth',
    documentation='Current number of orders in the processing queue',
    labelnames=['queue_name'],
)

QUEUE_PROCESSING_LATENCY_SECONDS = Histogram(
    name='queue_message_processing_duration_seconds',
    documentation='Time from message enqueue to processing completion',
    labelnames=['queue_name', 'message_type'],
    buckets=[1.0, 5.0, 10.0, 30.0, 60.0, 300.0, 600.0],
)


# ---- Downstream Service Metrics ----

UPSTREAM_REQUESTS_TOTAL = Counter(
    name='upstream_requests_total',
    documentation='Requests made to upstream (downstream from our perspective) services',
    labelnames=['service', 'endpoint', 'status_code'],
)

UPSTREAM_REQUEST_DURATION_SECONDS = Histogram(
    name='upstream_request_duration_seconds',
    documentation='Latency of calls to upstream services',
    labelnames=['service', 'endpoint'],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0],
)
```

### Using Metrics in Application Code

```python
import time
from metrics import (
    HTTP_REQUESTS_TOTAL,
    HTTP_REQUEST_DURATION_SECONDS,
    ORDERS_PROCESSED_TOTAL,
    SHIPMENT_CREATION_DURATION_SECONDS,
    MAPPING_ERRORS_TOTAL,
)

# ---- Increment a Counter ----
ORDERS_PROCESSED_TOTAL.labels(
    customer_id="acme-corp",
    status="success"
).inc()

# Error case
ORDERS_PROCESSED_TOTAL.labels(
    customer_id="acme-corp",
    status="failed"
).inc()

# ---- Time a block of code with Histogram ----

# Method 1: context manager
with SHIPMENT_CREATION_DURATION_SECONDS.labels(
    carrier="fedex",
    customer_id="acme-corp"
).time():
    shipment = await create_fedex_shipment(order)

# Method 2: manual timing (for async with error handling)
start = time.monotonic()
try:
    shipment = await create_fedex_shipment(order)
    status = "success"
except Exception as e:
    status = "error"
    raise
finally:
    duration = time.monotonic() - start
    SHIPMENT_CREATION_DURATION_SECONDS.labels(
        carrier="fedex",
        customer_id=customer_id
    ).observe(duration)

# ---- Set a Gauge ----
async def update_queue_depth_metric(queue_url: str):
    """Called periodically to update the queue depth gauge."""
    depth = await sqs.get_queue_attributes(
        QueueUrl=queue_url,
        AttributeNames=["ApproximateNumberOfMessages"]
    )
    QUEUE_DEPTH.labels(queue_name="order-processing").set(
        int(depth["Attributes"]["ApproximateNumberOfMessages"])
    )
```

---

## 7. Exposing Metrics from FastAPI

### The /metrics Endpoint

```python
# main.py
from fastapi import FastAPI, Response
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST, CollectorRegistry, REGISTRY

app = FastAPI()

@app.get("/metrics")
async def metrics():
    """
    Prometheus metrics endpoint.
    
    This endpoint is scraped by Prometheus every 15 seconds.
    It should NOT require authentication (it's on an internal port or network).
    It should NOT be exposed to the public internet.
    """
    return Response(
        content=generate_latest(REGISTRY),
        media_type=CONTENT_TYPE_LATEST,
    )
```

### Metrics Middleware

```python
# middleware.py
import time
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from metrics import HTTP_REQUESTS_TOTAL, HTTP_REQUEST_DURATION_SECONDS

# Paths to exclude from metrics (health checks create noise, not signal)
EXCLUDED_PATHS = {"/health", "/healthz", "/ready", "/metrics", "/favicon.ico"}

# Normalize paths to prevent high cardinality
# e.g., /api/v1/orders/12345 → /api/v1/orders/{order_id}
def normalize_path(path: str) -> str:
    """
    Normalize URL paths to prevent high-cardinality metrics.
    
    /api/v1/orders/12345         → /api/v1/orders/{id}
    /api/v1/orders/12345/items   → /api/v1/orders/{id}/items
    /api/v1/customers/acme-corp  → /api/v1/customers/{id}
    """
    import re
    # Replace numeric IDs
    path = re.sub(r'/\d+', '/{id}', path)
    # Replace UUID-like strings
    path = re.sub(
        r'/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}',
        '/{uuid}',
        path
    )
    # Replace alphanumeric IDs at path endpoints (common pattern: /resources/ORD-12345)
    path = re.sub(r'/[A-Z]+-\d+$', '/{id}', path)
    return path


class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.url.path in EXCLUDED_PATHS:
            return await call_next(request)
        
        path = normalize_path(request.url.path)
        method = request.method
        
        start = time.monotonic()
        response = None
        
        try:
            response = await call_next(request)
            return response
        except Exception:
            raise
        finally:
            duration = time.monotonic() - start
            status_code = str(response.status_code) if response else "500"
            
            HTTP_REQUESTS_TOTAL.labels(
                method=method,
                path=path,
                status_code=status_code,
            ).inc()
            
            HTTP_REQUEST_DURATION_SECONDS.labels(
                method=method,
                path=path,
                status_code=status_code,
            ).observe(duration)
```

---

## 8. PromQL: Querying Metrics

PromQL (Prometheus Query Language) is the query language you use in Grafana panels and Prometheus alert rules.

### Instant Queries vs Range Queries

**Instant**: the current value of a metric at a specific point in time.
**Range**: values over a time window — used for rates and aggregations.

### Essential PromQL Patterns

```promql
# ---- COUNTERS ----

# Current rate of requests/second (over last 5 minutes)
rate(http_requests_total{service="order-service"}[5m])

# Total requests in last hour
increase(http_requests_total{service="order-service"}[1h])

# Request rate, broken down by path
rate(http_requests_total{service="order-service"}[5m]) by (path)

# Error rate as a percentage
(
  rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
  /
  rate(http_requests_total{service="order-service"}[5m])
) * 100

# ---- HISTOGRAMS ----

# p99 request latency in milliseconds
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
) * 1000

# p99 latency by path
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
) by (path) * 1000

# p50, p95, p99 on one query (for Grafana)
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m])) * 1000
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) * 1000
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) * 1000

# ---- GAUGES ----

# Current queue depth
order_queue_depth{queue_name="order-processing"}

# Queue depth by queue
order_queue_depth

# ---- BUSINESS METRICS ----

# Orders processed per minute by customer (use increase for counters over a window)
increase(orders_processed_total{status="success"}[1m]) by (customer_id)

# Order failure rate by customer
(
  rate(orders_processed_total{status="failed"}[5m]) by (customer_id)
  /
  rate(orders_processed_total[5m]) by (customer_id)
) * 100

# Mapping errors per minute by customer
rate(mapping_errors_total[5m]) by (customer_id, error_type)

# ---- AGGREGATIONS ----

# Sum across all instances (when horizontally scaled)
sum(rate(http_requests_total{service="order-service"}[5m]))

# Average across instances
avg(rate(http_requests_total{service="order-service"}[5m])) by (path)

# Top 5 slowest paths by p99
topk(5,
  histogram_quantile(0.99,
    sum(rate(http_request_duration_seconds_bucket[5m])) by (path, le)
  )
)
```

### Label Selectors

```promql
# Exact match
http_requests_total{service="order-service"}

# Multiple exact matches (OR)
http_requests_total{service=~"order-service|shipment-service"}

# Regex match
http_requests_total{status_code=~"5.."}  # any 5xx

# Not equal
http_requests_total{status_code!="200"}

# Not regex match
http_requests_total{path!~"/health.*"}
```

---

## 9. Prometheus Alerting Rules

### Alerting Rule File

```yaml
# alerts/service-alerts.yaml
groups:
  - name: order_service_alerts
    rules:
    
      # ---- RED Alerts ----
      
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
            /
            rate(http_requests_total{service="order-service"}[5m])
          ) * 100 > 5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High error rate on order-service"
          description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"
      
      - alert: VeryHighErrorRate
        expr: |
          (
            rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
            /
            rate(http_requests_total{service="order-service"}[5m])
          ) * 100 > 25
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "CRITICAL: Very high error rate on order-service"
      
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
          ) > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency: {{ $value | humanizeDuration }}"
      
      - alert: ServiceDown
        expr: up{job="order-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "order-service is down"
      
      # ---- Business Metric Alerts ----
      
      - alert: OrderProcessingStalled
        expr: rate(orders_processed_total[5m]) == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "No orders processed in 10 minutes"
          description: "Order processing may be stalled. Check queue depth and worker health."
      
      - alert: HighMappingErrorRate
        expr: |
          (
            rate(mapping_errors_total[5m])
            /
            rate(orders_processed_total[5m])
          ) * 100 > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High data mapping error rate: {{ $value }}%"
          description: "Customer {{ $labels.customer_id }} has a high mapping error rate. Check their data format."
      
      - alert: QueueDepthHigh
        expr: order_queue_depth{queue_name="order-processing"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Order processing queue depth: {{ $value }} messages"
          description: "Queue is backing up. Consider scaling up workers."
      
      - alert: QueueDepthCritical
        expr: order_queue_depth{queue_name="order-processing"} > 5000
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Order queue at {{ $value }} messages — severe backlog"
```

---

## 10. Grafana Dashboards

### Dashboard Architecture

A good Grafana dashboard for an FDE audience has layers:
1. **Executive overview** (top row): Health at a glance — green/red status
2. **RED metrics** (second row): Rate, Errors, Duration time series
3. **Business metrics** (third row): Orders/min, mapping errors, customer-specific
4. **Resource metrics** (bottom): CPU, memory, queue depth (for engineers)

### Building a Grafana Dashboard (JSON Model)

```json
{
  "title": "Order Service Dashboard",
  "uid": "order-service-v1",
  "tags": ["order-service", "production"],
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  
  "panels": [
    {
      "title": "Request Rate (req/s)",
      "type": "timeseries",
      "gridPos": {"x": 0, "y": 0, "w": 8, "h": 8},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service=\"order-service\"}[5m]))",
          "legendFormat": "Total req/s"
        },
        {
          "expr": "sum(rate(http_requests_total{service=\"order-service\", status_code=~\"2..\"}[5m]))",
          "legendFormat": "Success req/s"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "color": {"mode": "palette-classic"}
        }
      }
    },
    
    {
      "title": "Error Rate (%)",
      "type": "timeseries",
      "gridPos": {"x": 8, "y": 0, "w": 8, "h": 8},
      "targets": [
        {
          "expr": "(\n  rate(http_requests_total{service=\"order-service\", status_code=~\"5..\"}[5m])\n  /\n  rate(http_requests_total{service=\"order-service\"}[5m])\n) * 100",
          "legendFormat": "Error %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 1},
              {"color": "red", "value": 5}
            ]
          }
        }
      }
    },
    
    {
      "title": "Latency Percentiles (ms)",
      "type": "timeseries",
      "gridPos": {"x": 16, "y": 0, "w": 8, "h": 8},
      "targets": [
        {
          "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])) * 1000",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])) * 1000",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m])) * 1000",
          "legendFormat": "p99"
        }
      ],
      "fieldConfig": {"defaults": {"unit": "ms"}}
    },
    
    {
      "title": "Orders Processed Per Minute by Customer",
      "type": "timeseries",
      "gridPos": {"x": 0, "y": 8, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "rate(orders_processed_total{status=\"success\"}[1m]) * 60 by (customer_id)",
          "legendFormat": "{{ customer_id }}"
        }
      ],
      "fieldConfig": {"defaults": {"unit": "orders/min"}}
    },
    
    {
      "title": "Mapping Errors per Hour by Customer",
      "type": "timeseries",
      "gridPos": {"x": 12, "y": 8, "w": 12, "h": 8},
      "targets": [
        {
          "expr": "rate(mapping_errors_total[5m]) * 3600 by (customer_id, error_type)",
          "legendFormat": "{{ customer_id }} - {{ error_type }}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "errors/hr",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 10},
              {"color": "red", "value": 100}
            ]
          }
        }
      }
    }
  ]
}
```

### Grafana Alerting to Slack

```yaml
# grafana-alerting/notification-policies.yaml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: slack-ops
    receivers:
      - uid: slack-ops
        type: slack
        settings:
          url: "${SLACK_WEBHOOK_URL}"
          channel: "#ops-alerts"
          title: "{{ .CommonLabels.alertname }}"
          text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
          
  - orgId: 1
    name: pagerduty-critical
    receivers:
      - uid: pagerduty-critical
        type: pagerduty
        settings:
          integrationKey: "${PAGERDUTY_KEY}"

policies:
  - orgId: 1
    receiver: slack-ops
    routes:
      - receiver: pagerduty-critical
        matchers:
          - "severity = critical"
        continue: true  # also send to slack
```

---

## 11. Custom Business Metrics

This is where FDE work differs from pure infrastructure engineering. The most valuable metrics are **business metrics**, not just infrastructure metrics.

### The "Business Metric" Mindset

Every integration has key business events. Instrument them:

```python
# ---- EDI Integration Metrics ----

EDI_FILES_RECEIVED = Counter(
    'edi_files_received_total',
    'EDI files received from trading partners',
    ['customer_id', 'transaction_set', 'isa_sender_id']
)

EDI_FILES_PARSED = Counter(
    'edi_files_parsed_total', 
    'EDI files successfully parsed',
    ['customer_id', 'transaction_set', 'status']  # status: success, parse_error, schema_error
)

EDI_SEGMENTS_PROCESSED = Counter(
    'edi_segments_processed_total',
    'Individual EDI segments processed',
    ['transaction_set', 'segment_id']
)

EDI_PROCESSING_DURATION = Histogram(
    'edi_file_processing_duration_seconds',
    'Time to process an EDI file end-to-end',
    ['customer_id', 'transaction_set'],
    buckets=[0.1, 0.5, 1.0, 5.0, 10.0, 30.0, 60.0, 300.0]
)


# ---- Shipment Tracking Metrics ----

TRACKING_EVENTS_RECEIVED = Counter(
    'tracking_events_received_total',
    'Tracking events received from carriers',
    ['carrier', 'event_type']
)

TRACKING_UPDATES_DELIVERED = Counter(
    'tracking_updates_delivered_total',
    'Tracking updates delivered to customers',
    ['customer_id', 'channel', 'status']  # channel: webhook, email, api_poll
)

SHIPMENT_STATUS_LAG_SECONDS = Histogram(
    'shipment_status_lag_seconds',
    'Time between carrier event and customer notification',
    ['carrier', 'customer_id'],
    buckets=[5, 30, 60, 300, 600, 1800, 3600]  # 5s to 1hr
)


# ---- WMS Integration Metrics ----

INVENTORY_SYNC_RECORDS = Counter(
    'inventory_sync_records_total',
    'Inventory records synced from WMS',
    ['wms_id', 'sync_type', 'status']  # sync_type: full, incremental
)

INVENTORY_SYNC_LAG_SECONDS = Gauge(
    'inventory_sync_lag_seconds',
    'Seconds since last successful inventory sync',
    ['wms_id']
)

ORDER_ALLOCATION_DURATION = Histogram(
    'order_allocation_duration_seconds',
    'Time to allocate inventory to an order in WMS',
    ['wms_id', 'warehouse_id'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0]
)
```

### Using Business Metrics in Code

```python
import time
from metrics import (
    EDI_FILES_RECEIVED, EDI_FILES_PARSED, EDI_PROCESSING_DURATION,
    MAPPING_ERRORS_TOTAL, INVENTORY_SYNC_LAG_SECONDS
)

async def process_edi_file(
    file_content: str, 
    customer_id: str, 
    transaction_set: str
) -> dict:
    """Process an EDI file with full business metrics."""
    
    EDI_FILES_RECEIVED.labels(
        customer_id=customer_id,
        transaction_set=transaction_set,
        isa_sender_id="UNKNOWN"  # extracted below
    ).inc()
    
    start = time.monotonic()
    
    try:
        # Parse
        parsed = edi_parser.parse(file_content)
        isa_sender = parsed.get("ISA", {}).get("sender_id", "UNKNOWN")
        
        # Map to our domain model
        try:
            mapped = mapper.map_to_order(parsed, customer_id=customer_id)
        except MappingError as e:
            MAPPING_ERRORS_TOTAL.labels(
                error_type=e.error_type,
                customer_id=customer_id,
                field_name=e.field_name,
            ).inc()
            EDI_FILES_PARSED.labels(
                customer_id=customer_id,
                transaction_set=transaction_set,
                status="mapping_error"
            ).inc()
            raise
        
        # Process
        result = await order_service.create_order(mapped)
        
        EDI_FILES_PARSED.labels(
            customer_id=customer_id,
            transaction_set=transaction_set,
            status="success"
        ).inc()
        
        return result
        
    except Exception as e:
        if not isinstance(e, MappingError):
            EDI_FILES_PARSED.labels(
                customer_id=customer_id,
                transaction_set=transaction_set,
                status="error"
            ).inc()
        raise
    finally:
        duration = time.monotonic() - start
        EDI_PROCESSING_DURATION.labels(
            customer_id=customer_id,
            transaction_set=transaction_set
        ).observe(duration)


async def update_inventory_sync_lag(wms_id: str, last_sync_time: datetime):
    """Update the staleness gauge for inventory sync."""
    lag_seconds = (datetime.utcnow() - last_sync_time).total_seconds()
    INVENTORY_SYNC_LAG_SECONDS.labels(wms_id=wms_id).set(lag_seconds)
```

---

## 12. CloudWatch Metrics (AWS-Native)

### When to Use CloudWatch vs Prometheus

| Scenario | Recommendation |
|----------|---------------|
| Running on AWS ECS/Lambda/RDS | Use CloudWatch — it's already there |
| Custom application metrics | CloudWatch custom metrics or Prometheus |
| Want Grafana dashboards | Either: Grafana has CloudWatch data source |
| Cost-sensitive at high metric volume | Prometheus (CloudWatch charges per metric/alarm) |
| Teams not managing infrastructure | CloudWatch (zero ops burden) |
| Existing Datadog/Grafana stack | Keep using it |

### CloudWatch Custom Metrics from Python

```python
import boto3
from datetime import datetime
import time

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

def put_metric(
    metric_name: str, 
    value: float, 
    unit: str,
    dimensions: list[dict],
    namespace: str = "OrderService"
):
    """
    Put a custom metric to CloudWatch.
    
    Note: CloudWatch charges $0.30/metric/month for custom metrics.
    At 10 metrics × 100 customers = 1000 metrics × $0.30 = $300/month.
    Plan your cardinality accordingly.
    """
    cloudwatch.put_metric_data(
        Namespace=namespace,
        MetricData=[{
            'MetricName': metric_name,
            'Value': value,
            'Unit': unit,
            'Timestamp': datetime.utcnow(),
            'Dimensions': dimensions,
        }]
    )


# Usage:
put_metric(
    metric_name='OrdersProcessed',
    value=1,
    unit='Count',
    dimensions=[
        {'Name': 'CustomerId', 'Value': 'acme-corp'},
        {'Name': 'Status', 'Value': 'success'},
    ]
)

put_metric(
    metric_name='ShipmentCreationDuration',
    value=342.1,  # milliseconds
    unit='Milliseconds',
    dimensions=[
        {'Name': 'Carrier', 'Value': 'fedex'},
    ]
)
```

### CloudWatch Alarms

```python
def create_error_rate_alarm(service_name: str, threshold_percent: float = 5.0):
    """Create a CloudWatch alarm for high error rate."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{service_name}-high-error-rate",
        AlarmDescription=f"Error rate > {threshold_percent}% for {service_name}",
        Namespace="OrderService",
        MetricName="RequestErrorRate",
        Dimensions=[{'Name': 'ServiceName', 'Value': service_name}],
        Statistic='Average',
        Period=300,  # 5 minutes
        EvaluationPeriods=2,
        Threshold=threshold_percent,
        ComparisonOperator='GreaterThanThreshold',
        TreatMissingData='breaching',
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789:ops-alerts'
        ],
    )
```

---

## 13. Full Working Example

### Complete FastAPI Service with Prometheus Metrics

```python
# Complete order service with metrics
# requirements.txt:
# fastapi uvicorn prometheus-client structlog httpx

import time
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI, Response, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import structlog
from prometheus_client import (
    Counter, Histogram, Gauge, 
    generate_latest, CONTENT_TYPE_LATEST
)
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request

# ---- Metrics Definitions ----

HTTP_REQUESTS = Counter(
    'http_requests_total',
    'HTTP requests',
    ['method', 'path', 'status']
)

HTTP_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'path'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

ORDERS_CREATED = Counter(
    'orders_created_total',
    'Orders created',
    ['customer_id', 'status']
)

ORDER_PROCESSING_TIME = Histogram(
    'order_processing_seconds',
    'Order processing time',
    ['customer_id'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)

ACTIVE_ORDERS = Gauge(
    'active_orders_count',
    'Orders currently being processed'
)

# ---- Middleware ----

def normalize_path(path: str) -> str:
    import re
    path = re.sub(r'/\d+', '/{id}', path)
    path = re.sub(r'/[0-9a-f-]{36}', '/{uuid}', path)
    return path

class MetricsMiddleware(BaseHTTPMiddleware):
    SKIP = {'/health', '/metrics'}
    
    async def dispatch(self, request: Request, call_next):
        if request.url.path in self.SKIP:
            return await call_next(request)
        
        path = normalize_path(request.url.path)
        start = time.monotonic()
        
        try:
            response = await call_next(request)
        except Exception:
            HTTP_REQUESTS.labels(
                method=request.method, path=path, status="500"
            ).inc()
            raise
        
        duration = time.monotonic() - start
        status = str(response.status_code)[0] + "xx"  # e.g., "2xx", "4xx", "5xx"
        
        HTTP_REQUESTS.labels(
            method=request.method, path=path, status=status
        ).inc()
        HTTP_DURATION.labels(method=request.method, path=path).observe(duration)
        
        return response


# ---- Application ----

log = structlog.get_logger()

@asynccontextmanager
async def lifespan(app: FastAPI):
    log.info("service_starting")
    yield
    log.info("service_stopping")

app = FastAPI(lifespan=lifespan)
app.add_middleware(MetricsMiddleware)


class OrderItem(BaseModel):
    sku: str
    quantity: int
    unit_price: float

class CreateOrderRequest(BaseModel):
    order_id: str
    customer_id: str
    items: List[OrderItem]

class OrderResponse(BaseModel):
    order_id: str
    status: str
    total_value: float


@app.post("/api/v1/orders", response_model=OrderResponse)
async def create_order(request: CreateOrderRequest):
    log.info("order_received", order_id=request.order_id, customer_id=request.customer_id)
    
    ACTIVE_ORDERS.inc()
    start = time.monotonic()
    
    try:
        # Validate
        if not request.items:
            ORDERS_CREATED.labels(customer_id=request.customer_id, status="validation_error").inc()
            raise HTTPException(status_code=422, detail="Order must have items")
        
        for item in request.items:
            if item.quantity <= 0:
                ORDERS_CREATED.labels(customer_id=request.customer_id, status="validation_error").inc()
                raise HTTPException(status_code=422, detail=f"Invalid quantity for {item.sku}")
        
        # Simulate processing time
        await simulate_processing(request.customer_id)
        
        total = sum(i.quantity * i.unit_price for i in request.items)
        
        ORDERS_CREATED.labels(customer_id=request.customer_id, status="success").inc()
        log.info("order_created", order_id=request.order_id, total_value=total)
        
        return OrderResponse(
            order_id=request.order_id,
            status="created",
            total_value=total,
        )
    except HTTPException:
        raise
    except Exception as e:
        ORDERS_CREATED.labels(customer_id=request.customer_id, status="error").inc()
        log.error("order_creation_failed", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal error")
    finally:
        ACTIVE_ORDERS.dec()
        duration = time.monotonic() - start
        ORDER_PROCESSING_TIME.labels(customer_id=request.customer_id).observe(duration)


async def simulate_processing(customer_id: str):
    import asyncio
    # Simulate different latencies per customer
    latencies = {"acme-corp": 0.1, "globex": 0.5, "initech": 2.0}
    await asyncio.sleep(latencies.get(customer_id, 0.2))


@app.get("/metrics")
async def metrics_endpoint():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)


@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## 14. FDE Context: The CTO Dashboard

### The Scenario

You're 3 months into an engagement with Global Freight Solutions. Their CTO asks: "Can we have a dashboard that shows the health of our integration? We want to show it in our weekly ops review."

### What Makes a CTO Dashboard

A CTO dashboard is different from an engineering dashboard:
- **Big numbers and clear status** — not tiny time series
- **Business language** — "orders processed" not "request rate"
- **Customer-specific** — show THEIR metrics, not platform-wide
- **Self-explanatory** — no one should need to explain what they're looking at

### Dashboard Design for a Logistics Customer

```
Row 1 — Today's Summary (stat panels with big numbers)
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ 14,237           │ │ 99.7%            │ │ 1.2s             │ │ 3                │
│ Orders Today     │ │ Success Rate     │ │ Avg Processing   │ │ Errors Today     │
│ ▲ +12% vs yday  │ │ ▼ -0.1%          │ │ ▲ +0.2s          │ │ ▼ -5 vs yday     │
└─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘

Row 2 — Time Series
┌─────────────────────────────────┐ ┌─────────────────────────────────┐
│ Orders per Hour (last 7 days)   │ │ Error Rate % (last 7 days)      │
│ [time series chart]             │ │ [time series chart - green/red]  │
└─────────────────────────────────┘ └─────────────────────────────────┘

Row 3 — EDI-Specific (if they use EDI)
┌─────────────────────────────┐ ┌──────────────────────────────┐
│ EDI Files Processed Today   │ │ Mapping Errors by Type       │
│ 856: 1,204   ✓              │ │ [bar chart by error type]    │
│ 850: 1,187   ✓              │ │                              │
│ 997: 2,391   ✓              │ │                              │
└─────────────────────────────┘ └──────────────────────────────┘

Row 4 — Shipment Tracking
┌─────────────────────────────────────────────────────────────────────────────┐
│ Shipments by Status (stacked bar, last 7 days)                              │
│ [created] [in-transit] [delivered] [exception]                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Grafana Dashboard That Pays for Itself

When you show this dashboard to a CTO in a quarterly business review:
1. They see their data in their language
2. They can identify trends without asking you
3. When something spikes, they know where to look
4. It demonstrates ongoing value of the integration

**FDE pro tip**: Create a read-only Grafana share link and give it to the customer's ops team. They can see it without needing an account. This becomes a self-service tool that reduces your support load.

---

## 15. Common Failure Modes

### Failure Mode 1: High Cardinality Labels Crashing Prometheus

**Symptom**: Prometheus uses gigabytes of memory, becomes slow or crashes.

**Cause**: Using high-cardinality values as metric labels:
```python
# WRONG — creates millions of time series
HTTP_REQUESTS.labels(
    user_id="user-12345",     # millions of users
    order_id="ORD-99999",     # millions of orders
    ip_address="1.2.3.4",     # millions of IPs
).inc()
```

**Fix**: Only use low-cardinality labels (< 100 unique values):
```python
# RIGHT — limited cardinality
HTTP_REQUESTS.labels(
    method="POST",           # ~5 values
    path="/api/v1/orders",   # ~20 normalized paths
    status="2xx",            # ~5 values
).inc()

# For per-customer metrics, have at most ~100 customers
ORDERS_CREATED.labels(customer_id="acme-corp", status="success").inc()
```

### Failure Mode 2: Prometheus Not Scraping the /metrics Endpoint

**Symptom**: No metrics appear in Prometheus despite service running.

**Debug**:
```bash
# Check if Prometheus can reach the endpoint
curl http://order-service:8001/metrics | head -20

# Check Prometheus targets page
# http://prometheus:9090/targets
# Should show order-service as UP
```

**Common causes**:
- Wrong port in prometheus.yml
- Service is on a different network than Prometheus
- `/metrics` endpoint requires auth but Prometheus not configured with credentials
- `prometheus-client` not calling `generate_latest(REGISTRY)` correctly

### Failure Mode 3: Counter Value Resetting Silently

**Symptom**: Your "orders processed" counter suddenly drops to 0.

**Cause**: Service restarted. Counters start at 0 on each restart.

**Fix**: This is expected behavior. Use `increase()` or `rate()` in PromQL — these functions handle counter resets automatically by detecting when the value drops.

```promql
# WRONG — raw counter value (shows reset as a drop)
orders_processed_total

# RIGHT — rate/increase handles resets automatically
increase(orders_processed_total[1h])  # orders in the last hour
rate(orders_processed_total[5m])      # orders/second
```

### Failure Mode 4: Histogram Buckets Not Capturing Your Data

**Symptom**: `histogram_quantile` returns `+Inf` or unrealistically high values.

**Cause**: Your observations are higher than all defined buckets, so they all fall in the `+Inf` bucket.

**Debug**:
```promql
# Check what fraction of observations are in the +Inf bucket
rate(http_request_duration_seconds_bucket{le="+Inf"}[5m])
vs
rate(http_request_duration_seconds_bucket{le="10.0"}[5m])
# If these differ, you have observations > 10 seconds
```

**Fix**: Add higher buckets:
```python
# If you have requests taking up to 60 seconds:
HTTP_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0]
)
```

---

## 16. Interview Angle

**Q: "Explain the RED method and why it matters."**

**Strong answer**: "RED stands for Rate, Errors, Duration — the three metrics every service should expose. Rate is requests per second, which tells you if the service is receiving expected traffic. Errors is the fraction of requests that fail, which is your primary health signal. Duration is latency percentiles — specifically p95 and p99, because averages hide tail latency. Together, these three cover the fundamental question: 'Is the service working correctly?' Every service I build has a Grafana dashboard with these three panels, and alerting configured on error rate and latency thresholds. When a customer asks 'is our integration healthy?', I can pull up this dashboard and give them an instant answer."

---

**Q: "What's the difference between a counter, gauge, and histogram? When do you use each?"**

**Strong answer**: "Counter for anything cumulative that only goes up — total requests, total orders processed, total errors. You always query the rate of change with `rate()`. Gauge for current state — queue depth, active connections, memory usage. Histogram for distributions — request latency, payload size. Histograms are the most powerful because they let you compute percentiles. If you only remember one: use histograms for any timing data, because 'average latency is 200ms' hides the fact that 1% of requests take 10 seconds. p99 latency is what your customers actually experience."

---

**Q: "How do you avoid high-cardinality label problems in Prometheus?"**

**Strong answer**: "Cardinality is the number of unique time series a metric creates. If you have 10 label combinations with 10 values each, that's 10^10 time series — Prometheus will run out of memory. The rule is: never use values with unbounded cardinality as labels. No user IDs, order IDs, request IDs, or IP addresses. Use normalized paths like `/api/v1/orders/{id}` not `/api/v1/orders/12345`. For per-customer metrics, it's okay to use customer_id if you have fewer than a few hundred customers. If you need to find specific orders, use logs with structured queries — that's what they're for."

---

## 17. Practice Exercise

### Exercise: RED Dashboard for a Logistics Integration

**Goal**: Build a FastAPI order processing service with full Prometheus metrics and a Grafana dashboard showing RED + business metrics.

**Setup**:
```bash
# Start Prometheus + Grafana
docker run -d -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus

docker run -d -p 3000:3000 grafana/grafana
```

**Service to build**:

1. FastAPI order service with these metrics:
   - `http_requests_total{method, path, status}` (Counter)
   - `http_request_duration_seconds{method, path}` (Histogram)
   - `orders_created_total{customer_id, status}` (Counter)
   - `active_orders_count` (Gauge)
   - `shipment_creation_seconds{carrier}` (Histogram)
   - `queue_depth_messages{queue_name}` (Gauge)

2. The service should have multiple "customers" with different latency profiles:
   - customer "acme-corp": fast (100ms)
   - customer "globex": medium (500ms)
   - customer "initech": slow (2s)
   - 5% random error rate

3. A load generator that sends 10 requests/second with a mix of customers:
```python
# load_generator.py
import httpx, random, asyncio, time

customers = ["acme-corp", "globex", "initech"]
sku_list = ["SKU-001", "SKU-002", "SKU-003"]

async def send_order(client, i):
    customer = random.choice(customers)
    data = {
        "order_id": f"ORD-{i}",
        "customer_id": customer,
        "items": [{"sku": random.choice(sku_list), "quantity": random.randint(1, 10), "unit_price": 9.99}]
    }
    try:
        await client.post("http://localhost:8000/api/v1/orders", json=data, timeout=5.0)
    except:
        pass

async def main():
    async with httpx.AsyncClient() as client:
        i = 0
        while True:
            await send_order(client, i)
            i += 1
            await asyncio.sleep(0.1)  # 10 req/s

asyncio.run(main())
```

4. **Grafana dashboard** with:
   - Stat panel: total orders processed today
   - Stat panel: current success rate (should be ~95%)
   - Time series: orders per minute by customer (should see acme > globex, globex > initech)
   - Time series: p99 latency by customer (should see initech >> globex >> acme)
   - Time series: error rate % (should be ~5%)
   - Alert: fire Grafana alert when error rate > 10%

5. **Verify your dashboard by introducing a fault**:
   - Change the "initech" customer's error rate from 5% to 50%
   - Observe the error rate spike in Grafana within 30 seconds
   - Observe the "per-customer" breakdown showing initech is the source
   - This is the FDE debugging experience: dashboard tells you which customer is affected

**Stretch goal**: Add a `mapping_errors_total{customer_id, error_type}` counter. Simulate specific mapping errors for "initech" (missing required field, invalid date format). Create a Grafana panel showing "mapping error rate by customer and type" — this is the panel that turns into a customer support conversation: "initech, your date format changed on Tuesday."

---

**Next**: [04-log-aggregation.md](./04-log-aggregation.md) — Where all these logs go and how to query them at scale.
