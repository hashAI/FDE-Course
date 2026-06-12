# SQL Fluency for Forward Deployment Engineers

**Course**: FDE Technical Depth  
**Section**: Part 1 — Data  
**File**: 01 of 02  
**Related**: [02-etl-elt.md](./02-etl-elt.md) | [../../../Part2-Project/00-overview.md](../../Part2-Project/00-overview.md)

---

## Why SQL is Non-Negotiable for FDEs

You are not a data engineer. You are not a DBA. But you will be the person in a customer call who pulls up a terminal and answers "why did order #4821 not get synced to our 3PL last night?" in under three minutes.

That requires SQL. Not theoretical knowledge of SQL — **working fluency**. The kind where you can stare at a schema you've never seen before, run a series of queries, and reconstruct the life history of a single record across five tables.

In logistics and supply chain specifically, data lives in relational databases. Orders, line items, shipments, tracking events, carrier bookings — all relational. ERPs (SAP, Oracle, NetSuite) expose data through SQL-queryable views. Warehouse Management Systems (WMS) have PostgreSQL or SQL Server backends. When something goes wrong, SQL is how you prove it.

> **FDE Context**: At every logistics software company, there will be a customer escalation where someone says "your system lost our orders." 90% of the time, the data is there — it's a mapping issue, a status transition bug, or a misconfigured filter. You find this with SQL. If you can't write the query, you're dependent on engineering to answer questions that you should own.

---

## The Schema We'll Use Throughout This File

All examples use a realistic logistics schema. Memorize this — it will appear in practice exercises and the capstone project.

```sql
-- Core tables for a logistics integration platform

CREATE TABLE customers (
    id          SERIAL PRIMARY KEY,
    external_id VARCHAR(64) UNIQUE NOT NULL,   -- e.g. Shopify customer ID
    email       VARCHAR(255) NOT NULL,
    name        VARCHAR(255),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    id              SERIAL PRIMARY KEY,
    external_id     VARCHAR(64) UNIQUE NOT NULL,  -- e.g. "ORD-2024-001"
    customer_id     INTEGER REFERENCES customers(id),
    status          VARCHAR(32) NOT NULL DEFAULT 'pending',
    -- statuses: pending, processing, booked, shipped, delivered, cancelled, failed
    total_value     NUMERIC(12, 2),
    currency        VARCHAR(3) DEFAULT 'USD',
    shipping_name   VARCHAR(255),
    shipping_addr1  VARCHAR(255),
    shipping_addr2  VARCHAR(255),
    shipping_city   VARCHAR(128),
    shipping_state  VARCHAR(64),
    shipping_zip    VARCHAR(20),
    shipping_country VARCHAR(3),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    synced_at       TIMESTAMPTZ,                  -- when we sent it to 3PL
    source          VARCHAR(32) DEFAULT 'shopify' -- shopify, magento, manual
);

CREATE TABLE line_items (
    id          SERIAL PRIMARY KEY,
    order_id    INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    sku         VARCHAR(128) NOT NULL,
    description TEXT,
    quantity    INTEGER NOT NULL DEFAULT 1,
    unit_price  NUMERIC(10, 2),
    weight_kg   NUMERIC(8, 3),
    hs_code     VARCHAR(16),       -- customs tariff code
    origin_country VARCHAR(3)
);

CREATE TABLE shipments (
    id              SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(id),
    booking_id      VARCHAR(128),      -- 3PL internal booking reference
    carrier         VARCHAR(64),       -- FEDEX, UPS, DHL, etc.
    tracking_number VARCHAR(128) UNIQUE,
    service_level   VARCHAR(64),       -- EXPRESS, STANDARD, etc.
    label_url       TEXT,
    estimated_delivery DATE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tracking_events (
    id              SERIAL PRIMARY KEY,
    shipment_id     INTEGER REFERENCES shipments(id),
    event_time      TIMESTAMPTZ NOT NULL,
    location        VARCHAR(255),
    status_code     VARCHAR(32) NOT NULL,
    -- BOOKED, PICKED_UP, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERED, EXCEPTION
    description     TEXT,
    raw_payload     JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sync_log (
    id              SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(id),
    direction       VARCHAR(16) NOT NULL,  -- 'outbound' or 'inbound'
    status          VARCHAR(16) NOT NULL,  -- 'success', 'failure', 'retry'
    attempt_number  INTEGER DEFAULT 1,
    error_message   TEXT,
    request_payload JSONB,
    response_payload JSONB,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Part 1: SELECT Fundamentals

### Basic Filtering with WHERE

```sql
-- Find all pending orders from the last 24 hours
SELECT
    id,
    external_id,
    status,
    total_value,
    created_at
FROM orders
WHERE status = 'pending'
  AND created_at >= NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC;

-- Find orders above a value threshold in specific countries
SELECT external_id, shipping_country, total_value
FROM orders
WHERE total_value > 500.00
  AND shipping_country IN ('US', 'CA', 'GB')
  AND status NOT IN ('cancelled', 'failed');

-- Pattern matching: find orders by partial external_id
SELECT * FROM orders
WHERE external_id LIKE 'ORD-2024-%'
  AND shipping_name ILIKE '%smith%';   -- ILIKE = case-insensitive in PostgreSQL

-- Range queries
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31 23:59:59';
-- Prefer this form (more explicit about range):
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
```

### Sorting and Pagination

```sql
-- Basic sort
SELECT id, external_id, created_at, total_value
FROM orders
ORDER BY created_at DESC, total_value DESC;

-- Pagination — LIMIT + OFFSET (simple but has performance problems at scale)
SELECT id, external_id, created_at
FROM orders
ORDER BY id
LIMIT 100 OFFSET 200;  -- page 3 of 100 results per page

-- Keyset pagination (much better for large tables)
-- Instead of OFFSET, use "give me records after id X"
SELECT id, external_id, created_at
FROM orders
WHERE id > 12345   -- "cursor" from previous page's last row
ORDER BY id
LIMIT 100;
```

> **FDE Context**: When a customer says "show me orders from last month that didn't get shipped," this is the query shape. You filter by date range, filter by status, sort by creation time. Keyset pagination matters when you're debugging a customer with 500,000 orders — OFFSET 490000 will bring PostgreSQL to its knees.

---

## Part 2: JOINs — The Most Critical Topic

### The Mental Model for JOINs

Think of a JOIN as answering: "For each row in table A, find the matching row(s) in table B."

The JOIN type determines what happens when no match exists:
- **INNER JOIN**: Only rows where a match exists in BOTH tables. Non-matching rows are dropped from both sides.
- **LEFT JOIN**: All rows from the LEFT table. Nulls fill in where no match exists in right.
- **RIGHT JOIN**: All rows from the RIGHT table. (Almost always rewrite as LEFT JOIN with tables swapped.)
- **FULL OUTER JOIN**: All rows from both tables. Nulls fill in where no match on either side.

### INNER JOIN

```sql
-- Get orders with their customer info (only orders that have a customer)
SELECT
    o.external_id    AS order_number,
    o.status,
    o.total_value,
    c.name           AS customer_name,
    c.email          AS customer_email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';
