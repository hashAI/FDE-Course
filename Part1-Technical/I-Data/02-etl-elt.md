# ETL and ELT Patterns for Forward Deployment Engineers

**Course**: FDE Technical Depth  
**Section**: Part 1 — Data  
**File**: 02 of 02  
**Related**: [01-sql-fluency.md](./01-sql-fluency.md) | [../../Part2-Project/00-overview.md](../../Part2-Project/00-overview.md) | [../../Part2-Project/03-llm-agent.md](../../Part2-Project/03-llm-agent.md)

---

## Why FDEs Need to Understand ETL/ELT

You will not build Airbnb's data warehouse. But you will absolutely be asked:

- "Can your integration feed our data warehouse?"
- "We need raw event data in Snowflake by 6 AM every day. How does that work?"
- "Our data team wants to run their own transforms on top of your synced data. What format does it come in?"
- "We're using dbt. Can we model your data in our warehouse?"

If you stare blankly when a customer's data engineer asks "is this ETL or ELT?", you've lost technical credibility in the room. If you can speak intelligently about the patterns — when to transform before loading, what makes a pipeline idempotent, what happens when the source schema changes — you sound like a peer, not a vendor.

The other reason: **the integration product you're deploying IS an ETL/ELT pipeline**. Orders flow from Shopify (extract) → get transformed (map fields, validate, enrich) → loaded into a 3PL system. Understanding ETL/ELT makes you a better integration engineer.

> **FDE Context**: At logistics/supply chain companies, data warehousing conversations come up constantly. Customers are analyzing freight costs, carrier performance, fulfillment rates. They want your integration data in their analytics stack. Being fluent in ETL/ELT concepts lets you scope those conversations accurately and identify when it's a product feature request vs. a custom project.

---

## Part 1: ETL vs ELT — The Core Distinction

### ETL: Extract, Transform, Load

**The traditional pattern.** Data is extracted from the source, transformed (cleaned, shaped, enriched) in an intermediate processing layer, then loaded into the destination in its final form.

```
Source System → [Extract] → Raw Data → [Transform] → Clean Data → [Load] → Target System
                (API/DB)              (compute layer)            (Data Warehouse)
```

**Who does the compute**: The integration/ETL layer (a Python process, Spark cluster, etc.). The data warehouse receives clean, final data.

**When ETL is right**:
- You need to apply complex business logic during transformation (not available in SQL)
- You're masking PII before it enters the warehouse (GDPR compliance — you can't let raw PII touch the warehouse at all)
- The destination is not a data warehouse with processing capability (e.g., loading into a transactional database, a 3PL API, a legacy system)
- Source data quality is so bad you need significant cleanup before it's usable at all
- The transformation involves external API calls (e.g., address validation, currency conversion with live rates)

### ELT: Extract, Load, Transform

**The modern pattern.** Data is extracted and loaded raw into the warehouse (or data lake), then transformed using the warehouse's own compute (SQL).

```
Source System → [Extract] → [Load Raw] → Data Warehouse → [Transform with SQL/dbt] → Modeled Tables
                (API/DB)               (raw tables)     (warehouse compute)
```

**Who does the compute**: The data warehouse (Snowflake, BigQuery, Redshift, DuckDB). SQL runs in the warehouse.

**When ELT is right**:
- You have a powerful cloud data warehouse (Snowflake, BigQuery) — use its compute
- You want to keep raw data for reprocessing (the raw data is the source of truth)
- Your team knows SQL better than Python — dbt lets data analysts own the transforms
- Schema evolution is expected — you load everything raw and adapt transforms without re-extracting
- You want to be able to re-derive business metrics from raw events (event sourcing philosophy)

### The Practical Difference in Logistics

```
Scenario: Order data from Shopify needs to go into the customer's analytics warehouse.

ETL approach:
1. Pull orders from Shopify API (extract)
2. Your Python code: normalize addresses, convert currency to USD, join with
   customer master data, calculate shipping zones, mask PII fields (transform)
3. Load the clean, enriched records into the warehouse orders_dim table (load)

Problem: The warehouse only ever sees what your code decided to give it.
If business logic changes, you need to re-run the entire pipeline from scratch.

ELT approach:
1. Pull orders from Shopify API (extract)
2. Load raw JSON blobs into warehouse table: raw_shopify_orders (load)
3. dbt models in the warehouse transform raw into clean: orders, order_lines, etc.
4. Data team can change dbt models without touching your pipeline.

Advantage: Raw data is always preserved. Business logic is versioned in git with dbt.
```

---

## Part 2: ETL Pipeline Patterns

### The Extract Phase

**From REST API:**

