# 01 — Async/Await in Python: Deep Dive for FDE Work

> **Related files:** [02-pydantic-type-hints.md](./02-pydantic-type-hints.md) | [../B-FastAPI/01-async-endpoints-di.md](../B-FastAPI/01-async-endpoints-di.md) | [../B-FastAPI/04-background-tasks-streaming.md](../B-FastAPI/04-background-tasks-streaming.md)

---

## Why This Matters Before You Read a Line of Code

You are sitting with a customer's integration team. Their warehouse management system needs to confirm a shipment. Your service must:

1. Call the carrier API to get a tracking number
2. Call the customer's ERP to update the order status
3. Call an internal audit service to log the event
4. Respond to the caller in under 2 seconds

If you do these sequentially: 400ms + 350ms + 200ms = 950ms plus overhead — tight but workable.

If the carrier API degrades to 1.2s: 1200ms + 350ms + 200ms = 1750ms — you're failing SLAs.

With async, steps 1+2+3 run **concurrently**: max(1200, 350, 200) = 1200ms. You just absorbed the carrier's performance degradation without breaking your SLA.

That is the practical reason async exists. Everything below explains how to use it correctly so you don't accidentally make things *slower* or introduce subtle bugs.

---

## Part 1: What the Event Loop Actually Is

### The Single Thread That Does Everything

Python's asyncio event loop is a **single-threaded scheduler**. It runs in one OS thread, but it can manage thousands of concurrent I/O operations. The key insight:

**Most of a network call's time is spent waiting, not computing.**

When you do `requests.get("https://api.carrier.com/track/123")`, your Python process:
1. Sends bytes to the OS network stack (fast, microseconds)
2. **Blocks** — sits doing nothing while the TCP packet travels to the server, the server processes it, and a response comes back
3. Receives bytes and returns them to you

During step 2, your CPU is idle. The event loop's job is to use that idle time to run other coroutines.

### The Event Loop Mechanics

```
                    ┌─────────────────────────────────────────┐
                    │           Event Loop (1 thread)          │
                    │                                          │
                    │  ┌─────────────────────────────────┐    │
                    │  │         Ready Queue              │    │
                    │  │  [coro_A, coro_B, coro_C]       │    │
                    │  └─────────────────┬───────────────┘    │
                    │                    │ pick next           │
                    │                    ▼                     │
                    │  ┌─────────────────────────────────┐    │
                    │  │      Run coroutine until        │    │
                    │  │      it hits an `await`         │    │
                    │  └─────────────────┬───────────────┘    │
                    │                    │                     │
                    │         ┌──────────┴──────────┐         │
                    │         │                     │         │
                    │    awaitable is          awaitable is   │
                    │    I/O wait              already ready  │
                    │         │                     │         │
                    │         ▼                     ▼         │
                    │  Register callback      Put back in     │
                    │  with OS selector       ready queue     │
                    │  (epoll/kqueue)                         │
                    │         │                               │
                    │         └── go pick next coro ──────►  │
                    └─────────────────────────────────────────┘
```

The OS selector (epoll on Linux, kqueue on macOS) is the magic. The event loop tells the OS: "wake me up when any of these file descriptors become readable/writable." It can register thousands of these. Then it sleeps efficiently, not spin-waiting.

### Proof: The Event Loop Is Real

```python
import asyncio
import threading

async def show_context():
    loop = asyncio.get_event_loop()
    print(f"Thread: {threading.current_thread().name}")
    print(f"Loop: {id(loop)}")
    await asyncio.sleep(0)  # yield control
    # We're back in the SAME thread, SAME loop
    print(f"Still thread: {threading.current_thread().name}")
    print(f"Still loop: {id(loop)}")

asyncio.run(show_context())
# Thread: MainThread
# Loop: 140234567890
# Still thread: MainThread
# Still loop: 140234567890
```

The coroutine suspends and resumes in the same thread. No OS context switch. No lock acquisition. This is why async is more efficient than threads for I/O-bound work.

---

## Part 2: async def, Coroutines, and the await Keyword

### What `async def` Creates

```python
def regular_function():
    return 42

async def coroutine_function():
    return 42

# Regular function: call it, get the result
result = regular_function()          # result = 42

# Coroutine function: call it, get a COROUTINE OBJECT (not the result!)
coro = coroutine_function()          # coro = <coroutine object coroutine_function>
print(coro)                          # <coroutine object coroutine_function at 0x...>

# The code hasn't run yet. You need to await it or run it.
result = asyncio.run(coroutine_function())  # result = 42
```

This is the most common beginner mistake: calling an async function and wondering why it returns a coroutine object instead of the result.

### What `await` Does

`await` has two jobs:
1. **Unwrap** the awaitable (coroutine, Task, Future) to get its result
2. **Yield control** back to the event loop if the awaitable isn't ready yet

```python
import asyncio
import time

async def fetch_from_carrier(tracking_id: str) -> dict:
    """Simulates a real HTTP call to a carrier API."""
    print(f"  → Starting carrier fetch for {tracking_id}")
    await asyncio.sleep(0.4)  # simulates network I/O
    print(f"  ← Carrier fetch complete for {tracking_id}")
    return {"tracking_id": tracking_id, "status": "IN_TRANSIT", "carrier": "FedEx"}

async def main():
    start = time.perf_counter()
    result = await fetch_from_carrier("TRACK123")
    elapsed = time.perf_counter() - start
    print(f"Result: {result}")
    print(f"Elapsed: {elapsed:.3f}s")

asyncio.run(main())
# → Starting carrier fetch for TRACK123
# ← Carrier fetch complete for TRACK123
# Result: {'tracking_id': 'TRACK123', 'status': 'IN_TRANSIT', 'carrier': 'FedEx'}
# Elapsed: 0.401s
```

### What Happens at `await asyncio.sleep(0.4)`

1. `asyncio.sleep(0.4)` creates a Future that will be resolved after 0.4 seconds
2. `await` sees the Future is not ready, suspends `main()`
3. Control returns to the event loop
4. Event loop checks its timer heap — in 0.4s, this Future should resolve
5. Event loop can run other coroutines in the meantime (if any exist)
6. After 0.4s, the event loop sets the Future's result, marks `main()` as ready
7. `main()` resumes from the `await` line