```

If an order has `customer_id = NULL` or the customer record was deleted, this order will NOT appear in the results. INNER JOIN silently drops orphaned records.

### LEFT JOIN — The Most Useful JOIN

```sql
-- Get all orders, with their shipment info if it exists
-- Orders without shipments still appear (shipment columns will be NULL)
SELECT
    o.external_id,
    o.status,
    o.created_at,
    s.tracking_number,
    s.carrier,
    s.estimated_delivery
FROM orders o
LEFT JOIN shipments s ON s.order_id = o.id
ORDER BY o.created_at DESC;

-- CRITICAL USE CASE: Find orders that have NO shipment yet
-- This is how you find "stuck" orders
SELECT
    o.external_id,
    o.status,
    o.created_at,
    EXTRACT(EPOCH FROM (NOW() - o.created_at)) / 3600 AS hours_old
FROM orders o
LEFT JOIN shipments s ON s.order_id = o.id
WHERE s.id IS NULL                      -- no matching shipment
  AND o.status NOT IN ('cancelled', 'failed')
  AND o.created_at < NOW() - INTERVAL '2 hours'   -- older than 2 hours
ORDER BY o.created_at;
```

> **FDE Context**: `LEFT JOIN ... WHERE right_table.id IS NULL` is the pattern for finding missing data. "Find all orders that were synced to the 3PL but never got a tracking number back." This pattern is in 30% of all production debugging queries you'll write.

### FULL OUTER JOIN

```sql
-- Compare orders from two systems: find mismatches
-- e_orders = orders from e-commerce system
-- pl_orders = orders from 3PL system
-- Find orders that exist in one system but not the other
SELECT
    COALESCE(e.external_id, pl.ecommerce_ref) AS order_ref,
    CASE
        WHEN pl.ecommerce_ref IS NULL THEN 'only in ecommerce'
        WHEN e.external_id IS NULL    THEN 'only in 3PL'
        ELSE 'in both'
    END AS sync_status
FROM orders e
FULL OUTER JOIN pl_bookings pl ON pl.ecommerce_ref = e.external_id
WHERE pl.ecommerce_ref IS NULL OR e.external_id IS NULL;
```

### Self-Joins

A self-join joins a table to itself. Common use: finding related records in the same table, or comparing a record to other records.

```sql
-- Example: orders table has a "parent_order_id" for split shipments
-- Find original orders and their child orders
ALTER TABLE orders ADD COLUMN parent_order_id INTEGER REFERENCES orders(id);

SELECT
    parent.external_id  AS original_order,
    parent.status       AS original_status,
    child.external_id   AS split_order,
    child.status        AS split_status
FROM orders parent
INNER JOIN orders child ON child.parent_order_id = parent.id
WHERE parent.parent_order_id IS NULL;   -- only root-level orders

-- Self-join: find customers who placed orders on the same day
SELECT
    a.external_id AS order_a,
    b.external_id AS order_b,
    a.customer_id,
    DATE(a.created_at) AS order_date
FROM orders a
INNER JOIN orders b
    ON a.customer_id = b.customer_id
    AND DATE(a.created_at) = DATE(b.created_at)
    AND a.id < b.id   -- CRITICAL: prevents (A,B) and (B,A) duplicate pairs
```

### Multi-Table JOINs

```sql
-- The full order lifecycle query — this is the FDE money query
-- Trace a complete order from creation to last tracking event
SELECT
    o.external_id           AS order_id,
    o.status                AS order_status,
    o.created_at            AS order_created,
    o.synced_at             AS sent_to_3pl,
    c.name                  AS customer,
    c.email                 AS customer_email,
    s.carrier,
    s.tracking_number,
    s.estimated_delivery,
    te.status_code          AS last_event_status,
    te.location             AS last_event_location,
    te.event_time           AS last_event_time,
    sl_last.status          AS last_sync_status,
    sl_last.error_message   AS sync_error
FROM orders o
LEFT JOIN customers c           ON c.id = o.customer_id
LEFT JOIN shipments s           ON s.order_id = o.id
LEFT JOIN LATERAL (
    -- Get only the most recent tracking event (LATERAL subquery)
    SELECT status_code, location, event_time
    FROM tracking_events
    WHERE shipment_id = s.id
    ORDER BY event_time DESC
    LIMIT 1
) te ON TRUE
LEFT JOIN LATERAL (
    -- Get only the most recent sync log entry
    SELECT status, error_message
    FROM sync_log
    WHERE order_id = o.id
    ORDER BY created_at DESC
    LIMIT 1
) sl_last ON TRUE
WHERE o.external_id = 'ORD-2024-001';
```

### Common JOIN Mistakes

**Mistake 1: The Cartesian Product (Missing JOIN condition)**
```sql
-- WRONG: forgot the ON clause — produces every combination
SELECT o.external_id, s.tracking_number
FROM orders o, shipments s;   -- Old-style syntax, easy to forget condition
-- With 10,000 orders and 9,000 shipments: returns 90,000,000 rows

-- RIGHT: always explicit JOIN conditions
SELECT o.external_id, s.tracking_number
FROM orders o
INNER JOIN shipments s ON s.order_id = o.id;
```

**Mistake 2: Fan-out Causing Incorrect Aggregations**
```sql
-- WRONG: if an order has 3 line items and 2 tracking events,
-- the SUM will be inflated (multiplied by join factor)
SELECT
    o.external_id,
    SUM(li.unit_price * li.quantity) AS wrong_total
FROM orders o
LEFT JOIN line_items li ON li.order_id = o.id
LEFT JOIN tracking_events te ON te.shipment_id IN (
    SELECT id FROM shipments WHERE order_id = o.id
)
GROUP BY o.external_id;
-- The line_items rows get duplicated for each tracking_event row

-- RIGHT: aggregate before joining
WITH order_totals AS (
    SELECT order_id, SUM(unit_price * quantity) AS total
    FROM line_items
    GROUP BY order_id
),
latest_events AS (
    SELECT DISTINCT ON (shipment_id) shipment_id, status_code, event_time
    FROM tracking_events
    ORDER BY shipment_id, event_time DESC
)
SELECT
    o.external_id,
    ot.total AS order_total,
    le.status_code AS latest_tracking_status
FROM orders o
LEFT JOIN order_totals ot ON ot.order_id = o.id
LEFT JOIN shipments s ON s.order_id = o.id
LEFT JOIN latest_events le ON le.shipment_id = s.id;
```

**Mistake 3: Lost Rows Because of NULL in JOIN Condition**
```sql
-- If customer_id is NULL for some orders, those orders are silently excluded
-- by INNER JOIN — and no error is raised
SELECT count(*) FROM orders;                    -- 10,432
SELECT count(*) FROM orders o
INNER JOIN customers c ON c.id = o.customer_id; -- 10,401 (31 orders lost!)