```python
import httpx
import asyncio
from datetime import datetime, timezone
from typing import Iterator, Any
import time

class ShopifyExtractor:
    """
    Extract orders from a paginated REST API.
    Handles: pagination, rate limiting, incremental extraction.
    """
    
    def __init__(self, base_url: str, api_key: str, shop_id: str):
        self.base_url = base_url
        self.api_key = api_key
        self.shop_id = shop_id
        self.client = httpx.Client(
            base_url=base_url,
            headers={"X-API-Key": api_key},
            timeout=30.0,
        )
    
    def extract_orders(
        self, 
        updated_since: datetime | None = None,
        page_size: int = 250,
    ) -> Iterator[dict]:
        """
        Yield orders one page at a time.
        
        Updated_since enables incremental extraction:
        only fetch records modified after the last successful run.
        """
        cursor = None
        page_num = 0
        total_extracted = 0
        
        while True:
            params = {"limit": page_size}
            if updated_since:
                params["updated_since"] = updated_since.isoformat()
            if cursor:
                params["cursor"] = cursor
            
            try:
                response = self.client.get("/orders", params=params)
                response.raise_for_status()
            except httpx.HTTPStatusError as e:
                if e.response.status_code == 429:  # Rate limited
                    retry_after = int(e.response.headers.get("Retry-After", 60))
                    print(f"Rate limited. Waiting {retry_after}s...")
                    time.sleep(retry_after)
                    continue  # Retry same page
                raise
            
            data = response.json()
            orders = data.get("orders", [])
            
            if not orders:
                break   # No more data
            
            for order in orders:
                yield order
                total_extracted += 1
            
            # Check for next page cursor (Link header or response body)
            cursor = data.get("next_cursor")  # or parse Link header
            if not cursor:
                break
            
            page_num += 1
            print(f"Extracted page {page_num}, total: {total_extracted}")
        
        print(f"Extraction complete. Total orders: {total_extracted}")
    
    def close(self):
        self.client.close()


# From database (using SQLAlchemy)
from sqlalchemy import create_engine, text
from typing import Generator

def extract_from_source_db(
    conn_str: str,
    table: str,
    watermark_column: str,
    watermark_value: datetime,
    batch_size: int = 1000,
) -> Generator[list[dict], None, None]:
    """
    Extract rows from a source database in batches.
    Uses a watermark (e.g., updated_at) for incremental extraction.
    """
    engine = create_engine(conn_str)
    
    query = text(f"""
        SELECT *
        FROM {table}
        WHERE {watermark_column} > :watermark
        ORDER BY {watermark_column}, id
        LIMIT {batch_size} OFFSET :offset
    """)
    
    offset = 0
    with engine.connect() as conn:
        while True:
            result = conn.execute(query, {
                "watermark": watermark_value,
                "offset": offset
            })
            batch = [dict(row._mapping) for row in result]
            
            if not batch:
                break
            
            yield batch
            offset += batch_size


# From file (CSV, NDJSON)
import csv
import json
from pathlib import Path

def extract_from_csv(file_path: str) -> Iterator[dict]:
    """Extract rows from a CSV file."""
    with open(file_path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield dict(row)

def extract_from_ndjson(file_path: str) -> Iterator[dict]:
    """Extract records from newline-delimited JSON."""
    with open(file_path, encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                yield json.loads(line)
```

### The Transform Phase