---

## Part 3: Coroutines vs Threads vs Processes

This is a question that comes up in FDE interviews. You need to give a precise, practical answer.

```
                    COROUTINES          THREADS             PROCESSES
                    ──────────────────  ──────────────────  ──────────────────
Concurrency type    Cooperative         Preemptive          True parallel
Context switch      Explicit (await)    OS-controlled       OS-controlled
OS overhead         None                ~8KB stack each     ~MBs per process
Shared memory       Yes (same thread)   Yes (GIL limited)   No (IPC needed)
GIL impact          N/A (1 thread)      Yes (CPU bound)     No (separate GIL)
Good for            I/O bound tasks     Blocking lib code   CPU bound tasks
Thread safety       Safe (1 thread)     Needs locks         No sharing needed
Max concurrent      Tens of thousands   Hundreds            Dozens
```

### When to Use Each

**Coroutines (asyncio)**
- HTTP calls (carrier APIs, ERP REST APIs, webhooks)
- Database queries with async drivers (asyncpg, motor)
- File I/O with aiofiles
- WebSocket connections
- Message queue consumers (aio-pika for RabbitMQ)

**Threads**
- Calling a library that has no async version (legacy SDK, SOAP client, some database drivers)
- CPU-light blocking operations (filesystem calls on some systems)
- When you need to run async code from sync code

**Processes**
- Heavy data transformation (pandas on large datasets)
- ML model inference
- Image/video processing
- Anything that needs to fully utilize multiple CPUs

### Real Example: Sequential vs Concurrent

```python
import asyncio
import httpx
import time
from typing import Any

# ─── Sequential (the naive way) ───────────────────────────────────────────────

async def get_tracking_status_sequential(tracking_ids: list[str]) -> list[dict]:
    """Calls carrier API one at a time. 10 calls × 400ms = 4 seconds."""
    async with httpx.AsyncClient() as client:
        results = []
        for tid in tracking_ids:
            response = await client.get(
                f"https://api.carrier.com/track/{tid}",
                timeout=5.0
            )
            results.append(response.json())
        return results

# ─── Concurrent (the right way) ───────────────────────────────────────────────

async def get_tracking_status_concurrent(tracking_ids: list[str]) -> list[dict]:
    """Calls carrier API concurrently. 10 calls × 400ms = ~400ms (+ overhead)."""
    async with httpx.AsyncClient() as client:
        tasks = [
            client.get(f"https://api.carrier.com/track/{tid}", timeout=5.0)
            for tid in tracking_ids
        ]
        responses = await asyncio.gather(*tasks)
        return [r.json() for r in responses]

# ─── Demo ─────────────────────────────────────────────────────────────────────

async def demo():
    tracking_ids = [f"TRACK{i:03d}" for i in range(10)]
    
    start = time.perf_counter()
    # In real code, these would hit a real API
    # Here we simulate with sleep
    await asyncio.sleep(0.4 * len(tracking_ids))  # sequential simulation
    sequential_time = time.perf_counter() - start
    
    start = time.perf_counter()
    await asyncio.sleep(0.4)  # concurrent simulation — all run at once
    concurrent_time = time.perf_counter() - start
    
    print(f"Sequential: {sequential_time:.2f}s")
    print(f"Concurrent: {concurrent_time:.2f}s")
    print(f"Speedup: {sequential_time / concurrent_time:.1f}x")
```

---

## Part 4: asyncio.gather(), asyncio.create_task(), asyncio.wait()

These three functions are your primary tools for concurrent execution. Each has different semantics.

### asyncio.gather() — The Workhorse

`gather()` runs awaitables concurrently and returns results in the same order as input.

```python
import asyncio
import httpx
from typing import NamedTuple

class ShipmentInfo(NamedTuple):
    tracking_data: dict
    order_data: dict
    inventory_data: dict

async def fetch_tracking(client: httpx.AsyncClient, tracking_id: str) -> dict:
    """Hit carrier API."""
    resp = await client.get(f"https://carrier.api/track/{tracking_id}")
    resp.raise_for_status()
    return resp.json()

async def fetch_order(client: httpx.AsyncClient, order_id: str) -> dict:
    """Hit ERP API."""
    resp = await client.get(f"https://erp.internal/orders/{order_id}")
    resp.raise_for_status()
    return resp.json()

async def fetch_inventory(client: httpx.AsyncClient, sku: str) -> dict:
    """Hit WMS API."""
    resp = await client.get(f"https://wms.internal/inventory/{sku}")
    resp.raise_for_status()
    return resp.json()

async def get_full_shipment_info(
    tracking_id: str,
    order_id: str,
    sku: str
) -> ShipmentInfo:
    """
    Fetches all three data sources concurrently.
    Total time = max(carrier, erp, wms) instead of carrier + erp + wms
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        tracking_data, order_data, inventory_data = await asyncio.gather(
            fetch_tracking(client, tracking_id),
            fetch_order(client, order_id),
            fetch_inventory(client, sku),
        )
    return ShipmentInfo(tracking_data, order_data, inventory_data)

# ─── gather() with error handling ─────────────────────────────────────────────

async def get_shipment_info_with_partial_failure(
    tracking_id: str,
    order_id: str,
    sku: str
) -> dict:
    """
    return_exceptions=True means gather won't cancel other tasks when one fails.
    You get exceptions as return values instead of them being raised.
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        results = await asyncio.gather(
            fetch_tracking(client, tracking_id),
            fetch_order(client, order_id),
            fetch_inventory(client, sku),
            return_exceptions=True  # Critical: don't cancel sibling tasks on failure
        )

    tracking_data, order_data, inventory_data = results

    # Check each result individually
    response = {}
    
    if isinstance(tracking_data, Exception):
        response["tracking_error"] = f"Carrier API failed: {tracking_data}"
    else:
        response["tracking"] = tracking_data

    if isinstance(order_data, Exception):
        response["order_error"] = f"ERP API failed: {order_data}"
    else:
        response["order"] = order_data

    if isinstance(inventory_data, Exception):
        response["inventory_error"] = f"WMS API failed: {inventory_data}"
    else:
        response["inventory"] = inventory_data

    return response
```