-- Check: which orders lost their customer?
SELECT id, external_id, customer_id
FROM orders
WHERE customer_id IS NULL OR customer_id NOT IN (SELECT id FROM customers);
```

---

## Part 3: Aggregations

### GROUP BY and HAVING

```sql
-- Orders per status
SELECT
    status,
    COUNT(*)                                    AS order_count,
    SUM(total_value)                            AS revenue,
    AVG(total_value)                            AS avg_order_value,
    MIN(total_value)                            AS min_order,
    MAX(total_value)                            AS max_order
FROM orders
GROUP BY status
ORDER BY order_count DESC;

-- Orders per day with running stats
SELECT
    DATE_TRUNC('day', created_at)   AS day,
    COUNT(*)                        AS orders,
    SUM(total_value)                AS daily_revenue,
    COUNT(DISTINCT customer_id)     AS unique_customers
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY day;

-- HAVING: filter on aggregated values (not individual rows)
-- Find customers with more than 10 orders in the last month
SELECT
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_value) AS total_spend
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY customer_id
HAVING COUNT(*) > 10
ORDER BY order_count DESC;

-- Combining WHERE and HAVING:
-- WHERE filters rows BEFORE grouping
-- HAVING filters groups AFTER grouping
SELECT
    shipping_country,
    status,
    COUNT(*) AS count
FROM orders
WHERE created_at >= '2024-01-01'     -- filters rows first
GROUP BY shipping_country, status
HAVING COUNT(*) > 100                -- then filters groups
ORDER BY count DESC;
```

> **FDE Context**: Aggregations answer the "how many" and "how much" questions that come up in every customer health review. "How many orders failed to sync this week?" "What's the average time from order creation to shipment booking?" "Which countries have the most delivery exceptions?" All aggregations.

---

## Part 4: Window Functions — The FDE Superpower

Window functions compute values across a set of rows related to the current row, without collapsing rows into groups (unlike GROUP BY). They are the most powerful SQL feature for analytics.

Syntax: `FUNCTION() OVER (PARTITION BY ... ORDER BY ... FRAME)`

### ROW_NUMBER, RANK, DENSE_RANK

```sql
-- ROW_NUMBER: unique sequential number, no ties
-- Use case: get the most recent shipment per order
SELECT *
FROM (
    SELECT
        s.*,
        ROW_NUMBER() OVER (PARTITION BY s.order_id ORDER BY s.created_at DESC) AS rn
    FROM shipments s
) ranked
WHERE rn = 1;   -- keep only the latest shipment per order

-- RANK vs DENSE_RANK — difference matters with ties
SELECT
    order_id,
    total_value,
    RANK()          OVER (ORDER BY total_value DESC) AS rank,
    DENSE_RANK()    OVER (ORDER BY total_value DESC) AS dense_rank,
    ROW_NUMBER()    OVER (ORDER BY total_value DESC) AS row_num
FROM orders
LIMIT 10;
-- If two orders tie for 2nd place:
-- RANK:        1, 2, 2, 4  (skips 3)
-- DENSE_RANK:  1, 2, 2, 3  (no gaps)
-- ROW_NUMBER:  1, 2, 3, 4  (arbitrary tiebreak)
```

### LAG and LEAD — Comparing to Adjacent Rows

These are extremely useful for detecting state transitions.

```sql
-- Detect order status changes: find when an order changed from pending to processing
SELECT
    order_id,
    status          AS current_status,
    LAG(status) OVER (PARTITION BY order_id ORDER BY created_at) AS previous_status,
    created_at      AS changed_at
FROM order_status_history   -- assume this audit table exists
WHERE LAG(status) OVER (PARTITION BY order_id ORDER BY created_at) = 'pending'
  AND status = 'processing';

-- More practical: find the time between tracking events for a shipment
SELECT
    te.shipment_id,
    te.status_code,
    te.event_time,
    LAG(te.event_time) OVER (PARTITION BY te.shipment_id ORDER BY te.event_time) AS prev_event_time,
    te.event_time - LAG(te.event_time) OVER (
        PARTITION BY te.shipment_id ORDER BY te.event_time
    ) AS time_since_last_event
FROM tracking_events te
ORDER BY te.shipment_id, te.event_time;

-- LEAD: look at the NEXT row
-- Find tracking events where the next status is EXCEPTION
SELECT
    status_code AS current_status,
    LEAD(status_code) OVER (PARTITION BY shipment_id ORDER BY event_time) AS next_status,
    event_time