```python
from pydantic import BaseModel, validator, Field
from typing import Optional
from datetime import datetime
import re


# Define target schema with Pydantic
class NormalizedOrder(BaseModel):
    """The clean, normalized form of an order."""
    external_id: str
    customer_email: str
    customer_name: str
    
    # Shipping address (normalized)
    shipping_street: str
    shipping_city: str
    shipping_state: str
    shipping_postal_code: str
    shipping_country_code: str  # ISO 3166-1 alpha-2 (2-char code)
    
    # Financials (always in USD)
    total_usd: float
    subtotal_usd: float
    tax_usd: float
    shipping_cost_usd: float
    
    line_items: list[dict]
    source_currency: str
    created_at: datetime
    
    # Derived fields
    is_international: bool
    total_weight_kg: float


class OrderTransformer:
    """
    Transform raw Shopify orders into NormalizedOrder.
    Handles: field mapping, currency conversion, address normalization, validation.
    """
    
    # Shopify→ISO country code mapping (partial)
    COUNTRY_MAP = {
        "United States": "US",
        "Canada": "CA", 
        "United Kingdom": "GB",
        "Australia": "AU",
        "Germany": "DE",
        "France": "FR",
        "Japan": "JP",
    }
    
    def __init__(self, currency_rates: dict[str, float] | None = None):
        # currency_rates: {"EUR": 1.08, "GBP": 1.27, ...} (to USD)
        self.currency_rates = currency_rates or {"USD": 1.0}
    
    def transform(self, raw_order: dict) -> NormalizedOrder | None:
        """
        Transform a raw Shopify order. Returns None if the order should be skipped.
        Raises ValueError if the order is malformed and can't be transformed.
        """
        # Skip cancelled orders
        if raw_order.get("financial_status") == "refunded":
            return None
        
        # Extract shipping address
        shipping = raw_order.get("shipping_address", {})
        if not shipping:
            # Some digital orders have no shipping address
            return None
        
        # Normalize country code
        raw_country = shipping.get("country", "")
        country_code = self._normalize_country(raw_country)
        
        # Convert currency to USD
        currency = raw_order.get("currency", "USD")
        rate = self.currency_rates.get(currency, 1.0)
        
        def to_usd(value: str | float | None) -> float:
            if value is None:
                return 0.0
            return round(float(value) * rate, 2)
        
        # Extract line items
        line_items = []
        total_weight_kg = 0.0
        for item in raw_order.get("line_items", []):
            weight_kg = self._normalize_weight(
                item.get("grams", 0), source_unit="grams"
            )
            total_weight_kg += weight_kg * item.get("quantity", 1)
            line_items.append({
                "sku": item.get("sku") or f"SKU-{item.get('product_id')}",
                "title": item.get("title", ""),
                "quantity": item.get("quantity", 1),
                "unit_price_usd": to_usd(item.get("price")),
                "weight_kg": weight_kg,
                "hs_code": item.get("harmonized_system_code"),
            })
        
        return NormalizedOrder(
            external_id=str(raw_order["id"]),
            customer_email=raw_order.get("email", "").lower().strip(),
            customer_name=self._extract_customer_name(raw_order),
            shipping_street=self._clean_address_line(shipping.get("address1", "")),
            shipping_city=shipping.get("city", "").strip(),
            shipping_state=shipping.get("province_code") or shipping.get("province", ""),
            shipping_postal_code=self._normalize_postal_code(
                shipping.get("zip", ""), country_code
            ),
            shipping_country_code=country_code,
            total_usd=to_usd(raw_order.get("total_price")),
            subtotal_usd=to_usd(raw_order.get("subtotal_price")),
            tax_usd=to_usd(raw_order.get("total_tax")),
            shipping_cost_usd=to_usd(raw_order.get("total_shipping_price_set", {})
                                     .get("shop_money", {}).get("amount")),
            line_items=line_items,
            source_currency=currency,
            created_at=datetime.fromisoformat(
                raw_order["created_at"].replace("Z", "+00:00")
            ),
            is_international=country_code != "US",
            total_weight_kg=round(total_weight_kg, 3),
        )
    
    def _normalize_country(self, raw_country: str) -> str:
        """Convert country name or code to ISO 3166-1 alpha-2."""
        raw = raw_country.strip()
        if len(raw) == 2:
            return raw.upper()
        return self.COUNTRY_MAP.get(raw, raw[:2].upper())
    
    def _normalize_weight(self, value: float, source_unit: str) -> float:
        """Convert weight to kg."""
        if source_unit == "grams":
            return value / 1000
        elif source_unit == "lbs":
            return value * 0.453592
        elif source_unit == "oz":
            return value * 0.0283495
        return value  # assume kg
    
    def _normalize_postal_code(self, raw: str, country: str) -> str:
        """Normalize postal code format."""
        raw = raw.strip().upper().replace(" ", "")
        if country == "US":
            # Ensure 5 digits (strip +4)
            digits = re.sub(r"[^0-9]", "", raw)
            return digits[:5]
        elif country == "CA":
            # Canadian format: A1A 1A1
            raw = re.sub(r"[^A-Z0-9]", "", raw)
            if len(raw) == 6:
                return f"{raw[:3]} {raw[3:]}"
        return raw
    
    def _clean_address_line(self, addr: str) -> str:
        return " ".join(addr.strip().split())
    
    def _extract_customer_name(self, order: dict) -> str:
        """Try multiple fields for customer name."""
        # Try shipping address first
        shipping = order.get("shipping_address", {})
        if shipping.get("name"):
            return shipping["name"].strip()
        # Fall back to first_name + last_name
        first = order.get("customer", {}).get("first_name", "")
        last = order.get("customer", {}).get("last_name", "")
        full = f"{first} {last}".strip()
        if full:
            return full
        return "Unknown Customer"


# Transform with validation and error collection
def transform_batch(
    raw_orders: list[dict],
    transformer: OrderTransformer,
) -> tuple[list[NormalizedOrder], list[dict]]:
    """
    Transform a batch of orders.
    Returns: (successful_orders, failed_records_with_errors)
    """
    successful = []
    failed = []
    
    for raw in raw_orders:
        try:
            result = transformer.transform(raw)
            if result is not None:
                successful.append(result)
        except Exception as e:
            failed.append({
                "raw_order": raw,
                "error": str(e),
                "error_type": type(e).__name__,
                "order_id": raw.get("id"),
            })
    
    return successful, failed
```

### The Load Phase

```python
from sqlalchemy import create_engine
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.orm import Session
import json

def load_to_database(
    orders: list[NormalizedOrder],
    session: Session,
    table_name: str = "normalized_orders",
) -> dict:
    """
    Load normalized orders into PostgreSQL.
    Uses UPSERT for idempotency: re-running loads the same data without duplicates.
    """
    if not orders:
        return {"inserted": 0, "updated": 0}
    
    # Convert Pydantic models to dicts
    records = []
    for order in orders:
        record = order.model_dump()
        record["line_items"] = json.dumps(record["line_items"])  # serialize JSON
        records.append(record)
    
    # Bulk upsert using PostgreSQL's ON CONFLICT
    stmt = pg_insert(NormalizedOrderTable).values(records)
    stmt = stmt.on_conflict_do_update(
        index_elements=["external_id"],
        set_={
            "customer_email": stmt.excluded.customer_email,
            "total_usd": stmt.excluded.total_usd,
            "shipping_country_code": stmt.excluded.shipping_country_code,
            "line_items": stmt.excluded.line_items,
            # NOTE: updated_at is set automatically by DB trigger
        }
    )
    
    result = session.execute(stmt)
    session.commit()
    
    return {
        "inserted": result.rowcount,  # Note: this isn't perfect with upserts
        "records": len(records),
    }


def load_to_data_warehouse(
    orders: list[NormalizedOrder],
    snowflake_conn_str: str,
) -> None:
    """
    Load to Snowflake data warehouse.
    ELT pattern: write as raw JSON to a staging table, let dbt model it.
    """
    import snowflake.connector  # pip install snowflake-connector-python
    
    conn = snowflake.connector.connect(
        connection_string=snowflake_conn_str
    )
    cursor = conn.cursor()
    
    # Write raw records as VARIANT (Snowflake's JSON type)
    # This is the ELT pattern: load raw, transform later with dbt
    cursor.executemany(
        """
        INSERT INTO RAW.SHOPIFY_ORDERS (
            external_id, raw_payload, loaded_at
        ) VALUES (%s, PARSE_JSON(%s), CURRENT_TIMESTAMP())
        """,
        [
            (order.external_id, json.dumps(order.model_dump()))
            for order in orders
        ]
    )
    conn.commit()
    cursor.close()
    conn.close()


# Append vs Upsert vs Replace
def load_append(records: list[dict], session: Session) -> None:
    """
    Append-only: always insert new rows.
    Use for: event logs, audit tables, tracking events.
    Never for: master data that can change (orders, customers).
    """
    session.execute(TrackingEventTable.__table__.insert(), records)
    session.commit()

def load_replace(records: list[dict], session: Session, date_partition: str) -> None:
    """
    Delete-then-insert for a date partition.
    Use for: daily snapshots, when you need clean replacement.
    Danger: non-atomic (brief window where data is missing).
    """
    session.execute(
        text("DELETE FROM daily_order_summary WHERE report_date = :date"),
        {"date": date_partition}
    )
    session.execute(DailyOrderSummary.__table__.insert(), records)
    session.commit()
```