**Key behavior of `gather()` without `return_exceptions=True`:** if any coroutine raises, the exception is re-raised immediately and other coroutines are **cancelled**. This is almost never what you want in a data-fetching scenario where partial results are better than no results.

### asyncio.create_task() — Background Execution

`create_task()` schedules a coroutine to run on the event loop immediately (as soon as the current coroutine yields). It returns a `Task` object you can await later or ignore.

```python
import asyncio

async def audit_log(event: dict) -> None:
    """Write to audit service — we don't want to make the user wait for this."""
    await asyncio.sleep(0.1)  # simulate audit service call
    print(f"Audit logged: {event}")

async def process_shipment(shipment_id: str) -> dict:
    """
    Process shipment, then log to audit service.
    We don't want to make the caller wait for the audit log.
    """
    # Do the actual work
    result = {"shipment_id": shipment_id, "status": "PROCESSED"}
    
    # Fire-and-forget the audit log
    # The task starts running immediately but we don't await it
    task = asyncio.create_task(
        audit_log({"event": "shipment_processed", "id": shipment_id})
    )
    
    # Return to caller immediately — audit log runs in background
    return result

async def main():
    result = await process_shipment("SHIP001")
    print(f"Got result: {result}")
    # Give the event loop a moment to run the background task
    await asyncio.sleep(0.2)

asyncio.run(main())
# Got result: {'shipment_id': 'SHIP001', 'status': 'PROCESSED'}
# Audit logged: {'event': 'shipment_processed', 'id': 'SHIP001'}
```