FROM tracking_events
WHERE LEAD(status_code) OVER (PARTITION BY shipment_id ORDER BY event_time) = 'EXCEPTION';
```

### Running Totals with SUM OVER

```sql
-- Running total of daily revenue
SELECT
    DATE_TRUNC('day', created_at)   AS day,
    SUM(total_value)                AS daily_revenue,
    SUM(SUM(total_value)) OVER (
        ORDER BY DATE_TRUNC('day', created_at)
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY day;

-- Moving 7-day average of order volume
SELECT
    day,
    daily_orders,
    AVG(daily_orders) OVER (
        ORDER BY day
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_7day_avg
FROM (
    SELECT
        DATE_TRUNC('day', created_at) AS day,
        COUNT(*) AS daily_orders
    FROM orders
    GROUP BY DATE_TRUNC('day', created_at)
) daily_counts
ORDER BY day;
```

### Frame Clauses Deep Dive

```sql
-- ROWS BETWEEN: physical row offset
-- RANGE BETWEEN: logical value range

-- Running total — standard
SUM(value) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- Running total of just the current and previous 2 rows (3-row window)
SUM(value) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- Symmetric window: 1 row before and 1 row after
AVG(value) OVER (ORDER BY day ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- All rows in the partition (same as no frame, default is RANGE UNBOUNDED PRECEDING)
SUM(value) OVER (PARTITION BY customer_id)
-- This computes total per customer for every row in that customer's partition
```

### NTILE and PERCENT_RANK

```sql
-- Divide orders into 4 quartiles by value
SELECT
    external_id,
    total_value,
    NTILE(4) OVER (ORDER BY total_value) AS value_quartile,
    PERCENT_RANK() OVER (ORDER BY total_value) AS percentile,
    CUME_DIST() OVER (ORDER BY total_value) AS cumulative_dist
FROM orders;

-- Practical: find orders in the top 10% by value
SELECT external_id, total_value
FROM (
    SELECT
        external_id,
        total_value,
        PERCENT_RANK() OVER (ORDER BY total_value) AS pct_rank
    FROM orders
) ranked
WHERE pct_rank >= 0.90;
```

> **FDE Context — Interview Angle**: Window functions come up in technical screens constantly. The canonical FDE question: "Write a query to find the most recent tracking event for each shipment." The naive answer uses a subquery. The expert answer uses `ROW_NUMBER() OVER (PARTITION BY shipment_id ORDER BY event_time DESC)` and filters `WHERE rn = 1`. This one-liner demonstrates you know window functions cold.

---

## Part 5: Common Table Expressions (CTEs)

CTEs make complex queries readable. They're defined with `WITH` and referenced like temporary tables.

```sql
-- Basic CTE: orders with their line item count and total
WITH order_summary AS (
    SELECT
        o.id,
        o.external_id,
        o.status,
        o.total_value,
        COUNT(li.id)                AS line_item_count,
        SUM(li.quantity)            AS total_units
    FROM orders o
    LEFT JOIN line_items li ON li.order_id = o.id
    GROUP BY o.id, o.external_id, o.status, o.total_value
),
orders_with_shipments AS (
    SELECT
        os.*,
        s.tracking_number,
        s.carrier,
        s.estimated_delivery
    FROM order_summary os
    LEFT JOIN shipments s ON s.order_id = os.id
)
SELECT *
FROM orders_with_shipments
WHERE status = 'shipped'
  AND estimated_delivery < NOW()   -- overdue shipments
ORDER BY estimated_delivery;

-- Multiple CTEs: complex debugging query
WITH failed_syncs AS (
    SELECT order_id, MAX(created_at) AS last_attempt, MAX(attempt_number) AS attempts
    FROM sync_log
    WHERE status = 'failure'
    GROUP BY order_id
),
successful_syncs AS (
    SELECT DISTINCT order_id
    FROM sync_log
    WHERE status = 'success'
)
SELECT
    o.external_id,
    o.status,
    o.total_value,
    fs.attempts,
    fs.last_attempt,
    sl.error_message
FROM orders o
INNER JOIN failed_syncs fs ON fs.order_id = o.id
LEFT JOIN successful_syncs ss ON ss.order_id = o.id
LEFT JOIN sync_log sl ON sl.order_id = o.id
    AND sl.created_at = fs.last_attempt   -- get the actual error message
WHERE ss.order_id IS NULL  -- never succeeded
ORDER BY fs.last_attempt DESC;
```

### Recursive CTEs

Recursive CTEs are used for hierarchical data (parent-child relationships, graphs, sequences).

```sql
-- Example: orders have a parent_order_id when they're split
-- Find all orders in a "split order tree" given the root order ID

WITH RECURSIVE order_tree AS (
    -- Base case: start with the root order
    SELECT id, external_id, parent_order_id, 0 AS depth
    FROM orders
    WHERE external_id = 'ORD-2024-001'  -- starting order
    
    UNION ALL
    
    -- Recursive case: find children of each order found so far
    SELECT o.id, o.external_id, o.parent_order_id, ot.depth + 1
    FROM orders o
    INNER JOIN order_tree ot ON o.parent_order_id = ot.id
    WHERE ot.depth < 10  -- safety: prevent infinite recursion
)
SELECT * FROM order_tree ORDER BY depth, external_id;

-- Practical: number sequence generator (useful for generating date series)
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01'::DATE AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < '2024-01-31'
)
SELECT
    ds.dt,
    COALESCE(COUNT(o.id), 0) AS orders
FROM date_series ds
LEFT JOIN orders o ON DATE(o.created_at) = ds.dt
GROUP BY ds.dt
ORDER BY ds.dt;
```

---

## Part 6: Subqueries

### Scalar Subqueries

Return a single value, used inline:

```sql
-- Find orders where total_value > average order value
SELECT external_id, total_value
FROM orders
WHERE total_value > (SELECT AVG(total_value) FROM orders)
ORDER BY total_value DESC;

-- With subquery in SELECT clause
SELECT
    external_id,
    total_value,
    (SELECT AVG(total_value) FROM orders) AS avg_value,
    total_value - (SELECT AVG(total_value) FROM orders) AS delta_from_avg
FROM orders;
```

### Correlated Subqueries

A correlated subquery references the outer query. It runs once per outer row (can be slow for large tables).

```sql
-- Find orders where the customer has made fewer than 3 total orders
SELECT o.external_id, o.customer_id, o.total_value
FROM orders o
WHERE (
    SELECT COUNT(*)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id   -- references outer query!
) < 3;

-- Same query, more efficient with window function:
SELECT external_id, customer_id, total_value
FROM (
    SELECT *, COUNT(*) OVER (PARTITION BY customer_id) AS orders_by_customer
    FROM orders
) sub
WHERE orders_by_customer < 3;
```

### EXISTS Subqueries

EXISTS returns true/false — useful for existence checks and often faster than IN for large sets.

```sql
-- Find orders that have at least one tracking event with EXCEPTION status
SELECT o.external_id, o.status
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM shipments s
    INNER JOIN tracking_events te ON te.shipment_id = s.id
    WHERE s.order_id = o.id
      AND te.status_code = 'EXCEPTION'
);

-- NOT EXISTS: find orders with no sync attempts at all
SELECT o.external_id, o.created_at
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM sync_log sl WHERE sl.order_id = o.id
);
```

---

## Part 7: String Functions

```sql
-- CONCAT and ||
SELECT CONCAT(shipping_name, ', ', shipping_city, ', ', shipping_country) AS full_address
FROM orders;
-- PostgreSQL shorthand:
SELECT shipping_name || ', ' || shipping_city AS short_address FROM orders;

-- SUBSTRING (extract part of string)
SELECT
    external_id,
    SUBSTRING(external_id FROM 5 FOR 4) AS year_part   -- 'ORD-2024-001' → '2024'
FROM orders;
-- Regex-based:
SELECT SUBSTRING(external_id FROM 'ORD-(\d{4})-') AS year FROM orders;

-- TRIM, LTRIM, RTRIM
SELECT TRIM('  hello world  ')     -- 'hello world'
SELECT LTRIM('   hello', ' ')      -- 'hello'
SELECT RTRIM('hello   ')           -- 'hello'

-- UPPER, LOWER
SELECT UPPER(shipping_country), LOWER(carrier) FROM shipments;

-- REPLACE
SELECT REPLACE(tracking_number, '-', '') AS clean_tracking FROM shipments;

-- LENGTH
SELECT external_id, LENGTH(external_id) AS id_length FROM orders;

-- SPLIT_PART (PostgreSQL-specific)
SELECT SPLIT_PART('ORD-2024-001', '-', 2) AS year;  -- returns '2024'

-- REGEXP_REPLACE, REGEXP_MATCH
SELECT REGEXP_REPLACE(shipping_phone, '[^0-9]', '', 'g') AS digits_only
FROM orders;  -- strip non-numeric chars from phone

SELECT REGEXP_MATCH(external_id, 'ORD-(\d{4})-(\d{3})') AS parts FROM orders;
-- Returns ARRAY: {'2024', '001'}

-- String aggregation
SELECT
    customer_id,
    STRING_AGG(external_id, ', ' ORDER BY created_at) AS order_list
FROM orders
GROUP BY customer_id;
```

---

## Part 8: Date/Time Functions

PostgreSQL has excellent date/time support. This matters for logistics where timestamps are everything.

```sql
-- NOW() and CURRENT_TIMESTAMP
SELECT NOW();                    -- timestamp with time zone
SELECT CURRENT_TIMESTAMP;        -- same, SQL standard
SELECT CURRENT_DATE;             -- date only, no time
SELECT CURRENT_TIME;             -- time only