---

## Part 3: Incremental vs Full Load

This is one of the most important design decisions in any ETL pipeline.

### Full Load

Re-extract and re-load everything, every run.

```python
class FullLoadPipeline:
    """
    Full load: extract all data, truncate target, reload everything.
    
    Pros: Simple, always produces consistent state.
    Cons: Slow, expensive, puts heavy load on source system.
    When to use: small datasets (<100K rows), tables that don't have a
                 reliable watermark, initial historical backfill.
    """
    
    def run(self, source: ShopifyExtractor, target: Session) -> dict:
        print("Starting full load...")
        
        # Extract everything
        all_orders = list(source.extract_orders())
        print(f"Extracted {len(all_orders)} orders from source")
        
        # Transform
        transformer = OrderTransformer()
        normalized, failed = transform_batch(all_orders, transformer)
        print(f"Transformed: {len(normalized)} success, {len(failed)} failed")
        
        # Replace target (truncate + insert in a transaction)
        with target.begin():
            target.execute(text("TRUNCATE TABLE normalized_orders"))
            load_to_database(normalized, target)
        
        return {"extracted": len(all_orders), "loaded": len(normalized), "failed": len(failed)}
```

### Incremental Load

Only process new or modified records since the last run.

```python
import json
from pathlib import Path
from datetime import datetime, timezone

class IncrementalPipeline:
    """
    Incremental load: only process records newer than the watermark.
    
    Watermark: a value (usually a timestamp or sequence ID) that marks
    the highest point we've processed. Stored in a checkpoint file or DB table.
    
    Pros: Fast, low source system load.
    Cons: Misses records with backdated updates.
          Requires a reliable watermark column (updated_at that always changes).
    """
    
    def __init__(self, checkpoint_path: str = "checkpoint.json"):
        self.checkpoint_path = Path(checkpoint_path)
    
    def load_checkpoint(self) -> datetime | None:
        """Load the last successful run watermark."""
        if not self.checkpoint_path.exists():
            return None
        data = json.loads(self.checkpoint_path.read_text())
        ts_str = data.get("last_watermark")
        if ts_str:
            return datetime.fromisoformat(ts_str)
        return None
    
    def save_checkpoint(self, watermark: datetime) -> None:
        """Save the watermark for the next run."""
        self.checkpoint_path.write_text(
            json.dumps({"last_watermark": watermark.isoformat(), "updated_at": datetime.now(timezone.utc).isoformat()})
        )
    
    def run(
        self, 
        source: ShopifyExtractor, 
        target: Session,
    ) -> dict:
        # Load last watermark
        last_watermark = self.load_checkpoint()
        run_start = datetime.now(timezone.utc)
        
        print(f"Incremental load since: {last_watermark or 'beginning'}")
        
        # Extract only new/modified records
        raw_orders = list(source.extract_orders(updated_since=last_watermark))
        print(f"Extracted {len(raw_orders)} orders")
        
        if not raw_orders:
            print("No new orders to process")
            return {"extracted": 0, "loaded": 0}
        
        # Transform
        transformer = OrderTransformer()
        normalized, failed = transform_batch(raw_orders, transformer)
        
        # Load (upsert — idempotent)
        result = load_to_database(normalized, target)
        
        # Save checkpoint: use run_start (not max updated_at from data)
        # Why run_start? If there's a record updated at 12:00:01 and our extract
        # started at 12:00:00, we want to catch it next time even if we save
        # 12:00:00 as the watermark. Using run_start with a small overlap is safer.
        self.save_checkpoint(run_start)
        
        return {
            "extracted": len(raw_orders),
            "loaded": result["inserted"],
            "failed": len(failed),
            "watermark": run_start.isoformat(),
        }
```

---

## Part 4: Idempotency — The Most Important Pipeline Property

An idempotent pipeline produces the same result no matter how many times it runs on the same input. This is non-negotiable for production pipelines.