**Warning:** If you create a task and never await it, exceptions will be silently swallowed (Python will log a warning, but won't crash). Always attach a callback to handle errors:

```python
def handle_task_exception(task: asyncio.Task) -> None:
    """Callback to log exceptions from fire-and-forget tasks."""
    try:
        task.result()
    except asyncio.CancelledError:
        pass  # task was cancelled, this is fine
    except Exception as exc:
        import logging
        logging.error(f"Background task failed: {exc}", exc_info=True)

async def safe_fire_and_forget(coro) -> asyncio.Task:
    """Schedule a coroutine as a background task with error handling."""
    task = asyncio.create_task(coro)
    task.add_done_callback(handle_task_exception)
    return task

# Usage
await safe_fire_and_forget(audit_log({"event": "something"}))
```

### asyncio.wait() — Fine-Grained Control

`asyncio.wait()` gives you control over what to do when tasks complete: wait for the first one to finish, wait for all, wait for first exception, etc.

```python
import asyncio
import httpx

async def call_carrier_api(client: httpx.AsyncClient, carrier: str, tid: str) -> dict:
    """Try a specific carrier's tracking API."""
    resp = await client.get(f"https://{carrier}.api/track/{tid}", timeout=3.0)
    resp.raise_for_status()
    return {"carrier": carrier, "data": resp.json()}

async def get_tracking_fastest_carrier(tracking_id: str) -> dict:
    """
    Try all carriers simultaneously, return the FIRST one that succeeds.
    This is useful when you have a tracking ID but don't know the carrier.
    Pattern: race condition / first-wins
    """
    async with httpx.AsyncClient() as client:
        tasks = {
            asyncio.create_task(call_carrier_api(client, carrier, tracking_id))
            for carrier in ["fedex", "ups", "usps", "dhl"]
        }
        
        try:
            # FIRST_COMPLETED: return as soon as any task finishes
            done, pending = await asyncio.wait(
                tasks,
                return_when=asyncio.FIRST_COMPLETED
            )
            
            # Cancel all remaining carrier calls
            for task in pending:
                task.cancel()
            
            # Get the result from the first completed task
            winner = done.pop()
            return winner.result()
            
        except Exception:
            # Cancel everything on error
            for task in tasks:
                task.cancel()
            raise

async def get_tracking_all_with_timeout(tracking_id: str) -> list[dict]:
    """
    Try all carriers, collect all that respond within 2 seconds.
    Pattern: best-effort parallel fetch with timeout
    """
    async with httpx.AsyncClient() as client:
        tasks = [
            asyncio.create_task(call_carrier_api(client, carrier, tracking_id))
            for carrier in ["fedex", "ups", "usps", "dhl"]
        ]
        
        # ALL_COMPLETED with a timeout
        done, pending = await asyncio.wait(tasks, timeout=2.0)
        
        # Cancel tasks that didn't finish in time
        for task in pending:
            task.cancel()
        
        results = []
        for task in done:
            try:
                results.append(task.result())
            except Exception as e:
                # Individual carrier failure — skip it
                print(f"Carrier task failed: {e}")
        
        return results
```

### Comparison Summary

| Function | Returns results in order | Stops on first error | Best for |
|---|---|---|---|
| `gather(*coros)` | Yes | Yes (without flag) | Fixed set of concurrent calls |
| `gather(*coros, return_exceptions=True)` | Yes | No | Fixed set, partial failure OK |
| `create_task(coro)` | N/A (you await individually) | No | Background work, fire-and-forget |
| `wait(tasks, FIRST_COMPLETED)` | No (use task.result()) | No | Race pattern, take first winner |
| `wait(tasks, ALL_COMPLETED, timeout=t)` | No | No | Best-effort with timeout |

---

## Part 5: Running Blocking Code in Threads

This is critical. If you call a blocking function inside an async coroutine, you **block the entire event loop** — all other concurrent coroutines freeze until it returns.

```python
import asyncio
import time

# ❌ WRONG: This blocks the entire event loop
async def bad_example():
    time.sleep(2)  # ALL OTHER COROUTINES FROZEN FOR 2 SECONDS
    return "done"

# ✅ RIGHT: Run blocking code in a thread pool
async def good_example():
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, time.sleep, 2)  # other coroutines run freely
    return "done"
```

### `run_in_executor()` Patterns

```python
import asyncio
import concurrent.futures
from pathlib import Path
import pandas as pd
import requests  # sync HTTP library (legacy code)

# ─── Basic Usage ─────────────────────────────────────────────────────────────

async def read_large_csv_async(filepath: str) -> pd.DataFrame:
    """
    pandas read_csv is synchronous. Run it in a thread executor.
    The event loop is free to handle other requests while pandas reads the file.
    """
    loop = asyncio.get_event_loop()
    df = await loop.run_in_executor(
        None,  # None = use default ThreadPoolExecutor
        pd.read_csv,
        filepath
    )
    return df

# ─── With lambda for multiple args ──────────────────────────────────────────

async def transform_data_async(df: pd.DataFrame) -> pd.DataFrame:
    """Heavy pandas transformation — offload to thread."""
    loop = asyncio.get_event_loop()
    
    def _transform(dataframe: pd.DataFrame) -> pd.DataFrame:
        # This runs in a thread — can use any sync code
        dataframe = dataframe.copy()
        dataframe["normalized"] = dataframe["value"].str.strip().str.upper()
        dataframe["date"] = pd.to_datetime(dataframe["date"], format="%Y-%m-%d")
        return dataframe.dropna(subset=["shipment_id"])
    
    result = await loop.run_in_executor(None, _transform, df)
    return result

# ─── With functools.partial for multiple keyword args ─────────────────────────

import functools

async def call_legacy_api(endpoint: str, payload: dict) -> dict:
    """
    Legacy customer SDK that only has sync methods.
    Wrap it to not block the event loop.
    """
    loop = asyncio.get_event_loop()
    
    # Can't pass keyword args directly to run_in_executor
    # Use functools.partial to bind them
    sync_call = functools.partial(
        requests.post,
        f"https://legacy.erp.customer.com/{endpoint}",
        json=payload,
        headers={"Authorization": "Bearer secret"},
        timeout=30
    )
    
    response = await loop.run_in_executor(None, sync_call)
    return response.json()

# ─── With a custom thread pool ───────────────────────────────────────────────

# Create a dedicated thread pool for CPU-heavy work
cpu_pool = concurrent.futures.ThreadPoolExecutor(
    max_workers=4,
    thread_name_prefix="cpu-worker"
)

# Create a dedicated thread pool for I/O-bound legacy code
io_pool = concurrent.futures.ThreadPoolExecutor(
    max_workers=20,
    thread_name_prefix="io-worker"
)

async def process_shipment_batch(shipments: list[dict]) -> list[dict]:
    """Use ProcessPoolExecutor for CPU-bound work."""
    loop = asyncio.get_event_loop()
    
    def _heavy_computation(data: list[dict]) -> list[dict]:
        # Complex validation, geocoding, rate calculation
        import time
        time.sleep(1)  # simulating CPU work
        return [{"id": s["id"], "processed": True} for s in data]
    
    # ProcessPoolExecutor bypasses the GIL entirely — true parallelism
    process_pool = concurrent.futures.ProcessPoolExecutor(max_workers=2)
    result = await loop.run_in_executor(process_pool, _heavy_computation, shipments)
    process_pool.shutdown(wait=False)
    return result
```

---

## Part 6: Common Mistakes and How to Debug Them

### Mistake 1: Blocking the Event Loop

The most common async bug. Hard to notice because the code "works" — it just becomes a bottleneck under load.

```python
import asyncio
import time
import httpx

# ❌ BLOCKING: uses requests (sync), called inside async function
async def bad_fetch(url: str) -> dict:
    import requests
    response = requests.get(url)  # BLOCKS EVENT LOOP
    return response.json()

# ❌ BLOCKING: CPU-heavy work without executor
async def bad_transform(data: list) -> list:
    # Heavy computation blocks the event loop
    return sorted(data, key=lambda x: complex_calculation(x))

# ✅ CORRECT: async HTTP client
async def good_fetch(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

# ✅ CORRECT: CPU work in executor
async def good_transform(data: list) -> list:
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(
        None,
        lambda: sorted(data, key=lambda x: complex_calculation(x))
    )

# How to DETECT event loop blocking:
# Use asyncio's debug mode — it logs when any callback/coroutine takes >100ms
import asyncio.events

async def monitor_blocking():
    # Enable debug mode — warns about slow callbacks
    loop = asyncio.get_event_loop()
    loop.set_debug(True)
    loop.slow_callback_duration = 0.1  # warn if anything takes >100ms
```

### Mistake 2: Forgetting await

Python will not raise an error if you forget `await`. You'll get a coroutine object instead of a result.

```python
import asyncio

async def get_data() -> dict:
    await asyncio.sleep(0.1)
    return {"key": "value"}

async def bad_usage():
    # ❌ WRONG: result is a coroutine object, not a dict
    result = get_data()  # forgot await
    print(result)  # <coroutine object get_data at 0x...>
    print(result["key"])  # TypeError: 'coroutine' object is not subscriptable

async def good_usage():
    # ✅ CORRECT
    result = await get_data()
    print(result)  # {'key': 'value'}
    print(result["key"])  # 'value'

# Python will show a RuntimeWarning:
# RuntimeWarning: coroutine 'get_data' was never awaited
# Enable warnings as errors in tests:
# pytest -W error::RuntimeWarning
```

### Mistake 3: Not Using `return_exceptions=True` in gather()

```python
import asyncio

async def risky_call(id: int) -> dict:
    if id == 3:
        raise ValueError(f"Item {id} failed")
    await asyncio.sleep(0.1)
    return {"id": id}

async def bad_gather():
    # ❌ If id=3 fails, ALL results are lost — including the successful ones
    try:
        results = await asyncio.gather(
            risky_call(1),
            risky_call(2),
            risky_call(3),
            risky_call(4),
        )
    except ValueError as e:
        print(f"Lost all results because of: {e}")

async def good_gather():
    # ✅ Get all results, failures are returned as exceptions
    results = await asyncio.gather(
        risky_call(1),
        risky_call(2),
        risky_call(3),
        risky_call(4),
        return_exceptions=True
    )
    
    for i, result in enumerate(results, 1):
        if isinstance(result, Exception):
            print(f"Item {i} failed: {result}")
        else:
            print(f"Item {i} succeeded: {result}")
```

### Mistake 4: Creating Tasks Without Keeping References

```python
import asyncio

# ❌ WRONG: Python's garbage collector may collect the task before it finishes
async def bad_task_creation():
    asyncio.create_task(some_background_work())  # no reference kept
    # Task may be GC'd and silently cancelled

# ✅ CORRECT: Keep a reference to prevent GC
_background_tasks: set[asyncio.Task] = set()

async def good_task_creation():
    task = asyncio.create_task(some_background_work())
    _background_tasks.add(task)
    task.add_done_callback(_background_tasks.discard)  # clean up when done
```

### Mistake 5: Mixing Sync and Async Incorrectly

```python
import asyncio

async def async_function():
    return "result"

# ❌ WRONG: Can't await in sync code
def sync_function():
    result = await async_function()  # SyntaxError

# ❌ WRONG: asyncio.run() inside existing event loop (in FastAPI/Jupyter)
def sync_function_in_fastapi():
    result = asyncio.run(async_function())  # RuntimeError: event loop is running

# ✅ CORRECT options:

# Option 1: Make the caller async too
async def better_sync_function():
    result = await async_function()
    return result

# Option 2: Use run_in_executor to call async from sync (rare, complex)
# Usually better to just make everything async

# Option 3: Use a sync wrapper only at the very top level
def main():
    result = asyncio.run(async_function())  # only works when no loop is running
    print(result)
```

---

## Part 7: Async and FastAPI

FastAPI is built on Starlette which is built on asyncio. This has specific implications.

### async def vs def in FastAPI

```python
from fastapi import FastAPI
import asyncio
import time

app = FastAPI()

# ─── Case 1: async def endpoint ──────────────────────────────────────────────
@app.get("/async-endpoint")
async def async_endpoint():
    """
    Runs in the event loop thread.
    GREAT for: httpx calls, async DB queries, anything awaitable
    BAD for: sync blocking calls (will freeze the event loop)
    """
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()

# ─── Case 2: def endpoint ─────────────────────────────────────────────────────
@app.get("/sync-endpoint")
def sync_endpoint():
    """
    FastAPI runs this in a THREADPOOL automatically.
    GREAT for: sync libraries (requests, psycopg2, boto3 without aioboto3)
    Thread pool size = controlled by anyio limiter (default: 40 threads)
    """
    response = requests.get("https://api.example.com/data")  # sync, OK here
    return response.json()

# ─── Case 3: async def with blocking code ─────────────────────────────────────
@app.get("/bad-endpoint")
async def bad_endpoint():
    """
    WORST of both worlds.
    This blocks the event loop but doesn't get the thread pool treatment.
    """
    time.sleep(2)  # BLOCKS EVENT LOOP — all other requests freeze
    return {"message": "done"}

# ─── Case 4: async def with executor for blocking code ────────────────────────
@app.get("/correct-endpoint")
async def correct_endpoint():
    """When you need async but also have blocking code."""
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_legacy_sdk_call)
    return result
```

### The FastAPI Request Lifecycle

```
Request arrives
      │
      ▼
Starlette ASGI handler (async)
      │
      ▼
Middleware stack (each middleware is async)
  - Correlation ID middleware
  - Timing middleware
  - Auth middleware
      │
      ▼
Dependency resolution (async or sync, per dependency)
  - get_db() → yields DB session
  - get_current_user() → verifies JWT
  - get_customer_config() → loads from cache
      │
      ▼
Route handler (async or sync)
      │
      ▼
Response serialization
      │
      ▼
Middleware cleanup (reverse order)
      │
      ▼
Response sent, DB session closed (from yield dependencies)
```

---

## Part 8: Context Variables for Request-Scoped Data

When you have middleware that sets a correlation ID, you need a way to access that ID from anywhere in the call stack without passing it explicitly through every function. `contextvars.ContextVar` solves this.

```python
import asyncio
import contextvars
import uuid
import logging
from fastapi import FastAPI, Request, Response
from fastapi.middleware import Middleware
import httpx

# ─── Define context variables ────────────────────────────────────────────────

# These are thread-safe and async-safe
correlation_id_var: contextvars.ContextVar[str] = contextvars.ContextVar(
    'correlation_id',
    default='no-correlation-id'
)

customer_id_var: contextvars.ContextVar[str | None] = contextvars.ContextVar(
    'customer_id',
    default=None
)

# ─── Middleware to set context vars ──────────────────────────────────────────

app = FastAPI()

@app.middleware("http")
async def correlation_id_middleware(request: Request, call_next):
    """
    Extract correlation ID from incoming request or generate one.
    Store in context var so any code in this request can access it.
    """
    # Check if caller provided a correlation ID (e.g., from their system)
    correlation_id = (
        request.headers.get("X-Correlation-ID") or
        request.headers.get("X-Request-ID") or
        str(uuid.uuid4())
    )
    
    # Set the context variable for this request's execution context
    token = correlation_id_var.set(correlation_id)
    
    try:
        response = await call_next(request)
        # Echo back the correlation ID so caller can trace
        response.headers["X-Correlation-ID"] = correlation_id
        return response
    finally:
        # Reset to prevent leaking into other requests
        correlation_id_var.reset(token)

# ─── Logging with automatic correlation ID ───────────────────────────────────

class CorrelationIDFilter(logging.Filter):
    """Injects correlation ID into every log record for this request."""
    
    def filter(self, record: logging.LogRecord) -> bool:
        record.correlation_id = correlation_id_var.get()
        return True

# Set up logger
logger = logging.getLogger(__name__)
logger.addFilter(CorrelationIDFilter())

# Log format includes correlation ID automatically
logging.basicConfig(
    format='%(asctime)s [%(correlation_id)s] %(levelname)s %(name)s: %(message)s',
    level=logging.INFO
)

# ─── Deep in your code — access correlation ID without passing it ─────────────

async def call_carrier_api(tracking_id: str) -> dict:
    """
    This function doesn't need to accept correlation_id as a parameter —
    it can get it from the context var.
    """
    correlation_id = correlation_id_var.get()
    
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://carrier.api/track/{tracking_id}",
            headers={
                "X-Correlation-ID": correlation_id,  # propagate to downstream
                "Accept": "application/json",
            }
        )
    
    # This log automatically includes the correlation ID
    logger.info(f"Carrier API response for {tracking_id}: status={response.status_code}")
    
    return response.json()

# ─── Context vars are safe across async boundaries ───────────────────────────

async def demonstrate_context_safety():
    """Each concurrent task gets its own copy of context vars."""
    
    async def task_with_context(task_id: str):
        # Set a unique correlation ID for this task
        token = correlation_id_var.set(f"task-{task_id}")
        await asyncio.sleep(0.1)  # yields to event loop
        # After yielding and resuming, the context var still has OUR value
        assert correlation_id_var.get() == f"task-{task_id}"
        correlation_id_var.reset(token)
    
    # Run concurrently — each task maintains its own context
    await asyncio.gather(
        task_with_context("A"),
        task_with_context("B"),
        task_with_context("C"),
    )
```

---

## Part 9: Async Context Managers and Generators

### Async Context Managers

```python
import asyncio
import httpx
from typing import AsyncGenerator, AsyncIterator
from contextlib import asynccontextmanager

# ─── Class-based async context manager ───────────────────────────────────────

class CustomerAPIClient:
    """
    Manages an HTTP client with connection pooling.
    Reusing connections is 3-5x faster than creating new ones per request.
    """
    
    def __init__(self, base_url: str, api_key: str, timeout: float = 30.0):
        self.base_url = base_url
        self.api_key = api_key
        self.timeout = timeout
        self._client: httpx.AsyncClient | None = None
    
    async def __aenter__(self) -> "CustomerAPIClient":
        self._client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"X-API-Key": self.api_key},
            timeout=self.timeout,
            limits=httpx.Limits(
                max_connections=20,
                max_keepalive_connections=10,
                keepalive_expiry=30
            )
        )
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if self._client:
            await self._client.aclose()
    
    async def get_order(self, order_id: str) -> dict:
        assert self._client is not None
        response = await self._client.get(f"/orders/{order_id}")
        response.raise_for_status()
        return response.json()

# Usage
async def process_orders(order_ids: list[str]) -> list[dict]:
    async with CustomerAPIClient(
        base_url="https://erp.customer.com/api/v2",
        api_key="secret-key"
    ) as client:
        results = await asyncio.gather(
            *[client.get_order(oid) for oid in order_ids],
            return_exceptions=True
        )
    return results

# ─── asynccontextmanager decorator ───────────────────────────────────────────

@asynccontextmanager
async def managed_db_transaction(db_pool):
    """
    Async context manager for database transactions.
    Used as a FastAPI dependency (see FastAPI module).
    """
    connection = await db_pool.acquire()
    transaction = connection.transaction()
    await transaction.start()
    
    try:
        yield connection
        await transaction.commit()
    except Exception:
        await transaction.rollback()
        raise
    finally:
        await db_pool.release(connection)

# Usage
async def transfer_inventory(from_warehouse: str, to_warehouse: str, sku: str, qty: int):
    async with managed_db_transaction(db_pool) as conn:
        await conn.execute(
            "UPDATE inventory SET qty = qty - $1 WHERE warehouse = $2 AND sku = $3",
            qty, from_warehouse, sku
        )
        await conn.execute(
            "UPDATE inventory SET qty = qty + $1 WHERE warehouse = $2 AND sku = $3",
            qty, to_warehouse, sku
        )
        # If either fails, transaction is rolled back
```

### Async Generators for Streaming

Async generators are perfect for streaming large datasets without loading everything into memory.

```python
import asyncio
import httpx
import json
from typing import AsyncIterator

# ─── Stream large paginated API response ─────────────────────────────────────

async def stream_all_orders(
    client: httpx.AsyncClient,
    since_date: str
) -> AsyncIterator[dict]:
    """
    Yields orders one by one from a paginated API.
    Never loads all orders into memory at once.
    Perfect for feeding into a streaming process.
    """
    page = 1
    page_size = 100
    
    while True:
        response = await client.get(
            "/orders",
            params={
                "since": since_date,
                "page": page,
                "page_size": page_size
            }
        )
        response.raise_for_status()
        data = response.json()
        
        orders = data["orders"]
        for order in orders:
            yield order  # yield one at a time
        
        # Check if there are more pages
        if len(orders) < page_size or not data.get("has_more"):
            break
        
        page += 1

async def process_orders_stream():
    """Process orders as they stream in, not all at once."""
    async with httpx.AsyncClient(base_url="https://erp.customer.com/api") as client:
        processed = 0
        async for order in stream_all_orders(client, "2024-01-01"):
            # Process each order as it arrives
            await process_single_order(order)
            processed += 1
            
            if processed % 100 == 0:
                print(f"Processed {processed} orders...")

# ─── Async generator for SSE streaming (covered in FastAPI module) ────────────

async def token_stream(prompt: str) -> AsyncIterator[str]:
    """
    Simulates streaming tokens from an LLM.
    Each token is yielded as it arrives.
    """
    tokens = ["The", " shipment", " has", " been", " delivered", " to", " the", " warehouse."]
    for token in tokens:
        await asyncio.sleep(0.05)  # simulate LLM generation time
        yield token

async def collect_stream(prompt: str) -> str:
    """Collect all tokens into a complete response."""
    result = ""
    async for token in token_stream(prompt):
        result += token
    return result
```

---

## Part 10: Complete Real-World Example — Multi-System Shipment Enrichment

This is the kind of code you'll write on day one at a logistics customer.

```python
"""
shipment_enrichment.py

Enriches a shipment record by concurrently fetching data from:
- Carrier API: get current tracking status and ETA
- ERP: get order details and customer info  
- WMS: get warehouse handling instructions
- Geocoding: get lat/lng for delivery address

FDE context: customer had sequential calls taking 3.2s per shipment.
After async refactor: 0.8s per shipment (4x improvement).
"""

import asyncio
import logging
import time
import uuid
from dataclasses import dataclass
from typing import Optional
import httpx
from contextvars import ContextVar

logger = logging.getLogger(__name__)

# ─── Context ──────────────────────────────────────────────────────────────────

request_id_var: ContextVar[str] = ContextVar('request_id', default='')

# ─── Data Models ─────────────────────────────────────────────────────────────

@dataclass
class TrackingInfo:
    status: str
    eta: str
    last_location: str
    carrier: str

@dataclass  
class OrderInfo:
    order_id: str
    customer_name: str
    priority: str
    special_instructions: str

@dataclass
class WarehouseInstructions:
    dock_door: int
    temperature_zone: str
    hazmat_handling: bool

@dataclass
class GeoLocation:
    latitude: float
    longitude: float
    timezone: str

@dataclass
class EnrichedShipment:
    shipment_id: str
    tracking: Optional[TrackingInfo]
    order: Optional[OrderInfo]
    warehouse: Optional[WarehouseInstructions]
    location: Optional[GeoLocation]
    enrichment_time_ms: float
    errors: list[str]

# ─── API Clients ──────────────────────────────────────────────────────────────

async def fetch_tracking_info(
    client: httpx.AsyncClient,
    tracking_number: str
) -> TrackingInfo:
    req_id = request_id_var.get()
    logger.info(f"[{req_id}] Fetching tracking for {tracking_number}")
    
    response = await client.get(
        f"https://carrier-api.internal/track/{tracking_number}",
        headers={"X-Request-ID": req_id}
    )
    response.raise_for_status()
    data = response.json()
    
    return TrackingInfo(
        status=data["status"],
        eta=data["estimated_delivery"],
        last_location=data["last_scan_location"],
        carrier=data["carrier_name"]
    )

async def fetch_order_info(
    client: httpx.AsyncClient,
    order_id: str
) -> OrderInfo:
    req_id = request_id_var.get()
    logger.info(f"[{req_id}] Fetching order {order_id}")
    
    response = await client.get(
        f"https://erp.internal/api/orders/{order_id}",
        headers={"X-Request-ID": req_id}
    )
    response.raise_for_status()
    data = response.json()
    
    return OrderInfo(
        order_id=order_id,
        customer_name=data["customer"]["name"],
        priority=data["priority"],
        special_instructions=data.get("delivery_instructions", "")
    )

async def fetch_warehouse_instructions(
    client: httpx.AsyncClient,
    sku: str
) -> WarehouseInstructions:
    req_id = request_id_var.get()
    
    response = await client.get(
        f"https://wms.internal/api/handling/{sku}",
        headers={"X-Request-ID": req_id}
    )
    response.raise_for_status()
    data = response.json()
    
    return WarehouseInstructions(
        dock_door=data["preferred_dock"],
        temperature_zone=data["storage_zone"],
        hazmat_handling=data["is_hazmat"]
    )

async def geocode_address(
    client: httpx.AsyncClient,
    address: str
) -> GeoLocation:
    response = await client.get(
        "https://geocoding.internal/geocode",
        params={"address": address}
    )
    response.raise_for_status()
    data = response.json()
    
    return GeoLocation(
        latitude=data["lat"],
        longitude=data["lng"],
        timezone=data["timezone"]
    )

# ─── Main Enrichment Function ─────────────────────────────────────────────────

async def enrich_shipment(
    shipment_id: str,
    tracking_number: str,
    order_id: str,
    sku: str,
    delivery_address: str
) -> EnrichedShipment:
    """
    Enriches a shipment by calling 4 systems concurrently.
    
    If any system fails, we return partial data with errors noted.
    We never fail the entire enrichment because one system is down.
    """
    # Set correlation ID for this enrichment request
    req_id = str(uuid.uuid4())[:8]
    token = request_id_var.set(req_id)
    
    start_time = time.perf_counter()
    logger.info(f"[{req_id}] Starting enrichment for shipment {shipment_id}")
    
    try:
        async with httpx.AsyncClient(timeout=httpx.Timeout(5.0, connect=2.0)) as client:
            # All 4 calls run concurrently
            tracking_result, order_result, warehouse_result, geo_result = (
                await asyncio.gather(
                    fetch_tracking_info(client, tracking_number),
                    fetch_order_info(client, order_id),
                    fetch_warehouse_instructions(client, sku),
                    geocode_address(client, delivery_address),
                    return_exceptions=True  # don't cancel siblings on failure
                )
            )
        
        elapsed_ms = (time.perf_counter() - start_time) * 1000
        errors = []
        
        # Extract results, noting any failures
        tracking = None
        if isinstance(tracking_result, Exception):
            errors.append(f"Carrier API failed: {type(tracking_result).__name__}: {tracking_result}")
            logger.warning(f"[{req_id}] Tracking fetch failed: {tracking_result}")
        else:
            tracking = tracking_result
        
        order = None
        if isinstance(order_result, Exception):
            errors.append(f"ERP API failed: {type(order_result).__name__}: {order_result}")
            logger.warning(f"[{req_id}] Order fetch failed: {order_result}")
        else:
            order = order_result
        
        warehouse = None
        if isinstance(warehouse_result, Exception):
            errors.append(f"WMS API failed: {type(warehouse_result).__name__}: {warehouse_result}")
        else:
            warehouse = warehouse_result
        
        location = None
        if isinstance(geo_result, Exception):
            errors.append(f"Geocoding failed: {type(geo_result).__name__}: {geo_result}")
        else:
            location = geo_result
        
        logger.info(
            f"[{req_id}] Enrichment complete for {shipment_id}: "
            f"{elapsed_ms:.0f}ms, {len(errors)} errors"
        )
        
        return EnrichedShipment(
            shipment_id=shipment_id,
            tracking=tracking,
            order=order,
            warehouse=warehouse,
            location=location,
            enrichment_time_ms=elapsed_ms,
            errors=errors
        )
    
    finally:
        request_id_var.reset(token)

# ─── Batch Processing ─────────────────────────────────────────────────────────

async def enrich_shipment_batch(
    shipments: list[dict],
    max_concurrent: int = 10
) -> list[EnrichedShipment]:
    """
    Process multiple shipments concurrently, but with a concurrency limit.
    Without the limit, 1000 shipments would try to make 4000 simultaneous HTTP calls.
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def enrich_with_semaphore(shipment: dict) -> EnrichedShipment:
        async with semaphore:
            return await enrich_shipment(
                shipment_id=shipment["id"],
                tracking_number=shipment["tracking_number"],
                order_id=shipment["order_id"],
                sku=shipment["sku"],
                delivery_address=shipment["delivery_address"]
            )
    
    results = await asyncio.gather(
        *[enrich_with_semaphore(s) for s in shipments],
        return_exceptions=True
    )
    
    return [r for r in results if not isinstance(r, Exception)]

if __name__ == "__main__":
    async def main():
        result = await enrich_shipment(
            shipment_id="SHIP-2024-001",
            tracking_number="1Z999AA1012345678",
            order_id="ORD-78234",
            sku="SKU-WIDGET-L",
            delivery_address="123 Main St, Chicago, IL 60601"
        )
        print(f"Shipment: {result.shipment_id}")
        print(f"Enrichment time: {result.enrichment_time_ms:.0f}ms")
        print(f"Errors: {result.errors or 'none'}")
    
    asyncio.run(main())
```

---

## FDE Context Callouts

> **FDE Context: The 3x Speed Pitch**
>
> When you're in a technical discovery with a customer and they say "our batch job takes 4 hours," your first question should be: "are those API calls sequential?" If yes, you can often reduce it to 45 minutes just by adding `asyncio.gather()`. This is a high-ROI, low-risk change that demonstrates immediate value. Come prepared with the before/after timing code to show them.

> **FDE Context: Concurrency Limits Matter**
>
> Don't let enthusiasm for async lead you to hammer a customer's API. If you fire 10,000 concurrent requests at their ERP, you'll either get rate-limited (429), exhaust their connection pool, or both. Always use `asyncio.Semaphore` to limit concurrency. Standard starting point: 10 concurrent requests, then tune up based on what the API can handle.

> **FDE Context: When Async Isn't the Answer**
>
> If a customer's bottleneck is a 30-second database query, making it async doesn't help — you're still waiting 30 seconds. Async helps with I/O concurrency, not I/O speed. Know the difference before proposing async as a solution.

---

## Common Failure Modes

| Symptom | Likely Cause | Diagnostic |
|---|---|---|
| Service handles 1 req/s instead of 50 | Blocking call inside `async def` | `loop.set_debug(True)`, `slow_callback_duration=0.05` |
| All requests freeze when one is slow | Not using `asyncio.gather()` | Trace concurrent coroutine count |
| `RuntimeError: This event loop is already running` | `asyncio.run()` inside FastAPI/Jupyter | Use `await` directly or `asyncio.create_task()` |
| `RuntimeWarning: coroutine was never awaited` | Forgot `await` keyword | Add `filterwarnings("error")` in tests |
| Task results lost silently | Uncaught exception in fire-and-forget task | Add `done_callback` to log errors |
| Memory leak under load | Tasks accumulating in `_background_tasks` set | Ensure `discard` callback fires on completion |
| `TimeoutError` after exactly 30s | `httpx.AsyncClient` default timeout | Set explicit `timeout=httpx.Timeout(connect=2, read=10, total=30)` |

---

## Interview Angle

**Q: "Explain how Python's GIL affects async programming."**

Great answer: "The GIL has almost no impact on async code because async runs in a single thread — there's nothing to contend for. The GIL becomes relevant when you're using threads, not coroutines. For I/O-bound work, asyncio sidesteps the GIL entirely. For CPU-bound work, you'd use `ProcessPoolExecutor` to get true parallelism with separate GIL instances per process."

**Q: "What's the difference between `asyncio.gather()` and `asyncio.create_task()`?"**

Great answer: "`gather()` runs coroutines concurrently and returns results in the same order as inputs — you get all results at once when they all complete. `create_task()` schedules a coroutine to start immediately but gives you a Task object you can await later or ignore. I use `gather()` when I need all results before proceeding, and `create_task()` for fire-and-forget background work like audit logging or metrics emission."

**Q: "How would you handle one of three concurrent API calls failing?"**

Great answer: "I'd use `asyncio.gather(..., return_exceptions=True)`. Without that flag, any single failure cancels the other tasks and you lose all results. With it, exceptions are returned as values and you inspect each result individually — if it's an `Exception` instance, handle it; otherwise use the result. This gives you partial success, which is almost always better than total failure in a data pipeline."

**Q: "Your async endpoint is slow. Walk me through how you'd diagnose it."**

Great answer: "First, check if I have any blocking calls inside `async def` — that's the most common cause. Enable `asyncio.set_debug(True)` with a `slow_callback_duration` of 50ms; it logs a warning whenever any callback takes longer than that. Second, check if I'm properly using `gather()` for concurrent calls or if I'm awaiting them sequentially. Third, check for N+1 patterns — making one API call inside a loop instead of batching. Fourth, look at connection pool exhaustion — if all 20 connections are in use and a new request arrives, it waits."

---

## Practice Exercise

**Scenario:** You're integrating with a 3PL customer who has three separate APIs:
- `GET /carriers/{id}/rates` — returns shipping rates (avg response: 600ms)
- `GET /addresses/{id}/validate` — validates delivery address (avg response: 200ms)  
- `GET /inventory/{sku}/availability` — checks stock (avg response: 400ms)

**Task 1:** Write an `async def enrich_quote_request(carrier_id, address_id, sku)` function that calls all three APIs concurrently and returns a combined result. Handle partial failures gracefully.

**Task 2:** Add a semaphore so that if you're processing 100 quotes in a batch, you never have more than 15 concurrent enrichments running.

**Task 3:** Add correlation ID tracking using `ContextVar` so every log message includes the quote request ID.

**Task 4:** One of the APIs is a legacy SOAP service wrapped in a sync Python library. Show how you'd call it without blocking the event loop.

**Task 5:** Write a test using `pytest-asyncio` and `respx` (httpx mocker) that verifies: when the carrier rates API times out, the response still includes valid address validation and inventory data.

**Expected output for Task 1:**
```python
result = await enrich_quote_request("FDX-001", "ADDR-789", "SKU-WIDGET-M")
# {
#   "carrier_rates": {"standard": 12.50, "express": 28.00},  # or None if failed
#   "address_valid": True,                                    # or None if failed  
#   "in_stock": True,                                         # or None if failed
#   "errors": []                                              # list any failures
# }
```

---

*Next: [02-pydantic-type-hints.md](./02-pydantic-type-hints.md) — Schema validation at customer API boundaries*