-- DATE_TRUNC: truncate to a precision
SELECT DATE_TRUNC('day',   created_at) AS day   FROM orders;
SELECT DATE_TRUNC('month', created_at) AS month FROM orders;
SELECT DATE_TRUNC('hour',  created_at) AS hour  FROM orders;
-- Other precisions: microseconds, milliseconds, second, minute, week, quarter, year

-- EXTRACT: get a component
SELECT
    EXTRACT(YEAR   FROM created_at) AS year,
    EXTRACT(MONTH  FROM created_at) AS month,
    EXTRACT(DOW    FROM created_at) AS day_of_week,  -- 0=Sunday, 6=Saturday
    EXTRACT(HOUR   FROM created_at) AS hour,
    EXTRACT(EPOCH  FROM created_at) AS unix_timestamp
FROM orders;

-- Interval arithmetic
SELECT
    created_at,
    created_at + INTERVAL '2 days' AS plus_2_days,
    created_at - INTERVAL '1 hour' AS minus_1_hour,
    NOW() - created_at AS age_of_order
FROM orders;

-- AGE: human-readable interval
SELECT AGE(NOW(), created_at) AS order_age FROM orders;
-- Returns: '3 days 14:32:10'

-- AT TIME ZONE: convert between time zones
SELECT
    created_at,
    created_at AT TIME ZONE 'UTC' AS utc,
    created_at AT TIME ZONE 'America/New_York' AS eastern,
    created_at AT TIME ZONE 'Asia/Shanghai' AS china
FROM orders;

-- Date truncation for grouping
SELECT
    DATE_TRUNC('week', created_at) AS week_start,
    COUNT(*) AS orders,
    SUM(total_value) AS revenue
FROM orders
GROUP BY DATE_TRUNC('week', created_at)
ORDER BY week_start;

-- Practical: find orders created on weekends
SELECT external_id, created_at
FROM orders
WHERE EXTRACT(DOW FROM created_at) IN (0, 6);   -- 0=Sun, 6=Sat

-- Time-to-ship calculation
SELECT
    o.external_id,
    o.created_at AS order_time,
    s.created_at AS shipment_booked_time,
    s.created_at - o.created_at AS time_to_ship,
    EXTRACT(EPOCH FROM (s.created_at - o.created_at))/3600 AS hours_to_ship
FROM orders o
INNER JOIN shipments s ON s.order_id = o.id
ORDER BY hours_to_ship DESC;
```

> **FDE Context**: Time zones destroy more integrations than anything else. A customer in Chicago creates an order at 11 PM CDT. Your system stores it in UTC. Their 3PL filters orders by "created today" in EST. The order disappears into a gap. Always store timestamps in UTC, convert at display time. When debugging timestamp issues, always check `AT TIME ZONE` conversions.

---

## Part 9: CASE WHEN

Conditional logic in SQL. Equivalent to if/else.

```sql
-- Simple CASE WHEN
SELECT
    external_id,
    total_value,
    CASE
        WHEN total_value < 50   THEN 'small'
        WHEN total_value < 200  THEN 'medium'
        WHEN total_value < 1000 THEN 'large'
        ELSE 'enterprise'
    END AS order_tier
FROM orders;

-- CASE in aggregation: conditional counts
SELECT
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered'  THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'failed'     THEN 1 END) AS failed,
    COUNT(CASE WHEN status = 'cancelled'  THEN 1 END) AS cancelled,
    -- Alternative with FILTER syntax (PostgreSQL 9.4+)
    COUNT(*) FILTER (WHERE status = 'shipped') AS shipped
FROM orders;

-- CASE in ORDER BY
SELECT external_id, status, created_at
FROM orders
ORDER BY
    CASE status
        WHEN 'failed'    THEN 1   -- show failures first
        WHEN 'pending'   THEN 2
        WHEN 'processing' THEN 3
        ELSE 4
    END,
    created_at DESC;

-- CASE to pivot (turn rows into columns)
SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(CASE WHEN shipping_country = 'US' THEN total_value ELSE 0 END) AS us_revenue,
    SUM(CASE WHEN shipping_country = 'CA' THEN total_value ELSE 0 END) AS ca_revenue,
    SUM(CASE WHEN shipping_country = 'GB' THEN total_value ELSE 0 END) AS gb_revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

---

## Part 10: NULL Handling

NULLs are not values. They are the absence of a value. This causes counterintuitive behavior.

```sql
-- NULL comparison: NEVER use = NULL, always IS NULL
SELECT * FROM orders WHERE synced_at IS NULL;     -- orders never synced
SELECT * FROM orders WHERE synced_at IS NOT NULL; -- orders that were synced

-- NULL in arithmetic: NULL + anything = NULL
SELECT 100 + NULL;   -- returns NULL, not 100!
-- This is why COUNT(*) ≠ COUNT(column) when column has NULLs
SELECT COUNT(*),                  -- counts all rows
       COUNT(synced_at),          -- counts only rows where synced_at is not null
       COUNT(DISTINCT customer_id) -- counts distinct non-null customer_ids
FROM orders;

-- COALESCE: return first non-NULL value
SELECT
    external_id,
    COALESCE(shipping_addr2, '') AS addr2,   -- never show NULL in output
    COALESCE(total_value, 0.00) AS value,
    COALESCE(synced_at, created_at) AS relevant_timestamp  -- fallback chain
FROM orders;

-- NULLIF: return NULL if two values are equal (opposite of COALESCE)
SELECT NULLIF(total_value, 0)    -- treat 0 as missing
FROM orders;
-- Use case: avoid division by zero
SELECT total_value / NULLIF(quantity, 0) AS unit_price FROM line_items;

-- NULL in boolean logic (three-valued logic)
-- TRUE AND NULL = NULL (not FALSE!)
-- FALSE AND NULL = FALSE
-- TRUE OR NULL = TRUE
-- FALSE OR NULL = NULL
SELECT * FROM orders WHERE NOT (status = 'pending');
-- This EXCLUDES rows where status IS NULL (they're not 'pending', but NOT IN includes only not-null values)
-- Safer:
SELECT * FROM orders WHERE status IS DISTINCT FROM 'pending';

-- IS DISTINCT FROM / IS NOT DISTINCT FROM: NULL-safe comparison
SELECT * FROM orders WHERE shipping_addr2 IS DISTINCT FROM NULL;  -- same as IS NOT NULL
SELECT 'a' IS DISTINCT FROM 'a';    -- FALSE (same as <>, but NULL-safe)
SELECT NULL IS DISTINCT FROM NULL;  -- FALSE (NULLs compare equal here)
```

---

## Part 11: UPSERT

UPSERT = INSERT if not exists, UPDATE if it does. Critical for idempotent sync operations.