```python
class IdempotentPipeline:
    """
    Demonstrates idempotency patterns:
    1. Upsert (not insert) for the load phase
    2. Deduplication before loading
    3. Checkpoint-based incremental extraction
    4. Transactional batches with rollback on failure
    """
    
    def run_batch_idempotently(
        self,
        orders: list[NormalizedOrder],
        session: Session,
        batch_id: str,   # unique ID for this batch run
    ) -> dict:
        """
        Run a load batch idempotently.
        If the batch was already loaded (same batch_id), skip it.
        """
        # Check if this batch was already processed
        existing = session.execute(
            text("SELECT id FROM pipeline_batches WHERE batch_id = :bid"),
            {"bid": batch_id}
        ).scalar()
        
        if existing:
            print(f"Batch {batch_id} already processed. Skipping.")
            return {"status": "skipped", "batch_id": batch_id}
        
        # Deduplicate within the batch (if source sends duplicates)
        seen_ids = set()
        unique_orders = []
        for order in orders:
            if order.external_id not in seen_ids:
                seen_ids.add(order.external_id)
                unique_orders.append(order)
        
        # Load in a single transaction
        try:
            with session.begin():
                # Upsert orders
                load_to_database(unique_orders, session)
                
                # Record that this batch was processed
                session.execute(
                    text("""
                        INSERT INTO pipeline_batches (batch_id, record_count, processed_at)
                        VALUES (:bid, :count, NOW())
                    """),
                    {"bid": batch_id, "count": len(unique_orders)}
                )
        except Exception as e:
            # Transaction automatically rolled back on exception
            print(f"Batch {batch_id} failed: {e}")
            raise
        
        return {"status": "success", "batch_id": batch_id, "records": len(unique_orders)}
```

---

## Part 5: Schema Evolution — What Happens When Things Change

This is where pipelines break in production.

```python
from typing import Any

class SchemaEvolutionHandler:
    """
    Handle changes to the source schema without breaking the pipeline.
    
    Common schema changes:
    - New field added to source (backward compatible)
    - Field renamed in source (breaking)
    - Field removed from source (may break if required)
    - Field type changed (breaking)
    - Nested structure changed (breaking)
    """
    
    # Field mapping: handles renames
    FIELD_MAP = {
        "order_number": "external_id",     # old name → new name
        "total_price": "total_value",
        "buyer_email": "customer_email",
    }
    
    def extract_field_safely(
        self, 
        record: dict, 
        field: str, 
        default: Any = None,
        alternatives: list[str] | None = None,
    ) -> Any:
        """
        Extract a field with fallbacks for renamed fields.
        
        >>> handler.extract_field_safely(order, "external_id", alternatives=["order_number", "id"])
        """
        # Try main field name
        if field in record:
            return record[field]
        
        # Try alternatives (handles renames)
        for alt in (alternatives or []):
            if alt in record:
                return record[alt]
        
        # Apply field map
        mapped = self.FIELD_MAP.get(field)
        if mapped and mapped in record:
            return record[mapped]
        
        return default
    
    def validate_schema(self, record: dict, required_fields: list[str]) -> list[str]:
        """
        Validate that a record has required fields.
        Returns list of missing fields (empty list = valid).
        """
        missing = []
        for field in required_fields:
            value = self.extract_field_safely(record, field)
            if value is None:
                missing.append(field)
        return missing
    
    def add_forward_compatibility(self, record: dict) -> dict:
        """
        Add default values for new required fields that older records don't have.
        This lets you add columns to the target schema without breaking old records.
        """
        defaults = {
            "source_platform": "shopify",
            "api_version": "2024-01",
            "is_b2b": False,
            "warehouse_zone": None,
        }
        for key, default_val in defaults.items():
            if key not in record:
                record[key] = default_val
        return record


# Schema change detection
def detect_schema_drift(
    current_sample: dict,
    reference_schema: dict,
) -> dict:
    """
    Compare a live record's structure to the expected schema.
    Used to alert when the source API changes its response shape.
    
    Returns: {
        'new_fields': [...],   # fields in current but not in reference
        'missing_fields': [...], # fields in reference but not in current
        'type_changes': {...}   # fields where type changed
    }
    """
    current_keys = set(current_sample.keys())
    reference_keys = set(reference_schema.keys())
    
    new_fields = current_keys - reference_keys
    missing_fields = reference_keys - current_keys
    
    type_changes = {}
    for key in current_keys & reference_keys:
        expected_type = reference_schema[key]
        actual_type = type(current_sample[key]).__name__
        if actual_type != expected_type:
            type_changes[key] = {
                "expected": expected_type,
                "actual": actual_type,
            }
    
    return {
        "new_fields": list(new_fields),
        "missing_fields": list(missing_fields),
        "type_changes": type_changes,
        "drift_detected": bool(new_fields or missing_fields or type_changes),
    }
```

---

## Part 6: Backfill — Reprocessing Historical Data

When something goes wrong (or you add a new field), you need to reprocess historical data.