```sql
-- PostgreSQL: INSERT ... ON CONFLICT DO UPDATE (aka "upsert")
INSERT INTO orders (external_id, customer_id, status, total_value, updated_at)
VALUES ('ORD-2024-999', 42, 'pending', 299.99, NOW())
ON CONFLICT (external_id)    -- conflict on unique column
DO UPDATE SET
    status      = EXCLUDED.status,        -- EXCLUDED refers to the attempted insert row
    total_value = EXCLUDED.total_value,
    updated_at  = EXCLUDED.updated_at
WHERE orders.updated_at < EXCLUDED.updated_at;   -- only update if newer

-- ON CONFLICT DO NOTHING (ignore duplicates)
INSERT INTO sync_log (order_id, direction, status, created_at)
VALUES (123, 'outbound', 'success', NOW())
ON CONFLICT DO NOTHING;

-- Practical: bulk upsert from a staging table
INSERT INTO orders (external_id, status, total_value, synced_at)
SELECT external_id, status, total_value, NOW()
FROM staging_orders
ON CONFLICT (external_id) DO UPDATE SET
    status    = EXCLUDED.status,
    synced_at = EXCLUDED.synced_at;
```

---

## Part 12: Query Tuning Basics

### EXPLAIN and EXPLAIN ANALYZE

```sql
-- EXPLAIN shows the query plan without running the query
EXPLAIN SELECT * FROM orders WHERE external_id = 'ORD-2024-001';

-- EXPLAIN ANALYZE actually runs the query and shows real timings
EXPLAIN ANALYZE SELECT * FROM orders WHERE external_id = 'ORD-2024-001';

-- More verbose output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT * FROM orders WHERE external_id = 'ORD-2024-001';
```

Reading EXPLAIN output:
```
Seq Scan on orders  (cost=0.00..245.00 rows=1 width=312) (actual time=0.021..15.342 rows=1 loops=1)
  Filter: ((external_id)::text = 'ORD-2024-001'::text)
  Rows Removed by Filter: 9999
Planning Time: 0.123 ms
Execution Time: 15.398 ms

-- vs with an index:
Index Scan using orders_external_id_idx on orders  (cost=0.29..8.31 rows=1 width=312)
  Index Cond: ((external_id)::text = 'ORD-2024-001'::text)
Planning Time: 0.087 ms
Execution Time: 0.043 ms
```

Key things to look for:
- **Seq Scan**: scanning every row. Bad for large tables unless you want all rows.
- **Index Scan**: using an index. Good for selective queries.
- **Index Only Scan**: even better — doesn't need to visit heap at all.
- **Hash Join vs Nested Loop vs Merge Join**: join algorithms. Nested loop is bad for large tables without an index.
- **cost=X..Y**: estimated cost. X = startup cost, Y = total cost. Higher = slower.
- **rows=N**: estimated row count. Wildly wrong estimates = stale statistics. Run `ANALYZE orders;`.

### Seq Scan vs Index Scan

```sql
-- When PostgreSQL chooses a Seq Scan even with an index:
-- 1. Low selectivity: the query would return >10-15% of rows anyway
--    (scanning the index + heap is MORE work than just scanning the heap)
-- 2. Small table: cheaper to just read all rows
-- 3. Stale statistics: planner thinks more rows will be returned

-- Force PostgreSQL to use an index (for testing only)
SET enable_seqscan = OFF;
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
SET enable_seqscan = ON;  -- ALWAYS re-enable after testing!

-- When NOT to use an index:
-- - Low cardinality columns with many results (status = 'active' returns 80% of rows)
-- - Columns rarely queried
-- - Tables updated extremely frequently (index maintenance overhead)
-- - When you're doing a full table scan intentionally (ETL, batch processing)
```

### The N+1 Query Problem

This is the most common performance problem in web applications.

```sql
-- The N+1 pattern in Python pseudocode:
-- WRONG:
orders = db.query("SELECT * FROM orders LIMIT 100")  # 1 query
for order in orders:
    items = db.query(f"SELECT * FROM line_items WHERE order_id = {order.id}")  # N queries!
# Total: 101 queries for 100 orders

-- RIGHT: Use a JOIN
orders_with_items = db.query("""
    SELECT o.*, li.sku, li.quantity, li.unit_price
    FROM orders o
    LEFT JOIN line_items li ON li.order_id = o.id
    WHERE o.id = ANY(:order_ids)
    ORDER BY o.id, li.id
""", order_ids=order_ids)
# Total: 1 query
```

---

## Part 13: Indexing

### B-tree Indexes (Default)

```sql
-- Create basic index
CREATE INDEX idx_orders_external_id ON orders (external_id);
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_status ON orders (status);

-- Composite index: order matters! Most selective column first
-- This index helps queries that filter by (status, created_at) OR just (status)
-- It does NOT help queries that filter only by (created_at)
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

-- Covering index: include extra columns so the query can be answered from index alone
CREATE INDEX idx_orders_status_covering ON orders (status)
INCLUDE (external_id, total_value, created_at);
-- A query SELECT external_id, total_value FROM orders WHERE status = 'pending'
-- can be answered entirely from the index (index-only scan)
```

### Partial Indexes

Index only a subset of rows. Much smaller, faster for queries on that subset.

```sql
-- Only index pending orders (if you query pending orders much more than others)
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Index orders that haven't been synced yet
CREATE INDEX idx_orders_unsynced ON orders (created_at)
WHERE synced_at IS NULL;

-- This query will use the partial index:
SELECT * FROM orders WHERE status = 'pending' AND created_at > NOW() - INTERVAL '1 hour';
```

### Expression Indexes

Index the result of an expression.

```sql
-- If you frequently query by LOWER(email):
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

-- Query that uses this index:
SELECT * FROM customers WHERE LOWER(email) = 'user@example.com';

-- Index on date truncation (if you query by day frequently):
CREATE INDEX idx_orders_day ON orders (DATE_TRUNC('day', created_at));
```

---

## Part 14: SQLAlchemy — Python ORM and Core

SQLAlchemy is the standard Python SQL toolkit. FDEs use it to write data access code for integrations.

### Setup and Connection

```python
# requirements.txt:
# sqlalchemy>=2.0
# psycopg2-binary  (sync driver)
# asyncpg          (async driver)

from sqlalchemy import create_engine, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, Session
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy import String, Integer, Numeric, DateTime, ForeignKey
from datetime import datetime
from typing import Optional, List
import os

# Sync engine
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:pass@localhost/logisync")
engine = create_engine(DATABASE_URL, echo=True, pool_size=10, max_overflow=20)

# Async engine (for FastAPI)
ASYNC_DATABASE_URL = DATABASE_URL.replace("postgresql://", "postgresql+asyncpg://")
async_engine = create_async_engine(ASYNC_DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(async_engine, expire_on_commit=False)
```

### ORM Models

```python
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import String, Integer, Numeric, DateTime, ForeignKey, Text, JSON
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime
from typing import Optional, List

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "customers"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    external_id: Mapped[str] = mapped_column(String(64), unique=True, nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    name: Mapped[Optional[str]] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    
    # Relationship
    orders: Mapped[List["Order"]] = relationship("Order", back_populates="customer")
    
    def __repr__(self):
        return f"<Customer {self.external_id} {self.email}>"


class Order(Base):
    __tablename__ = "orders"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    external_id: Mapped[str] = mapped_column(String(64), unique=True, nullable=False)
    customer_id: Mapped[Optional[int]] = mapped_column(ForeignKey("customers.id"))
    status: Mapped[str] = mapped_column(String(32), default="pending", nullable=False)
    total_value: Mapped[Optional[float]] = mapped_column(Numeric(12, 2))
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    shipping_name: Mapped[Optional[str]] = mapped_column(String(255))
    shipping_addr1: Mapped[Optional[str]] = mapped_column(String(255))
    shipping_city: Mapped[Optional[str]] = mapped_column(String(128))
    shipping_state: Mapped[Optional[str]] = mapped_column(String(64))
    shipping_zip: Mapped[Optional[str]] = mapped_column(String(20))
    shipping_country: Mapped[Optional[str]] = mapped_column(String(3))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    synced_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    
    # Relationships
    customer: Mapped[Optional["Customer"]] = relationship("Customer", back_populates="orders")
    line_items: Mapped[List["LineItem"]] = relationship("LineItem", back_populates="order", cascade="all, delete-orphan")
    shipments: Mapped[List["Shipment"]] = relationship("Shipment", back_populates="order")
    sync_logs: Mapped[List["SyncLog"]] = relationship("SyncLog", back_populates="order")


class LineItem(Base):
    __tablename__ = "line_items"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id", ondelete="CASCADE"), nullable=False)
    sku: Mapped[str] = mapped_column(String(128), nullable=False)
    description: Mapped[Optional[str]] = mapped_column(Text)
    quantity: Mapped[int] = mapped_column(Integer, default=1, nullable=False)
    unit_price: Mapped[Optional[float]] = mapped_column(Numeric(10, 2))
    weight_kg: Mapped[Optional[float]] = mapped_column(Numeric(8, 3))
    hs_code: Mapped[Optional[str]] = mapped_column(String(16))
    
    order: Mapped["Order"] = relationship("Order", back_populates="line_items")


class Shipment(Base):
    __tablename__ = "shipments"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"), nullable=False)
    booking_id: Mapped[Optional[str]] = mapped_column(String(128))
    carrier: Mapped[Optional[str]] = mapped_column(String(64))
    tracking_number: Mapped[Optional[str]] = mapped_column(String(128), unique=True)
    service_level: Mapped[Optional[str]] = mapped_column(String(64))
    label_url: Mapped[Optional[str]] = mapped_column(Text)
    estimated_delivery: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=False))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    
    order: Mapped["Order"] = relationship("Order", back_populates="shipments")
    tracking_events: Mapped[List["TrackingEvent"]] = relationship("TrackingEvent", back_populates="shipment")


class TrackingEvent(Base):
    __tablename__ = "tracking_events"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    shipment_id: Mapped[int] = mapped_column(ForeignKey("shipments.id"), nullable=False)
    event_time: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    location: Mapped[Optional[str]] = mapped_column(String(255))
    status_code: Mapped[str] = mapped_column(String(32), nullable=False)
    description: Mapped[Optional[str]] = mapped_column(Text)
    raw_payload: Mapped[Optional[dict]] = mapped_column(JSONB)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    
    shipment: Mapped["Shipment"] = relationship("Shipment", back_populates="tracking_events")


class SyncLog(Base):
    __tablename__ = "sync_log"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"), nullable=False)
    direction: Mapped[str] = mapped_column(String(16), nullable=False)
    status: Mapped[str] = mapped_column(String(16), nullable=False)
    attempt_number: Mapped[int] = mapped_column(Integer, default=1)
    error_message: Mapped[Optional[str]] = mapped_column(Text)
    request_payload: Mapped[Optional[dict]] = mapped_column(JSONB)
    response_payload: Mapped[Optional[dict]] = mapped_column(JSONB)
    duration_ms: Mapped[Optional[int]] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    
    order: Mapped["Order"] = relationship("Order", back_populates="sync_logs")
```

### SQLAlchemy Core (Direct SQL with Python)

```python
from sqlalchemy import select, insert, update, delete, and_, or_, func, text
from sqlalchemy.orm import Session

# Select with Core
def get_unsynced_orders(session: Session, limit: int = 100) -> list:
    stmt = (
        select(Order)
        .where(
            and_(
                Order.synced_at.is_(None),
                Order.status.notin_(["cancelled", "failed"]),
                Order.created_at < func.now() - text("INTERVAL '5 minutes'")
            )
        )
        .order_by(Order.created_at.asc())
        .limit(limit)
    )
    return session.scalars(stmt).all()


# Join query with Core
def get_order_with_lifecycle(session: Session, external_id: str) -> dict:
    stmt = (
        select(
            Order.external_id,
            Order.status,
            Order.created_at,
            Order.synced_at,
            Customer.name.label("customer_name"),
            Customer.email.label("customer_email"),
            Shipment.tracking_number,
            Shipment.carrier,
            Shipment.estimated_delivery,
        )
        .select_from(Order)
        .outerjoin(Customer, Order.customer_id == Customer.id)
        .outerjoin(Shipment, Shipment.order_id == Order.id)
        .where(Order.external_id == external_id)
    )
    result = session.execute(stmt).mappings().first()
    return dict(result) if result else None


# Upsert with Core
from sqlalchemy.dialects.postgresql import insert as pg_insert

def upsert_order(session: Session, order_data: dict) -> Order:
    stmt = pg_insert(Order).values(**order_data)
    stmt = stmt.on_conflict_do_update(
        index_elements=["external_id"],
        set_={
            "status": stmt.excluded.status,
            "total_value": stmt.excluded.total_value,
            "updated_at": stmt.excluded.updated_at,
        },
        where=Order.updated_at < stmt.excluded.updated_at,
    )
    result = session.execute(stmt)
    session.commit()
    return result


# Bulk insert
def bulk_insert_tracking_events(session: Session, events: list[dict]) -> None:
    session.execute(insert(TrackingEvent), events)
    session.commit()
```

### Async SQLAlchemy with FastAPI

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from contextlib import asynccontextmanager
from typing import AsyncGenerator

app = FastAPI()

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise

# Async ORM query
async def get_order_async(db: AsyncSession, external_id: str) -> Order | None:
    stmt = (
        select(Order)
        .where(Order.external_id == external_id)
        .options(
            # Eager load relationships to avoid N+1
            selectinload(Order.line_items),
            selectinload(Order.shipments).selectinload(Shipment.tracking_events),
        )
    )
    result = await db.execute(stmt)
    return result.scalar_one_or_none()