```python
import asyncio
from datetime import datetime, timedelta

class BackfillPipeline:
    """
    Backfill: reprocess historical data for a date range.
    
    Use cases:
    - Bug fix: incorrect transformation logic was applied for 2 weeks
    - New field: added a new enrichment field, backfill it for all historical records
    - Data loss: target database corrupted for a date range, re-extract from source
    """
    
    def __init__(
        self, 
        extractor: ShopifyExtractor,
        transformer: OrderTransformer,
        session: Session,
        batch_size: int = 500,
    ):
        self.extractor = extractor
        self.transformer = transformer
        self.session = session
        self.batch_size = batch_size
    
    async def backfill_date_range(
        self,
        start_date: datetime,
        end_date: datetime,
        dry_run: bool = False,
    ) -> dict:
        """
        Backfill all orders in a date range.
        
        dry_run=True: extract and transform but don't write to DB.
        Useful for validating the backfill before committing.
        """
        print(f"Starting backfill: {start_date.date()} to {end_date.date()}")
        print(f"Dry run: {dry_run}")
        
        stats = {
            "extracted": 0,
            "transformed": 0,
            "failed": 0,
            "loaded": 0,
            "dry_run": dry_run,
        }
        
        # Process day by day to avoid memory issues
        current_date = start_date
        while current_date < end_date:
            next_date = current_date + timedelta(days=1)
            
            # Extract orders for this day
            day_orders = list(self.extractor.extract_orders(
                updated_since=current_date,
                # In real API: also filter by created_before=next_date
            ))
            
            stats["extracted"] += len(day_orders)
            
            # Process in batches
            for i in range(0, len(day_orders), self.batch_size):
                batch = day_orders[i:i + self.batch_size]
                normalized, failed = transform_batch(batch, self.transformer)
                
                stats["transformed"] += len(normalized)
                stats["failed"] += len(failed)
                
                if not dry_run:
                    result = load_to_database(normalized, self.session)
                    stats["loaded"] += result.get("inserted", 0)
            
            print(f"  {current_date.date()}: {len(day_orders)} orders")
            current_date = next_date
        
        print(f"Backfill complete: {stats}")
        return stats
```

---

## Part 7: Common Tools — Awareness Level

You don't need to be an expert in these tools, but you need to speak intelligently when customers mention them.

### dbt (data build tool)

dbt is a SQL-first transformation tool that runs inside your data warehouse. You write SELECT statements, dbt wraps them as tables or views, handles dependencies, and provides testing.

```sql
-- A dbt model is just a SELECT statement in a .sql file
-- models/orders_clean.sql

{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns'
) }}

SELECT
    id AS order_id,
    external_id,
    created_at,
    updated_at,
    status,
    UPPER(shipping_country) AS country_code,
    total_price::NUMERIC / 100 AS total_usd,   -- Shopify sends cents
    -- Derived fields
    CASE 
        WHEN shipping_country != 'US' THEN TRUE 
        ELSE FALSE 
    END AS is_international,
    _loaded_at                                  -- added by the ELT loader
FROM {{ source('raw', 'shopify_orders') }}

{% if is_incremental() %}
  -- Only process new/updated records on incremental runs
  WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

```python
# What FDEs need to know about dbt:
# 1. dbt models are SELECT statements. The tool manages CREATE TABLE/VIEW.
# 2. materialized: 'table' rebuilds the table, 'view' creates a view,
#    'incremental' only processes new rows.
# 3. {{ source() }} = references raw tables from the ELT loader
# 4. {{ ref() }} = references another dbt model (handles dependency ordering)
# 5. dbt test: can validate that order_id is unique, not null, in a set of values
# 6. dbt run --select orders_clean: runs just this one model

# FDE answer when asked "does your integration work with dbt?":
# "Yes. Our integration loads raw order data into your warehouse's raw schema.
# Your dbt models can reference that source table. We document the exact schema
# and update frequency so your data team can build models on top of it."
```

### Apache Airflow

Airflow is a workflow orchestrator. You write DAGs (Directed Acyclic Graphs) that define task dependencies and schedules.

```python
# A simple Airflow DAG for a nightly order sync
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    "owner": "data-engineering",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": True,
    "email": ["alerts@company.com"],
}

with DAG(
    dag_id="order_sync_nightly",
    default_args=default_args,
    schedule_interval="0 2 * * *",   # 2 AM daily
    start_date=datetime(2024, 1, 1),
    catchup=False,   # Don't backfill missed runs
    tags=["orders", "sync"],
) as dag:
    
    # Task 1: Extract from Shopify
    extract_task = PythonOperator(
        task_id="extract_shopify_orders",
        python_callable=run_extraction,
        op_kwargs={"updated_since_hours": 25},  # 25h window for overlap safety
    )
    
    # Task 2: Transform
    transform_task = PythonOperator(
        task_id="transform_orders",
        python_callable=run_transformation,
    )
    
    # Task 3: Load to warehouse
    load_task = PythonOperator(
        task_id="load_to_warehouse",
        python_callable=run_load,
    )
    
    # Task 4: Run dbt to update analytics models
    dbt_run = BashOperator(
        task_id="dbt_run_orders",
        bash_command="cd /opt/dbt && dbt run --select orders_clean orders_daily_summary",
    )
    
    # Task 5: Data quality check
    dbt_test = BashOperator(
        task_id="dbt_test_orders",
        bash_command="cd /opt/dbt && dbt test --select orders_clean",
    )
    
    # Define dependencies (task graph)
    extract_task >> transform_task >> load_task >> dbt_run >> dbt_test