@app.get("/orders/{external_id}")
async def read_order(external_id: str, db: AsyncSession = Depends(get_db)):
    order = await get_order_async(db, external_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order

# The selectinload pattern prevents N+1:
# Instead of one query per relationship (N+1 queries),
# it does 2 queries total: one for orders, one for all related items
from sqlalchemy.orm import selectinload
```

---

## Part 15: The FDE Money Query — Full Order Lifecycle Trace

This is the query you need to write in customer escalations. Practice it until you can write it from memory.

```sql
-- "Why did order ORD-2024-001 not get delivered?"
-- This single query tells the complete story

WITH order_base AS (
    SELECT
        o.id,
        o.external_id,
        o.status,
        o.created_at,
        o.synced_at,
        o.total_value,
        o.shipping_country,
        c.name AS customer_name,
        c.email AS customer_email
    FROM orders o
    LEFT JOIN customers c ON c.id = o.customer_id
    WHERE o.external_id = 'ORD-2024-001'
),
sync_history AS (
    SELECT
        order_id,
        json_agg(
            json_build_object(
                'attempt', attempt_number,
                'status', status,
                'error', error_message,
                'time', created_at,
                'duration_ms', duration_ms
            ) ORDER BY created_at
        ) AS sync_attempts
    FROM sync_log
    WHERE order_id = (SELECT id FROM order_base)
    GROUP BY order_id
),
shipment_data AS (
    SELECT
        s.order_id,
        s.booking_id,
        s.carrier,
        s.tracking_number,
        s.service_level,
        s.estimated_delivery,
        json_agg(
            json_build_object(
                'time', te.event_time,
                'status', te.status_code,
                'location', te.location,
                'description', te.description
            ) ORDER BY te.event_time
        ) AS tracking_events
    FROM shipments s
    LEFT JOIN tracking_events te ON te.shipment_id = s.id
    WHERE s.order_id = (SELECT id FROM order_base)
    GROUP BY s.order_id, s.booking_id, s.carrier, s.tracking_number,
             s.service_level, s.estimated_delivery
),
line_item_data AS (
    SELECT
        order_id,
        json_agg(
            json_build_object(
                'sku', sku,
                'description', description,
                'quantity', quantity,
                'unit_price', unit_price,
                'weight_kg', weight_kg,
                'hs_code', hs_code
            ) ORDER BY id
        ) AS items
    FROM line_items
    WHERE order_id = (SELECT id FROM order_base)
    GROUP BY order_id
)
SELECT
    ob.*,
    sh.sync_attempts,
    sd.booking_id,
    sd.carrier,
    sd.tracking_number,
    sd.service_level,
    sd.estimated_delivery,
    sd.tracking_events,
    ld.items AS line_items,
    -- Diagnosis hints
    CASE
        WHEN ob.synced_at IS NULL THEN 'NEVER SENT TO 3PL'
        WHEN sd.tracking_number IS NULL THEN 'SENT BUT NO BOOKING CONFIRMED'
        WHEN sd.tracking_events IS NULL THEN 'BOOKED BUT NEVER PICKED UP'
        WHEN NOT (sd.tracking_events::text LIKE '%DELIVERED%') THEN 'IN TRANSIT'
        ELSE 'DELIVERED'
    END AS diagnosis
FROM order_base ob
LEFT JOIN sync_history sh ON sh.order_id = ob.id
LEFT JOIN shipment_data sd ON sd.order_id = ob.id
LEFT JOIN line_item_data ld ON ld.order_id = ob.id;
```

---

## Common Failure Modes

1. **Cartesian product from missing join condition**: returns billions of rows, crashes the database. Always verify row counts with `SELECT COUNT(*)` before SELECT *.

2. **Fan-out in aggregations**: joining to a one-to-many relationship before aggregating causes inflated sums. Aggregate in a CTE first, then join.

3. **NULL comparison with `= NULL`**: always `IS NULL`. The expression `WHERE column = NULL` silently returns zero rows.

4. **Missing index on foreign key**: PostgreSQL does NOT automatically create indexes on foreign key columns (unlike some other databases). Every `REFERENCES` column should have a manual `CREATE INDEX`.

5. **OFFSET pagination at scale**: `OFFSET 50000` requires scanning and discarding 50,000 rows. Use keyset pagination for anything beyond page 10.

6. **Timezone-naive timestamps**: storing timestamps without timezone in a multi-region system leads to phantom data gaps. Always use `TIMESTAMPTZ`.

7. **Correlated subquery in SELECT clause**: if you have a correlated subquery in the SELECT list and return 1,000 rows, the subquery runs 1,000 times. Rewrite as a JOIN or window function.

8. **Stale query planner statistics**: after a large data load, run `ANALYZE table_name;` or `VACUUM ANALYZE table_name;` to update statistics so the query planner makes good decisions.

---

## Interview Angle

**Most common SQL questions for FDE interviews:**

1. "Write a query to find all orders that failed to sync in the last 24 hours, along with their last error message." (Tests: JOINs, date functions, latest-per-group pattern)

2. "Given an `orders` table and a `shipments` table, find orders that have been booked but haven't received a delivery scan in 5+ days." (Tests: LEFT JOIN, date arithmetic, tracking events)

3. "How would you optimize a query that's doing a full table scan on the orders table for status = 'pending'?" (Tests: indexing knowledge, partial indexes, EXPLAIN ANALYZE)

4. "Write a query to calculate the 7-day rolling average of shipments per day." (Tests: window functions, DATE_TRUNC)

5. "Explain the difference between WHERE and HAVING." (Tests: fundamentals — many people confuse these)

**The framing that impresses interviewers**: Don't just write the SQL — explain what you'd do with it. "I'd run this query to identify the stuck orders, then look at the error_message to see if it's an auth error, a schema mismatch, or a rate limit. If it's auth, I'd check the API key expiry. If it's a schema mismatch, I'd look at what changed in the customer's Shopify setup."

---

## Practice Exercise

**Scenario**: You're on a call with a logistics customer. They say: "We had 47 orders go out yesterday, but our 3PL says they only received 43. We need to know which 4 orders didn't make it and why."

Write the following queries using the schema from this file:

1. **Query 1**: Find all orders created in the last 48 hours with their sync status. Show: `external_id`, `status`, `created_at`, `synced_at`, `sync_success` (true/false), `last_sync_error`.

2. **Query 2**: Find the 4 orders that exist in the `orders` table but have no successful entry in `sync_log` for the last 48 hours.

3. **Query 3**: For those 4 orders, show the complete sync attempt history (all attempts, errors, timing) sorted by attempt number.

4. **Query 4**: Using a window function, rank the 4 failed orders by how many sync attempts they've had (most attempts first).

5. **Bonus**: Write the equivalent SQLAlchemy query for Query 2 using the ORM.

**Expected answers**: Work through each query mentally before looking at solutions. The goal is to build the intuition to translate "customer complaint" → "SQL query" fluently.

---

*Next file: [02-etl-elt.md](./02-etl-elt.md) — ETL vs ELT patterns for FDE context*