# What FDEs need to know about Airflow:
# - DAG: the pipeline. Task: a single step. Edge: dependency.
# - Airflow handles: scheduling, retries, alerts, monitoring, backfills.
# - The scheduler runs tasks when their dependencies are met.
# - XCom: how tasks pass data to each other (small data only — not DataFrames).
# - Variables & Connections: where Airflow stores secrets and config.
```

### Airbyte

Airbyte is a managed ELT connector platform. It extracts from 300+ sources and loads to your warehouse.

```python
# FDE context for Airbyte:
# - Airbyte handles the Extract + Load phases for ELT
# - You configure a "source" (Shopify, Postgres, S3) and "destination" (Snowflake, BigQuery)
# - Airbyte handles pagination, auth, incremental sync, schema mapping
# - Your integration product might COMPETE with Airbyte, or COMPLEMENT it

# Common customer question: "Can I use Airbyte to sync your data?"
# Answer depends on whether you have an Airbyte connector.
# If you don't: "Our API follows REST conventions; a custom Airbyte source
# could be built using their CDK in about a day."

# Airbyte connector config (conceptual):
airbyte_source_config = {
    "source_type": "custom_http",
    "api_url": "https://api.logisync.com",
    "auth_type": "api_key",
    "api_key": "{{ secret.LOGISYNC_API_KEY }}",
    "streams": [
        {
            "name": "orders",
            "path": "/v1/orders",
            "primary_key": "id",
            "cursor_field": "updated_at",   # for incremental sync
            "pagination": {
                "type": "cursor",
                "cursor_path": "next_cursor",
                "cursor_param": "cursor",
            }
        }
    ]
}
```

### AWS Glue

AWS Glue is managed ETL on AWS. PySpark-based transformations that run serverlessly.

```python
# Glue is relevant when customers are AWS-heavy
# They might want to extract your data using a Glue job
# Key concepts FDEs need to know:

# 1. Glue Data Catalog: central metadata store (table schemas, locations)
# 2. Glue Job: Python or Spark script that does ETL
# 3. Glue Crawler: automatically infers schema from data in S3 or RDS
# 4. Glue Connection: named connection to a JDBC source (RDS, Redshift, etc.)

# A simple Glue job in PySpark:
"""
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
job = Job(glueContext)

# Read from data catalog (your source: e.g., RDS orders table)
orders_df = glueContext.create_dynamic_frame.from_catalog(
    database="logisync",
    table_name="orders"
)

# Transform: select and rename columns
from awsglue.transforms import SelectFields, RenameField
orders_df = SelectFields.apply(frame=orders_df, paths=["id", "external_id", "status", "created_at"])
orders_df = RenameField.apply(frame=orders_df, path="external_id", newName="order_number")

# Write to S3 as Parquet (for analytics consumption)
glueContext.write_dynamic_frame.from_options(
    frame=orders_df,
    connection_type="s3",
    connection_options={"path": "s3://my-warehouse/orders/"},
    format="parquet",
)

job.commit()
"""

# FDE answer for "can we use Glue to load your data?":
# "Absolutely. If you're on AWS, you can set up a Glue job to read from our
# PostgreSQL database using a JDBC connection, or we can push data to S3
# and your Glue crawler can catalog it automatically."
```

---

## Part 8: A Complete ETL Pipeline in Python

Putting it all together: a production-ready ETL pipeline class.

```python
import logging
import time
from dataclasses import dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import Callable

import structlog

logger = structlog.get_logger(__name__)


@dataclass
class PipelineConfig:
    source_api_url: str
    source_api_key: str
    db_url: str
    batch_size: int = 500
    max_retries: int = 3
    checkpoint_path: str = "pipeline_checkpoint.json"
    dry_run: bool = False


@dataclass
class PipelineRun:
    run_id: str
    started_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    finished_at: datetime | None = None
    status: str = "running"
    
    extracted: int = 0
    transformed: int = 0
    transform_failed: int = 0
    loaded: int = 0
    
    errors: list[str] = field(default_factory=list)
    
    @property
    def duration_seconds(self) -> float:
        if self.finished_at:
            return (self.finished_at - self.started_at).total_seconds()
        return (datetime.now(timezone.utc) - self.started_at).total_seconds()


class OrderSyncPipeline:
    """
    Production ETL pipeline for syncing orders from e-commerce to 3PL.
    
    Features:
    - Incremental extraction with checkpoint
    - Batch processing with configurable size
    - Error collection (fail fast vs. continue on error)
    - Dry run mode
    - Structured logging
    - Retry logic for transient failures
    """
    
    def __init__(self, config: PipelineConfig):
        self.config = config
        self.checkpoint_mgr = IncrementalPipeline(config.checkpoint_path)
        self.extractor = ShopifyExtractor(
            base_url=config.source_api_url,
            api_key=config.source_api_key,
            shop_id="default",
        )
        self.transformer = OrderTransformer()
        # Session setup omitted for brevity
    
    def run(self, run_id: str | None = None) -> PipelineRun:
        """Execute the pipeline. Returns a PipelineRun with stats."""
        run = PipelineRun(run_id=run_id or f"run-{int(time.time())}")
        log = logger.bind(run_id=run.run_id, dry_run=self.config.dry_run)
        
        log.info("pipeline.started")
        
        try:
            # Load watermark
            watermark = self.checkpoint_mgr.load_checkpoint()
            log.info("pipeline.watermark", watermark=watermark)
            
            # Extract
            batch = []
            for raw_order in self.extractor.extract_orders(updated_since=watermark):
                batch.append(raw_order)
                run.extracted += 1
                
                if len(batch) >= self.config.batch_size:
                    self._process_batch(batch, run, log)
                    batch = []
            
            # Process final partial batch
            if batch:
                self._process_batch(batch, run, log)
            
            # Save checkpoint
            if not self.config.dry_run and run.status == "running":
                self.checkpoint_mgr.save_checkpoint(run.started_at)
            
            run.status = "success"
            log.info("pipeline.completed", **{
                "extracted": run.extracted,
                "loaded": run.loaded,
                "failed": run.transform_failed,
                "duration_s": run.duration_seconds,
            })
        
        except Exception as e:
            run.status = "failed"
            run.errors.append(str(e))
            log.error("pipeline.failed", error=str(e), exc_info=True)
            raise
        
        finally:
            run.finished_at = datetime.now(timezone.utc)
        
        return run
    
    def _process_batch(
        self,
        raw_batch: list[dict],
        run: PipelineRun,
        log: structlog.BoundLogger,
    ) -> None:
        """Transform and load a single batch."""
        # Transform
        normalized, failed = transform_batch(raw_batch, self.transformer)
        run.transformed += len(normalized)
        run.transform_failed += len(failed)
        
        if failed:
            log.warning("batch.transform_failures", count=len(failed),
                       sample_errors=[f["error"] for f in failed[:3]])
        
        # Load
        if normalized and not self.config.dry_run:
            load_to_database(normalized, self.session)
            run.loaded += len(normalized)
            log.debug("batch.loaded", count=len(normalized))


# Entry point
if __name__ == "__main__":
    import os
    
    config = PipelineConfig(
        source_api_url=os.getenv("SOURCE_API_URL"),
        source_api_key=os.getenv("SOURCE_API_KEY"),
        db_url=os.getenv("DATABASE_URL"),
        batch_size=int(os.getenv("BATCH_SIZE", "500")),
        dry_run=os.getenv("DRY_RUN", "false").lower() == "true",
    )
    
    pipeline = OrderSyncPipeline(config)
    result = pipeline.run()
    
    print(f"Pipeline {result.status}: {result.extracted} extracted, {result.loaded} loaded")
    if result.status == "failed":
        exit(1)
```

---

## Common Failure Modes

1. **Missing incremental update column**: The source table has no `updated_at` column (or it's not indexed). Full load required forever, pipeline gets slower as data grows.

2. **Late-arriving data**: An order was created 3 days ago but updated 10 minutes ago (customer changed address). Your watermark is yesterday's date. The update is missed. Solution: use a slightly wider window (watermark - 1 hour) and rely on UPSERT idempotency.

3. **Checkpoint saved before load completes**: System crashes after marking the checkpoint but before finishing the load. Next run misses the records. Solution: save checkpoint AFTER successful load, inside a transaction if possible.

4. **Schema drift crashing the pipeline**: Source API added a required field that your Pydantic model doesn't know about, or changed a field type. Solution: schema drift detection, fail gracefully, alert the team.

5. **Fan-out from joins in the extract**: You JOIN two tables to enrich data during extract, and one side has multiple rows. You get duplicated records. Always check row counts before and after joins.

6. **Not handling NULL in watermark queries**: `WHERE updated_at > :watermark` skips rows where `updated_at IS NULL`. Those records are never processed. Solution: `WHERE updated_at > :watermark OR updated_at IS NULL`.

7. **Currency conversion using stale rates**: Fetching exchange rates once at pipeline start and using them for a 3-hour run. Rates change. Solution: cache rates for the pipeline run duration but refresh at the start of each run.

---

## Interview Angle

**Key concepts to demonstrate:**

1. **ETL vs ELT distinction with a concrete example**: "If the customer needs PII masked before it touches the warehouse, that's ETL — we transform before loading. If they want raw order data in Snowflake for their data team to model, that's ELT — we load raw and they transform with dbt."

2. **Idempotency**: "Every pipeline I write uses UPSERT rather than INSERT. That means I can run it twice on the same data and it's safe — the second run just updates records with identical values. This is critical for recovery after failures."

3. **Incremental extraction**: "For a customer with 2 million orders, a full reload every night would take 3 hours and put load on their source system. Incremental extraction with a watermark brings that to under 5 minutes."

4. **Schema evolution handling**: "When a source API version changes its response structure, the pipeline should fail gracefully with a clear error and schema diff, not silently corrupt data."

---

## Practice Exercise

**Scenario**: A manufacturing customer has 500,000 product records in an ERP system (Microsoft Dynamics). They want this product catalog in Snowflake for analytics. Products are updated throughout the day (price changes, inventory updates). New products are added weekly.

Design and partially implement the following:

1. **Pipeline type decision**: Should this be ETL or ELT? Write 3-4 sentences explaining your reasoning.

2. **Extraction strategy**: The ERP exposes a REST API with pagination (`?page=1&per_page=200`) and a `modified_since` parameter. Write the extractor class.

3. **Watermark strategy**: The `modified_at` column exists but is not indexed on the ERP side. Queries on it time out for large date ranges. How would you handle incremental extraction differently?

4. **Idempotency**: The customer's Snowflake warehouse doesn't support `ON CONFLICT`. They're using MERGE statements. Write a Python function that does a MERGE-equivalent load using Snowflake's Python connector.

5. **Schema drift**: The ERP sometimes returns `unit_cost` as a string like `"$24.99"` and sometimes as a float `24.99`. Write a transform function that handles both forms.

---

*Next: [../../Part2-Project/00-overview.md](../../Part2-Project/00-overview.md) — The LogiSync AI capstone project*
