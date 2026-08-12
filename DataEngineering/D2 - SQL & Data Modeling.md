---
tags:
  - DE101
  - domain-2
  - sql
  - data-modeling
date: 2026-06-20
status: complete
domain: "2 of 7"
track: data-engineering
---

# D2 — SQL & Data Modeling

**Back to:** [[00 - Onboarding Roadmap]]

> [!NOTE] Domain Overview
> You already query data. This domain pushes you to **build and optimize** data structures. The shift: from writing `SELECT` to designing the tables that others `SELECT` from.

> [!TIP] Quick Self-Assessment
> If you can comfortably write multi-table joins and GROUP BY queries, skip ahead to 2.1. SQL basics are assumed.

---

## 2.1 — Window Functions & CTEs

> [!NOTE] What You'll Learn
> Window functions let you compute rankings, running totals, and row comparisons without collapsing your result set. CTEs replace nested subqueries with readable, named building blocks. These two tools are the foundation of analytical SQL.

### Window Functions vs GROUP BY

The critical difference:

| | `GROUP BY` | Window Function |
|---|---|---|
| Rows returned | One row per group | Same number of rows as input |
| Use case | Aggregate summary | Per-row context with group awareness |
| Example | Total revenue per region | Revenue + cumulative total, one row per order |

`GROUP BY` **collapses** rows. Window functions **annotate** rows.

> [!IMPORTANT] Window Functions Don't Reduce Rows
> Every input row produces exactly one output row. The window function computes its result by looking at a *window* of surrounding rows, but the current row is never removed. This is the fundamental difference from `GROUP BY`.

### Syntax

```sql
FUNC() OVER (
    PARTITION BY <column>   -- defines the group (optional; omit for one global window)
    ORDER BY    <column>    -- defines row order within the group
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- frame clause (optional)
)
```

- **`PARTITION BY`** — resets the window per group (like `GROUP BY`, but rows aren't collapsed)
- **`ORDER BY`** — determines which rows are "before" or "after" the current row
- **Frame clause** — limits how many rows to include in the calculation (defaults vary by function)

### Key Window Functions

| Function | Category | Description |
|----------|----------|-------------|
| `ROW_NUMBER()` | Ranking | Unique sequential number — no ties ever |
| `RANK()` | Ranking | Ties get the same rank; next rank **skips** |
| `DENSE_RANK()` | Ranking | Ties get the same rank; next rank **does not skip** |
| `LAG(col, n)` | Navigation | Value from `n` rows **before** the current row |
| `LEAD(col, n)` | Navigation | Value from `n` rows **after** the current row |
| `SUM() OVER` | Aggregation | Running or windowed sum |
| `AVG() OVER` | Aggregation | Running or windowed average |
| `FIRST_VALUE()` | Navigation | First value in the window frame |

### Frame Clauses

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW          -- running total from start to now
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW                  -- rolling 7-row window
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- entire partition (full group total)
```

### CTEs: Writing SQL That Reads Like Steps

A CTE (Common Table Expression) names a subquery so it can be referenced by name later in the same query.

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name;
```

**Why CTEs over nested subqueries:**

| | Nested Subquery | CTE |
|---|---|---|
| Readability | Inside-out — hard to follow | Top-to-bottom — reads like numbered steps |
| Reuse | Must duplicate if referenced twice | Name it once, reference it anywhere |
| Debugging | Isolating the inner query is painful | Test each CTE independently |

**Chaining CTEs** — multiple named steps in one query:

```sql
WITH step1 AS (...),
     step2 AS (SELECT ... FROM step1),
     step3 AS (SELECT ... FROM step2)
SELECT * FROM step3;
```

### DuckDB Examples

**Example 1 — Ranking orders per customer**

```sql
SELECT
    customer_id,
    order_id,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS row_num,
    RANK()       OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS dense_rnk
FROM orders;
-- For two tied values: RANK produces 1,2,2,4 — DENSE_RANK produces 1,2,2,3
```

**Example 2 — Running total of revenue per customer**

```sql
SELECT
    order_date,
    customer_id,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders;
```

**Example 3 — Month-over-month comparison with LAG()**

```sql
-- Assume: monthly_revenue(month DATE, revenue DECIMAL(12,2))
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month)                              AS prev_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month)                    AS mom_change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month))
        / NULLIF(LAG(revenue, 1) OVER (ORDER BY month), 0) * 100,
        2
    )                                                                   AS mom_pct_change
FROM monthly_revenue
ORDER BY month;
```

**Example 4 — Multi-step CTE pipeline**

```sql
WITH completed_orders AS (
    -- Step 1: filter to completed orders only
    SELECT *
    FROM orders
    WHERE status = 'completed'
),
customer_totals AS (
    -- Step 2: aggregate per customer
    SELECT
        customer_id,
        COUNT(*)          AS order_count,
        SUM(total_amount) AS total_spend
    FROM completed_orders
    GROUP BY customer_id
),
high_value_customers AS (
    -- Step 3: filter to high-value segment
    SELECT *
    FROM customer_totals
    WHERE total_spend > 1000
)
SELECT *
FROM high_value_customers
ORDER BY total_spend DESC;
```

> [!WARNING] Common Window Function Mistakes
>
> ❌ **Nested subqueries instead of CTEs** — deeply nested `SELECT` inside `SELECT` inside `SELECT` is impossible to debug
> ✅ Use named CTEs — each step is isolated, named, and testable independently
>
> ❌ **Confusing `RANK()` and `DENSE_RANK()`** — `RANK()` produces gaps after ties (`1, 2, 2, 4`); `DENSE_RANK()` never skips (`1, 2, 2, 3`). Using `RANK()` for "top N" may exclude valid rows at the boundary
> ✅ Use `DENSE_RANK()` when you need contiguous rankings with no gaps
>
> ❌ **Omitting `ORDER BY` in a windowed aggregation** — `SUM(amount) OVER (PARTITION BY customer_id)` computes the *full partition total* on every row, not a running total
> ✅ Add `ORDER BY order_date` to get a true cumulative accumulation

---

## 2.2 — SQL for Data Engineering

> [!NOTE] What You'll Learn
> Beyond `SELECT`, data engineers write SQL to build and maintain data infrastructure — creating tables, loading data safely, and writing pipelines that produce the same result whether run once or ten times.

### DDL — Defining Structure

DDL (Data Definition Language) defines the shape of your database objects.

```sql
-- Customers table with constraints
CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    email        VARCHAR UNIQUE NOT NULL,
    name         VARCHAR NOT NULL,
    country      VARCHAR,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table with FK and CHECK constraints
CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
    status       VARCHAR NOT NULL CHECK (status IN ('pending', 'completed', 'cancelled')),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

**Constraint types and their purpose:**

| Constraint | Purpose |
|-----------|---------|
| `PRIMARY KEY` | Unique, non-null identifier for every row |
| `NOT NULL` | Column must always have a value |
| `UNIQUE` | No two rows may share this value |
| `CHECK` | Custom business rule the value must satisfy |
| `FOREIGN KEY` | References a row in another table — enforces referential integrity |

**Schema evolution:**

```sql
ALTER TABLE orders ADD COLUMN shipping_address VARCHAR;
ALTER TABLE orders DROP COLUMN shipping_address;
DROP TABLE IF EXISTS staging_orders;  -- IF EXISTS prevents errors in idempotent scripts
```

### DML — Moving Data

DML (Data Manipulation Language) inserts, modifies, and removes rows.

| Statement | Effect | Supports `WHERE` | Speed on Full Clear |
|-----------|--------|:-----------------:|:-----------------:|
| `INSERT` | Add rows | N/A | — |
| `UPDATE` | Modify existing rows | Yes | Slow (row-by-row) |
| `DELETE` | Remove rows | Yes | Slow (row-by-row) |
| `TRUNCATE` | Empty entire table instantly | No | Fast |

```sql
INSERT INTO customers (customer_id, name, email, country)
VALUES (1, 'Alice Nguyen', 'alice@example.com', 'Vietnam');

UPDATE customers SET country = 'Singapore' WHERE customer_id = 1;

DELETE FROM customers WHERE customer_id = 1;

TRUNCATE TABLE staging_events;  -- faster than DELETE for a full table wipe
```

### MERGE / UPSERT — The Idempotent Write Pattern

> [!IMPORTANT] Pipelines Must Be Idempotent
> **Idempotent** means: running your pipeline twice produces the same result as running it once. `UPSERT` is the SQL mechanism that makes this possible — it inserts a row if it doesn't exist yet, or updates it if it does. No duplicates, no errors on re-run.

In DuckDB, use `INSERT ... ON CONFLICT DO UPDATE`:

```sql
INSERT INTO customers (customer_id, name, email, country)
VALUES (1, 'Alice Nguyen', 'alice@example.com', 'Vietnam')
ON CONFLICT (customer_id) DO UPDATE SET
    name    = excluded.name,
    email   = excluded.email,
    country = excluded.country;
-- `excluded` refers to the row that was attempted to be inserted
-- Safe to run multiple times — result is always the same
```

### Views

A **view** is a stored query definition — not a stored copy of data. The query runs every time the view is selected.

```sql
CREATE VIEW vw_customer_summary AS
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id)   AS total_orders,
    SUM(o.total_amount) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

SELECT * FROM vw_customer_summary;  -- executes the full JOIN + aggregate every time
```

**Materialized views** store the query result as a physical snapshot — faster reads, requires explicit refresh, not supported natively in DuckDB. The concept applies in Snowflake, BigQuery, and PostgreSQL.

> [!TIP] DuckDB Materialization Pattern
> Materialize manually: `CREATE TABLE summary AS SELECT ... FROM vw_customer_summary;`
> On refresh: `DROP TABLE IF EXISTS summary; CREATE TABLE summary AS SELECT ...;`

### Temporary Tables

```sql
CREATE TEMP TABLE staging_orders AS
SELECT * FROM raw_orders WHERE status = 'pending';

-- Use in subsequent pipeline steps
SELECT * FROM staging_orders WHERE total_amount > 100;

-- Automatically dropped at end of the session — no cleanup needed
```

Temp tables are session-scoped — ideal for multi-step staging without polluting the main schema.

### TRUNCATE vs DELETE

```sql
-- DELETE: transactional, row-by-row, supports WHERE clause
DELETE FROM staging_events;                                        -- removes all rows (slow)
DELETE FROM staging_events WHERE ingested_at < '2024-01-01';       -- conditional

-- TRUNCATE: instantly empties the table, no row-by-row processing, no WHERE
TRUNCATE TABLE staging_events;
```

> [!WARNING] DML Anti-Patterns
>
> ❌ `DELETE FROM orders` with no `WHERE` clause in a production script — silently wipes everything
> ✅ Always include `WHERE` on `DELETE` in production tables; use `TRUNCATE` explicitly only for full staging table clears
>
> ❌ Using views for heavy aggregations queried thousands of times per day — the query re-runs on every `SELECT`
> ✅ Materialize the result into a physical table and refresh on a schedule
>
> ❌ `DROP TABLE orders` in a pipeline script — catastrophic if run against the wrong environment
> ✅ Use `DROP TABLE IF EXISTS staging_orders` only for staging or temp tables; protect production tables with access controls

---

## 2.3 — Query Performance

> [!NOTE] What You'll Learn
> Writing correct SQL is step one. Writing SQL that runs in seconds instead of hours is step two. This section covers how databases execute queries, what makes predicates fast or slow, and the most impactful optimisation levers available to every data engineer.

### How a Database Executes a Query

When you submit a SQL query, the database engine performs three steps:

1. **Parse** — checks syntax; validates that table and column names exist
2. **Plan** — the query optimiser generates a physical execution plan (which indexes to use, join order, scan strategy)
3. **Execute** — runs the plan and returns results

The plan is what you inspect with `EXPLAIN`.

### Reading EXPLAIN in DuckDB

```sql
EXPLAIN
SELECT customer_id, SUM(total_amount) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;

-- For execution stats (rows processed, timing):
EXPLAIN ANALYZE
SELECT customer_id, SUM(total_amount) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;
```

DuckDB renders `EXPLAIN` as a **visual tree** — each node is a step in the execution plan, read bottom-up (the deepest node executes first).

Key operators to recognise:

| Operator | Meaning |
|----------|---------|
| `SEQ_SCAN` | Sequential (full) table scan — reads every row. DuckDB embeds filter info directly inside this node (look for `Filters:` in the node body) |
| `PROJECTION` | Column selection step — trims to only the columns needed |
| `HASH_GROUP_BY` | Grouping via hash table — efficient in DuckDB |
| `HASH_JOIN` | Join via hash table — appears when two tables are joined |

> [!TIP] DuckDB vs Traditional Databases
> Unlike PostgreSQL, DuckDB does **not** show a separate `INDEX_SCAN` or `FILTER` operator in most plans. Filters are **embedded inside `SEQ_SCAN`** (shown as `Filters: order_date>='2024-01-01'` within the node). This is DuckDB's predicate pushdown in action — the filter is applied during the scan, not as a separate step.

### Indexes

An index is a separate data structure that maps column values to row locations — like a book index pointing to page numbers. Without an index, the database reads every row (**full table scan**). With an index, it jumps directly to matching rows.

**B-tree analogy:** imagine the index as a sorted binary search tree. Instead of scanning 10 million names one by one, the database bisects the tree and finds your value in ~23 steps (`log₂(10,000,000) ≈ 23`).

```sql
-- Single-column index
CREATE INDEX idx_orders_date ON orders(order_date);

-- Composite index: useful when both columns appear together in WHERE or JOIN
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

**When NOT to create an index:**

- **Low-cardinality columns** — a `status` column with 3 values doesn't filter enough rows for the index to help
- **Write-heavy staging tables** — every `INSERT` and `UPDATE` must also update the index
- **Small tables** — a full scan of 500 rows is faster than the overhead of index traversal

> [!TIP] DuckDB Indexing Context
> DuckDB is an analytical engine optimised for bulk scans. It uses internal **zone maps** (min/max statistics per row group) to skip irrelevant row groups automatically — without an explicit `CREATE INDEX`. Explicit indexes benefit point lookups on primary keys. For most range queries in DuckDB, the zone map statistics handle the optimisation automatically.

### SARGABLE Predicates

> [!IMPORTANT] What Is SARGABLE?
> **SARGABLE** = **S**earch **ARG**ument **ABLE** — a predicate the database engine *can* push down to an index or apply early in execution.
>
> **The rule:** if you apply a function or transformation to the *column side* of a predicate, no index can be used — the engine must evaluate every row. Move all transformations to the *literal (value) side*.

> [!WARNING] Top Non-SARGABLE Patterns
>
> **Pattern 1 — Function wrapped around the column:**
> ❌ `WHERE YEAR(order_date) = 2024` — wraps `order_date` in a function; the index on `order_date` is bypassed
> ✅ `WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'` — range predicate on the raw column; index is used
>
> **Pattern 2 — Leading wildcard in LIKE:**
> ❌ `WHERE customer_name LIKE '%nguyen'` — the wildcard at the start forces a scan of every value
> ✅ `WHERE customer_name LIKE 'nguyen%'` — trailing wildcard allows left-anchored index traversal
>
> **Pattern 3 — Arithmetic on the column:**
> ❌ `WHERE total_amount / 100 > 50` — transforms the column; index cannot be used
> ✅ `WHERE total_amount > 5000` — arithmetic moved to the literal; index is usable

### SARGABLE vs Non-SARGABLE in DuckDB

```sql
-- Isolated table for SARGABLE examples (separate from §2.2 tables)
CREATE TABLE orders_perf (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL
);
CREATE INDEX idx_perf_date     ON orders_perf(order_date);
CREATE INDEX idx_perf_customer ON orders_perf(customer_id);

-- ❌ Non-SARGABLE: function on column side
EXPLAIN SELECT * FROM orders_perf WHERE YEAR(order_date) = 2024;

-- ✅ SARGABLE: range predicate on raw column
EXPLAIN SELECT * FROM orders_perf WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';

-- ❌ Non-SARGABLE: arithmetic on column side
EXPLAIN SELECT * FROM orders_perf WHERE total_amount / 100 > 50;

-- ✅ SARGABLE: arithmetic on literal side
EXPLAIN SELECT * FROM orders_perf WHERE total_amount > 5000;

-- ❌ Non-SARGABLE: implicit type cast forces evaluation of every row
EXPLAIN SELECT * FROM orders_perf WHERE CAST(customer_id AS VARCHAR) LIKE '%23';

-- ✅ SARGABLE: exact match on the column's native type
EXPLAIN SELECT * FROM orders_perf WHERE customer_id = 123;
```

### Real-World Impact

> [!EXAMPLE] Uber — Non-SARGABLE at Scale
> Uber's Presto environment processes raw trip data stored as **100TB+ Parquet files**. Their engineering team built a custom Parquet reader specifically to enable predicate pushdown — the mechanism SARGABLE predicates activate. A non-SARGABLE predicate like `WHERE YEAR(trip_start) = 2023` bypasses this entirely, forcing a full scan across billions of rows. The SARGABLE equivalent — `WHERE trip_start >= '2023-01-01' AND trip_start < '2024-01-01'` — lets the reader skip irrelevant row groups using Parquet column statistics. At this scale, SARGABLE predicates are not a micro-optimisation; they determine whether a query is viable at all. *(Source: Uber Engineering — [Presto Infrastructure](https://www.uber.com/us/en/blog/presto/))*

### Other Common Performance Fixes

- **Avoid `SELECT *`** — fetches every column even when you need two; in columnar formats like Parquet this is especially expensive
- **Use `LIMIT` during exploration** — `SELECT * FROM large_table LIMIT 100` before committing to a full scan
- **Avoid `DISTINCT` as a crutch** — `DISTINCT` triggers a deduplication pass across the full result set; fix the underlying duplicate problem instead
- **Filter early** — apply `WHERE` clauses that dramatically reduce row count before `JOIN`s, not after
- **Predicate pushdown** — DuckDB and Spark automatically push filters to the earliest possible point in the execution plan; writing explicit early filters gives the optimiser clear signals

---

### JOIN Best Practices & Anti-Patterns

JOINs are where most query slowness and data bugs originate. Most problems come down to three causes: joining on the wrong columns, joining too much data, or joining on ambiguous conditions.

> [!IMPORTANT] Always Know Your JOIN Type
> `INNER JOIN` only returns rows with matches on **both** sides — unmatched rows are silently dropped.
> `LEFT JOIN` keeps all rows from the left table, filling `NULL` for unmatched right-side columns.
> Mixing them up produces results that look correct but are missing data.

**Best practices:**

- **Join on primary/surrogate keys** — these are indexed and unambiguous; avoid joining on `VARCHAR` columns like names or emails
- **Filter before you join** — pre-filter large tables in a CTE before joining them; don't rely on the optimiser to always reorder
- **Be explicit about join type** — always write `INNER JOIN`, `LEFT JOIN`, etc.; never use implicit comma syntax
- **Respect `NULL` in join keys** — `NULL = NULL` is `FALSE` in SQL; rows where the join key is `NULL` will never match

```sql
-- Pre-filter before join: reduces rows early
WITH active_customers AS (
    SELECT customer_id, name
    FROM customers
    WHERE status = 'active'           -- filter BEFORE the join
),
recent_orders AS (
    SELECT customer_id, order_id, total
    FROM orders
    WHERE order_date >= '2024-01-01'  -- filter BEFORE the join
)
SELECT c.name, o.order_id, o.total
FROM active_customers c
INNER JOIN recent_orders o ON c.customer_id = o.customer_id;
```

> [!WARNING] Common JOIN Anti-Patterns
>
> **❌ Implicit / comma JOIN** — produces a Cartesian product if you forget the condition:
> ```sql
> -- Bad: if WHERE is missing or wrong, this returns millions of rows
> SELECT * FROM orders, customers WHERE orders.cid = customers.id;
> ```
> ✅ Use explicit `INNER JOIN ... ON` syntax always.
>
> ---
>
> **❌ JOIN on a function or expression** — this is the JOIN version of non-SARGABLE:
> ```sql
> -- Bad: forces full scan + evaluation on every row combination
> ON YEAR(a.order_date) = YEAR(b.event_date)
> ```
> ```sql
> -- Good: join on the raw column; filter with a range predicate
> ON a.order_date = b.event_date
> WHERE a.order_date >= '2024-01-01'
> ```
>
> ---
>
> **❌ Multiple LEFT JOINs on one-to-many relationships** — silently multiplies rows:
> ```sql
> -- If a customer has 3 orders AND 2 addresses, result has 3×2=6 rows per customer
> SELECT * FROM customers
> LEFT JOIN orders    ON customers.id = orders.customer_id
> LEFT JOIN addresses ON customers.id = addresses.customer_id;
> ```
> ✅ Aggregate or deduplicate one-to-many sides in a CTE before joining.
>
> ---
>
> **❌ Using `DISTINCT` to hide a broken JOIN** — masks the root cause (duplicate rows from a bad join condition):
> ```sql
> -- "It works now" but DISTINCT is expensive and hides the real problem
> SELECT DISTINCT c.name FROM customers c LEFT JOIN orders o ON c.id = o.cid;
> ```
> ✅ Fix the join condition or pre-aggregate instead.

---

## 2.4 — Database Design & Normalization

> [!NOTE] What You'll Learn
> Normalization is the process of organising tables to eliminate data anomalies. Understanding it helps you design OLTP schemas correctly — and explains *why* analytics schemas intentionally break all the rules.

### Why Normalization?

Without normalization, embedding everything into one flat table creates three classes of **data anomaly**:

- **Insert anomaly** — you cannot add a product without attaching it to an order
- **Update anomaly** — changing a customer's email requires updating it in every single order row; miss one and the data is inconsistent
- **Delete anomaly** — deleting the last order for a customer also destroys the customer's record

The solution: separate entities into their own tables and link them with foreign keys.

### Walking Through Normal Forms

**Unnormalized (0NF) — all data in one flat table:**

| order_id | customer_name | customer_email | customer_city | product_name | product_price | quantity |
|----------|--------------|----------------|--------------|-------------|--------------|---------|
| 1 | Alice | alice@ex.com | Hanoi | Laptop | 999.00 | 1 |
| 1 | Alice | alice@ex.com | Hanoi | Mouse | 29.00 | 2 |
| 2 | Bob | bob@ex.com | HCMC | Laptop | 999.00 | 1 |

Problems: customer info repeats on every order line; product info repeats across orders.

**1NF — Atomic values, no repeating groups:**
Each column holds one indivisible value. No comma-separated lists or arrays crammed into a single cell (e.g., no `"Laptop, Mouse"` in a `products` field). The table above already satisfies 1NF.

**2NF — No partial dependencies** (relevant when the primary key is composite):
If the PK is `(order_id, product_name)`, then `customer_email` depends only on `order_id` — not the full composite key. This **partial dependency** is fixed by moving all customer columns to a separate `customers` table.

**3NF — No transitive dependencies:**
A transitive dependency: PK → column A → column B (A determines B, but B is not directly about the PK). If `customer_city` is always determined by `customer_id`, it belongs in `customers`, not repeated in `orders`.

### 3NF Schema in DuckDB

```sql
-- One entity per table
CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    name        VARCHAR NOT NULL,
    email       VARCHAR UNIQUE NOT NULL,
    city        VARCHAR
);

CREATE TABLE products (
    product_id  INTEGER PRIMARY KEY,
    name        VARCHAR NOT NULL,
    price       DECIMAL(10,2) NOT NULL CHECK (price > 0)
);

CREATE TABLE orders (
    order_id    INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    order_date  DATE NOT NULL DEFAULT CURRENT_DATE
);

-- Junction table: breaks the many-to-many between orders and products
CREATE TABLE order_items (
    order_id   INTEGER NOT NULL REFERENCES orders(order_id),
    product_id INTEGER NOT NULL REFERENCES products(product_id),
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,  -- price snapshot at time of order
    PRIMARY KEY (order_id, product_id)
);
```

### Relationships

**One-to-many** — one customer can have many orders; each order belongs to one customer:

```
customers (1) ──────── (many) orders
```

**Many-to-many** — one order contains many products; one product appears in many orders:

```
orders (many) ──── order_items ──── (many) products
```

The **junction table** (`order_items`) decomposes the many-to-many into two clean one-to-many relationships.

### Constraints as Data Quality

Constraints are your first line of defence against bad data — enforced by the database engine before any application code runs. Define them inline in `CREATE TABLE`:

```sql
-- All constraints declared at table creation time
CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    name       VARCHAR NOT NULL,
    price      DECIMAL(10,2) NOT NULL CHECK (price > 0)  -- business rule enforced by DB
);

CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    email       VARCHAR UNIQUE NOT NULL,                  -- no duplicate registrations
    name        VARCHAR NOT NULL
);
```

> [!NOTE] DuckDB Limitation
> `ALTER TABLE ... ADD CONSTRAINT` is not yet supported in DuckDB. In production databases (PostgreSQL, MySQL, Snowflake), you can add constraints after table creation:
> ```sql
> -- Standard SQL (PostgreSQL / Snowflake) — not DuckDB
> ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);
> ALTER TABLE customers ADD CONSTRAINT uq_email UNIQUE (email);
> ```
> In DuckDB, always define constraints inside `CREATE TABLE`.

### Normalize vs Denormalize

> [!IMPORTANT] The Core Trade-off
> **Normalize for writes (OLTP):** 3NF minimises data duplication → fewer anomalies → consistent, safe updates.
> **Denormalize for reads (OLAP):** Pre-joined wide tables eliminate expensive runtime joins → faster analytical queries.
>
> This is the fundamental tension between operational databases and analytics warehouses — and the transition interns will build in [[D4 - Batch Processing & ETL]].

**PostgreSQL** is the most common OLTP database interns will encounter in real jobs. It enforces constraints strictly and is optimised for transactional writes. You won't set one up in this roadmap, but recognise it when you see it.

### Real-World Example

> [!EXAMPLE] Airbnb — OLTP to OLAP Transition
> Airbnb's transactional systems (listings, bookings, users) are heavily normalised — separate tables per entity with FK relationships — to prevent anomalies when a host updates their listing. But when that data flows into their analytics warehouse (**Minerva**, Airbnb's metrics platform), it's deliberately denormalised into wide flat tables (one row per booking with all listing and user attributes pre-joined). Minerva standardises metric definitions on top of these pre-joined source-of-truth tables so that "bookings by country" means the same thing across every team. This is exactly the OLTP → OLAP transition interns will build in [[D4 - Batch Processing & ETL]]. *(Source: Airbnb Engineering — Minerva, Spark+AI Summit 2021)*

> [!WARNING] Normalization Anti-Patterns
>
> ❌ Over-normalising analytics tables — splitting `dim_customer` into five sub-tables in an OLAP schema adds joins with no benefit
> ✅ Reserve 3NF for transactional systems; denormalise analytics tables deliberately
>
> ❌ Skipping FK constraints to "keep the schema simple" — referential integrity errors silently corrupt downstream analytics
> ✅ Enforce FKs in OLTP schemas; they catch broken references before they propagate
>
> ❌ Allowing `NULL` in primary key columns
> ✅ PKs must always be `NOT NULL` — nullable PKs make rows unaddressable and break joins

---

## 2.5 — Dimensional Modeling

> [!NOTE] What You'll Learn
> Dimensional modeling is the standard approach for designing analytics schemas. It deliberately trades normalization for query simplicity and speed. Ralph Kimball defined this methodology; it remains the industry standard for data warehouses.

### Fact Tables vs Dimension Tables

| | Fact Table | Dimension Table |
|---|---|---|
| Contains | Measurements and events | Descriptive context |
| Examples | `fact_plays`, `fact_orders`, `fact_clicks` | `dim_user`, `dim_product`, `dim_date` |
| Row count | Billions — grows continuously | Thousands to millions — relatively stable |
| Columns | Metrics + foreign keys to dims | Descriptive attributes (names, categories, flags) |
| Metrics | Additive — can be summed across any dimension | Non-additive |

Fact table metrics should be **additive across all dimensions**: you can sum `play_duration` by user, by track, by date, or any combination.

### Star Schema vs Snowflake Schema

**Star schema** — fact at the centre, dimensions one join away. Simple, fast:

```
              dim_user
                  |
 dim_date ── fact_plays ── dim_track
```

**Snowflake schema** — dimension tables are further normalised into sub-dimensions. Saves storage at the cost of more joins:

```
              dim_user
                  |
 dim_date ── fact_plays ── dim_track ── dim_artist
                                            |
                                         dim_genre
```

In practice, **star schema is preferred** for most analytics workloads. Joins are cheap in modern columnar warehouses, and query simplicity matters far more than storage efficiency.

### Defining the Grain

> [!IMPORTANT] Define the Grain Before Any Columns
> The **grain** is the single most important design decision in dimensional modeling. It answers: *what does one row in this fact table represent?*
>
> Define the grain before adding any columns. Every column in the fact table must be true at that grain. Mixing grains in one table — for example, storing both individual plays and daily roll-ups in the same `fact_plays` — corrupts every aggregation.

Examples:
- `fact_plays` grain: **one row per individual song play**
- `fact_orders` grain: **one row per order line item** (not per order — line-item grain gives product-level detail)
- `fact_daily_sessions` grain: **one row per user per day**

### Surrogate Keys vs Natural Keys

| | Natural Key | Surrogate Key |
|---|---|---|
| Definition | Real-world identifier (`user_id = 'alice@email.com'`) | Artificial integer (`user_sk = 1001`) |
| Stability | Can change — emails change, IDs get recycled | Never changes — assigned once at insert |
| SCD Type 2 | One natural key maps to multiple surrogate keys (one per version) | Enables full history tracking |
| Performance | String joins are slower | Integer joins are faster |

**Always use surrogate keys in dimension tables.**

### Slowly Changing Dimensions (SCD)

Real-world attributes change over time. SCD types define how to handle those changes.

| Type | Strategy | History Preserved | Complexity | When to Use |
|------|----------|:-----------------:|:----------:|-------------|
| Type 0 | Immutable — never allow changes | N/A | Minimal | Date dimension, country codes |
| Type 1 | Overwrite the current value | ❌ None | Low | History doesn't matter (e.g., typo fix) |
| Type 2 | Insert a new row on change | ✅ Full | Medium | **Default choice** — user location, product category |
| Type 3 | Add a `prev_value` column | ⚠️ Last change only | Low | Rarely used — only one prior value is tracked |

> [!IMPORTANT] Type 2 Is the Default
> SCD Type 2 preserves full history by inserting a new row with the updated values and marking the old row as expired. This lets analysts answer "what was this user's country when they made this purchase in 2022?" — even if the user moved years ago.

**Type 2 row structure:**

| Column | Purpose |
|--------|---------|
| `customer_sk` | Surrogate key — unique per row version |
| `customer_id` | Natural key — same across all versions of one customer |
| `effective_date` | When this version became active |
| `expiry_date` | When this version was superseded (`NULL` = still the current record) |
| `is_current` | Fast filter: `WHERE is_current = TRUE` |

### DuckDB Examples

**Example 1 — Star schema DDL**

```sql
CREATE TABLE dim_user (
    user_sk  BIGINT PRIMARY KEY,
    user_id  VARCHAR NOT NULL,
    username VARCHAR NOT NULL,
    country  VARCHAR,
    tier     VARCHAR CHECK (tier IN ('free', 'premium'))
);

CREATE TABLE dim_track (
    track_sk BIGINT PRIMARY KEY,
    track_id VARCHAR NOT NULL,
    title    VARCHAR NOT NULL,
    artist   VARCHAR NOT NULL,
    genre    VARCHAR
);

CREATE TABLE dim_date (
    date_sk     INTEGER PRIMARY KEY,  -- e.g. 20240115 (YYYYMMDD integer)
    full_date   DATE NOT NULL,
    year        INTEGER,
    quarter     INTEGER,
    month       INTEGER,
    day_of_week VARCHAR
);

CREATE TABLE fact_plays (
    play_sk           BIGINT PRIMARY KEY,
    user_sk           BIGINT NOT NULL REFERENCES dim_user(user_sk),
    track_sk          BIGINT NOT NULL REFERENCES dim_track(track_sk),
    date_sk           INTEGER NOT NULL REFERENCES dim_date(date_sk),
    play_duration_sec INTEGER,
    is_complete       BOOLEAN NOT NULL DEFAULT FALSE
);
```

**Example 2 — SCD Type 2 table DDL**

```sql
CREATE TABLE dim_customer (
    customer_sk    BIGINT PRIMARY KEY,   -- surrogate key: unique per row version
    customer_id    VARCHAR NOT NULL,     -- natural key: same across all versions
    name           VARCHAR NOT NULL,
    country        VARCHAR NOT NULL,
    effective_date DATE NOT NULL,
    expiry_date    DATE,                 -- NULL = this is the current active record
    is_current     BOOLEAN NOT NULL DEFAULT TRUE
);
```

**Example 3 — SCD Type 2 update when a customer changes country**

```sql
-- Step 1: expire the current record
UPDATE dim_customer
SET
    expiry_date = CURRENT_DATE,
    is_current  = FALSE
WHERE customer_id = 'CUST-001'
  AND is_current  = TRUE;

-- Step 2: insert the new version as a fresh row
-- NOTE: MAX+1 is for illustration only — in production use a SEQUENCE or
--       BIGINT GENERATED ALWAYS AS IDENTITY to prevent race conditions on concurrent inserts
INSERT INTO dim_customer (
    customer_sk, customer_id, name, country, effective_date, expiry_date, is_current
)
SELECT
    (SELECT COALESCE(MAX(customer_sk), 0) + 1 FROM dim_customer),
    'CUST-001',
    'Alice Nguyen',
    'Singapore',      -- new country
    CURRENT_DATE,
    NULL,
    TRUE;
```

**Example 4 — Querying the star schema**

```sql
-- Top 5 tracks by premium users in Vietnam in Q1 2024
SELECT
    t.title,
    t.artist,
    COUNT(*)                    AS play_count,
    SUM(f.play_duration_sec)    AS total_seconds
FROM fact_plays    f
JOIN dim_user      u ON f.user_sk  = u.user_sk
JOIN dim_track     t ON f.track_sk = t.track_sk
JOIN dim_date      d ON f.date_sk  = d.date_sk
WHERE u.tier    = 'premium'
  AND u.country = 'Vietnam'
  AND d.year    = 2024
  AND d.quarter = 1
GROUP BY t.title, t.artist
ORDER BY play_count DESC
LIMIT 5;
```

### Real-World Examples

> [!EXAMPLE] Spotify — Star Schema at Scale
> Spotify generates billions of play events per day — one event per individual listen, confirmed by their public event delivery infrastructure. Dimensions (user, track, artist, album) are **slowly-changing** lookup tables — updated far less frequently than the event stream. (Note: Spotify's `dim_user` and `dim_track` tables are themselves large — hundreds of millions of rows — but their write frequency is low compared to billions of daily plays.) A query like "top tracks by premium users in Southeast Asia in Q1" is a single fact-to-dimension join — star schema makes this fast and the SQL stays readable regardless of scale. *(Source: Spotify Engineering — [Event Delivery](https://engineering.atspotify.com/2016/02/spotifys-event-delivery-the-road-to-the-cloud-part-i/))*

> [!EXAMPLE] Meta — SCD Type 2 for User History
> Tracking historical dimension changes with SCD Type 2 is the standard practice in large-scale data warehouses — and Meta's Hive-based warehouse (one of the world's largest Hive deployments) is a canonical example of this pattern in practice. When a user changes their country, a new row is inserted with the updated value and the change is timestamped. This lets analysts answer "what country was this user in when they made this purchase in 2022?" — even if the user moved years later. The specific mechanism mirrors the SCD Type 2 pattern defined by the Kimball Group and is representative of how any major warehouse (Meta, Google, Amazon) would handle this. *(Meta's internal implementation details are not publicly documented; this reflects the industry-standard approach for Hive-scale warehouses.)*

> [!WARNING] Dimensional Modeling Anti-Patterns
>
> ❌ Mixing grains in one fact table — storing both individual plays and daily summaries in `fact_plays` corrupts every aggregation
> ✅ One fact table per grain; build a separate `fact_daily_plays` if pre-aggregated summaries are needed
>
> ❌ Using natural keys (emails, product codes) as dimension primary keys — natural keys change; old fact rows then point to nothing
> ✅ Always generate surrogate integer keys for all dimension tables
>
> ❌ SCD Type 1 on a dimension where analysts need historical answers — a user's country is overwritten and the old value is gone
> ✅ Default to SCD Type 2; use Type 1 only for genuine corrections (typos, data entry fixes) where history is truly irrelevant

---

## 2.6 — Data Types & Type Safety

> [!NOTE] What You'll Learn
> Choosing the right data type prevents silent bugs, precision loss, and pipeline failures. Modern analytical engines extend SQL with richer types — nested structures and arrays are first-class citizens, not workarounds.

### Traditional SQL Types

**Numeric types:**

| Type | Range / Notes | Use For |
|------|---------------|---------|
| `SMALLINT` | −32,768 to 32,767 | Small counters |
| `INTEGER` / `INT` | ±2.1 billion | Standard IDs and counts |
| `BIGINT` | ±9.2 × 10¹⁸ | Large event counters, user IDs at scale |
| `DECIMAL(p, s)` | Exact — `p` total digits, `s` decimal places | **Money, rates** — always use for financial values |
| `FLOAT` / `DOUBLE` | Approximate — binary floating-point | Scientific computation only — **never money** |

**String types:**

| Type | Notes |
|------|-------|
| `VARCHAR(n)` | Variable-length, max `n` characters — enforces an upper bound |
| `TEXT` | Unlimited length — simpler, but no length constraint |

Use `VARCHAR(n)` when the column has a meaningful maximum length (e.g., `country_code VARCHAR(2)`). Use `TEXT` for open-ended strings like descriptions.

**Date and time types:**

| Type | Notes |
|------|-------|
| `DATE` | Calendar date only: `2024-01-15` |
| `TIMESTAMP` | Date + time, **no timezone**: `2024-01-15 14:30:00` |
| `TIMESTAMPTZ` | Date + time + timezone — **essential in distributed systems** |

**Other common types:** `BOOLEAN` (`TRUE` / `FALSE` / `NULL`), `UUID` (128-bit universally unique identifier).

> [!WARNING] The Timezone Trap
>
> ❌ `TIMESTAMP` in a globally distributed system — `2024-01-15 14:30:00` is ambiguous: 14:30 *where*? Hanoi? London? UTC?
> ✅ Use `TIMESTAMPTZ` for all event and log timestamps. Store in UTC at write time; convert to local timezone at display time only.

### Analytical / Big-Data Types (DuckDB + Spark)

Modern analytical engines extend SQL with richer types for complex, nested data:

| Type | Description | Example Use Case |
|------|-------------|-----------------|
| `HUGEINT` / `INT128` | Beyond `BIGINT` — up to ~1.7 × 10³⁸ | Astronomical-scale counters |
| `DECIMAL(38, 10)` | Maximum precision decimal | Financial aggregations at extreme scale |
| `LIST` / `VARCHAR[]` | Array of repeated values in one column | Product tags, user interest categories |
| `STRUCT` | Named nested record | Address, geolocation as a single column |
| `MAP` | Dynamic key-value pairs | Event properties, feature flags, metadata |
| `TIMESTAMPTZ` | Timezone-aware timestamp | All event tables in global systems |

### Traditional vs Analytical Type Patterns

| Scenario | Traditional Approach | Analytical Approach |
|---------|---------------------|---------------------|
| Store user tags | Junction table `user_tags` (extra join) | `tags VARCHAR[]` — one column, no join needed |
| Store address | 4 separate columns (`street`, `city`, …) | `address STRUCT(street VARCHAR, city VARCHAR, country VARCHAR)` |
| Store dynamic metadata | Fixed schema columns (schema change per new property) | `metadata MAP(VARCHAR, VARCHAR)` — schema-free extension |
| Financial amounts | `DECIMAL(10,2)` | `DECIMAL(19,4)` for higher precision at scale |

### DuckDB Examples

**Example 1 — FLOAT vs DECIMAL precision trap**

```sql
-- Float is an approximation
SELECT 0.1::FLOAT + 0.2::FLOAT AS result;
-- Returns: 0.30000000000000004  (not 0.3!)

-- Decimal is exact
SELECT 0.1::DECIMAL(10,2) + 0.2::DECIMAL(10,2) AS result;
-- Returns: 0.30  (exact)

-- Real impact: sum 1,000 transactions of $9.99
SELECT SUM(price) AS float_sum
FROM (SELECT 9.99::FLOAT AS price FROM range(1000)) t;
-- May not equal exactly 9990.00

SELECT SUM(price) AS decimal_sum
FROM (SELECT 9.99::DECIMAL(10,2) AS price FROM range(1000)) t;
-- Exactly 9990.00 — every time
```

**Example 2 — LIST / ARRAY column**

```sql
CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    name       VARCHAR NOT NULL,
    tags       VARCHAR[]   -- array of category tags
);

INSERT INTO products VALUES
    (1, 'Laptop',   ['electronics', 'computers', 'portable']),
    (2, 'Notebook', ['stationery', 'office']),
    (3, 'USB Hub',  ['electronics', 'accessories', 'portable']);

-- Unnest to query individual tag values
SELECT product_id, name, unnest(tags) AS tag
FROM products;

-- Filter products containing a specific tag
SELECT * FROM products
WHERE list_contains(tags, 'electronics');
```

**Example 3 — STRUCT column**

```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    name    VARCHAR NOT NULL,
    address STRUCT(street VARCHAR, city VARCHAR, country VARCHAR)
);

INSERT INTO users VALUES
    (1, 'Alice', {'street': '123 Main St', 'city': 'Hanoi',     'country': 'Vietnam'}),
    (2, 'Bob',   {'street': '45 Orchard',  'city': 'Singapore', 'country': 'Singapore'});

-- Access nested fields with dot notation
SELECT user_id, name, address.city, address.country
FROM users;
```

**Example 4 — NULL semantics in COUNT, SUM, and JOIN**

```sql
CREATE TABLE sales (id INTEGER, amount DECIMAL(10,2));
INSERT INTO sales VALUES (1, 100.00), (2, NULL), (3, 200.00);

-- COUNT(*) counts rows — includes the NULL row
-- COUNT(col) counts non-NULL values only
SELECT
    COUNT(*)      AS total_rows,     -- 3
    COUNT(amount) AS non_null_rows   -- 2
FROM sales;

-- SUM silently ignores NULLs — returns 300.00, not an error
SELECT SUM(amount) FROM sales;

-- NULL in WHERE: = NULL never matches; IS NULL does
SELECT * FROM sales WHERE amount = NULL;     -- ❌ returns 0 rows
SELECT * FROM sales WHERE amount IS NULL;    -- ✅ returns the NULL row

-- NULL from a LEFT JOIN: customers with no orders produce NULL order_id
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
-- customers with no orders: order_id = NULL (join NULL, not data NULL)
```

**Example 5 — COALESCE and NULLIF**

```sql
-- COALESCE: return the first non-NULL value in the list
SELECT
    id,
    COALESCE(amount, 0) AS amount_safe   -- replace NULL with 0 for downstream arithmetic
FROM sales;

-- NULLIF: return NULL if two values are equal — classic division-by-zero guard
SELECT
    customer_id,
    total_revenue / NULLIF(order_count, 0) AS avg_order_value
FROM customer_summary;
-- When order_count = 0: NULLIF returns NULL → the division returns NULL, not an error
```

### NULL Semantics Summary

> [!IMPORTANT] NULL Is Unknown, Not Zero
> `NULL` means the value is **unknown** — not zero, not empty string, not false.
> - `NULL = NULL` evaluates to `NULL` (unknown) — **never** `TRUE`
> - `COUNT(*)` counts rows and includes `NULL` rows; `COUNT(col)` skips `NULL` values
> - `SUM(col)` ignores `NULL` rows silently — this is expected SQL behaviour
> - Always use `IS NULL` / `IS NOT NULL` to test for null; `= NULL` never matches

### Implicit Casting Hazards

Some databases silently convert types — this hides bugs and causes subtle failures:

```sql
-- Explicit CAST: intent is clear, fails loudly if conversion is impossible
INSERT INTO orders (order_id, total_amount)
VALUES (1, CAST('99.99' AS DECIMAL(10,2)));

-- DuckDB shorthand (:: operator)
INSERT INTO orders (order_id, total_amount)
VALUES (1, '99.99'::DECIMAL(10,2));

-- ❌ Implicit cast: some databases accept this silently; behaviour varies
INSERT INTO orders (order_id, total_amount) VALUES (1, '99.99');
```

Always prefer explicit `CAST` — it makes the intended type conversion visible in code reviews and fails immediately if the conversion is not valid.

> [!IMPORTANT] FLOAT vs DECIMAL for Money
> Never use `FLOAT` or `DOUBLE` for monetary values. Floating-point types are binary approximations — rounding errors accumulate at scale and represent real money. Always use `DECIMAL(p, s)`. Minimum: `DECIMAL(10,2)` for standard retail amounts; `DECIMAL(19,4)` for financial systems.

> [!WARNING] Type Safety Anti-Patterns
>
> ❌ Storing dates as strings: `order_date VARCHAR DEFAULT '2024-01-15'` — strings cannot be compared with `>` / `<`, cannot be sorted correctly, and cannot participate in date arithmetic
> ✅ Use `DATE` or `TIMESTAMP` types for all temporal values
>
> ❌ Using `FLOAT` for monetary columns — `$9.99` may be stored as `$9.989999...`; rounding errors compound at scale
> ✅ Use `DECIMAL(10,2)` at minimum; `DECIMAL(19,4)` for financial systems
>
> ❌ Marking every column as nullable "just in case" — `NULL`s propagate silently through joins and aggregations, causing invisible data quality issues
> ✅ Default to `NOT NULL`; allow `NULL` only when unknown is a genuinely valid real-world state for that column
>
> ❌ Using `TIMESTAMP` (no timezone) for event tables in global systems — the wall-clock time is ambiguous
> ✅ Use `TIMESTAMPTZ` for all event and log timestamps; store in UTC at write time

### Real-World Example

> [!EXAMPLE] Stripe + Uber — Precision and Flexibility at Scale
> **Stripe** solves the floating-point money problem by storing all monetary amounts as **integers in the smallest currency unit** — `$10.99` is stored as the integer `1099` (cents). This eliminates floating-point imprecision entirely. Their entire public API is typed as `Integer` for all `amount` fields. For **SQL-based financial systems** (like a data warehouse or analytics pipeline), the equivalent best practice is `DECIMAL(19,4)` — exact numeric precision, never `FLOAT`. A `FLOAT` representation of $9.99 may be stored as $9.989999999...; summing millions of transactions compounds this into real monetary error. *(Source: [Stripe API Docs — Currencies](https://docs.stripe.com/currencies), [Charges Object](https://docs.stripe.com/api/charges/object))*
>
> **Uber's** raw analytical tables in Presto are documented as highly nested — *"it is not uncommon to see more than five levels of nesting."* This nesting maps directly to `STRUCT` and `MAP` types in Parquet, allowing dynamic trip metadata (driver attributes, zone-level fields) to be stored without schema migrations. *(Source: [Uber Engineering — Presto](https://www.uber.com/us/en/blog/presto/))*

### Forward Pointer

How SQL types map to file format types — and where precision is silently lost during serialisation across Parquet, Avro, and JSON — is covered in [[D3 - Data Storage & Formats#Type Mapping & Precision Loss|D3 §3.4]].

---

## ✅ Practice Checklist

- [ ] Write a query using `ROW_NUMBER()` to rank the top 5 customers by total spend per region — one ranking window per region
- [ ] Write a multi-step CTE that stages completed orders, aggregates by customer, and filters to customers with more than 3 orders — each step as a separate named CTE
- [ ] Create a `customers` table in DuckDB and run an `INSERT ... ON CONFLICT DO UPDATE` upsert twice with the same customer ID — verify the row count stays at 1 and the values are updated
- [ ] Write SARGABLE and non-SARGABLE versions of the same date-filtered query and compare `EXPLAIN ANALYZE` output in DuckDB — note which plan shows an early filter pushdown
- [ ] Design a 3NF schema for a simple e-commerce system (customers, orders, order_items, products) and create the full DDL in DuckDB with `PRIMARY KEY`, `NOT NULL`, `CHECK`, and `FOREIGN KEY` constraints
- [ ] Build a star schema with `fact_plays`, `dim_user`, and `dim_track` in DuckDB; insert sample rows; write a query joining all three tables to find the top 3 tracks by total play duration
- [ ] Create a `products` table with a `VARCHAR[]` tags column, insert 5 rows, query using `list_contains()`, and compare `FLOAT` vs `DECIMAL(10,2)` by summing 1,000 values of `9.99` to see the precision difference

---

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| [DuckDB Window Functions](https://duckdb.org/docs/sql/window_functions) | Window function syntax, frame clauses, and DuckDB-specific examples |
| [Kimball Dimensional Modeling Techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) | Fact/dimension design, SCD types, grain definition — the canonical reference |
| [DE Roadmap — SQL Section](https://roadmap.sh/data-engineer) | SQL skill coverage and progression for data engineers |
| [Use The Index, Luke](https://use-the-index-luke.com/) | SQL indexing, SARGABLE predicates, and query performance — free and excellent |
| [DuckDB Data Types Overview](https://duckdb.org/docs/sql/data_types/overview) | DuckDB native types including LIST, STRUCT, MAP, HUGEINT |

---

## 🃏 Quick-Reference Flash Cards

**Q:** What is the key difference between a window function and `GROUP BY`?
**A:** `GROUP BY` collapses multiple rows into one row per group. A window function computes a result for each row while keeping all rows in the output — the row count is unchanged. Window functions annotate; `GROUP BY` aggregates.

---

**Q:** What is a CTE and why use it over a nested subquery?
**A:** A CTE (`WITH name AS (...)`) is a named, reusable subquery defined before the main query. It improves readability (top-to-bottom vs inside-out nesting), allows each step to be referenced by name, and makes individual steps independently testable.

---

**Q:** What is an UPSERT / MERGE, and why is it idempotent?
**A:** An UPSERT inserts a row if it doesn't exist, or updates it if it does (`INSERT ... ON CONFLICT DO UPDATE`). It's idempotent because running it twice produces the same result as running it once — no duplicate rows are created and no errors are raised.

---

**Q:** What is a SARGABLE predicate?
**A:** **S**earch **ARG**ument **ABLE** — a predicate the database can push to an index. A predicate is non-SARGABLE when a function or transformation is applied to the column side (e.g., `YEAR(date_col) = 2024`). Fix: move all transformations to the literal side (`date_col >= '2024-01-01' AND date_col < '2025-01-01'`).

---

**Q:** What are the three data anomalies that normalization prevents?
**A:** **Insert anomaly** — cannot add an entity without linking it to another; **Update anomaly** — changing a value requires updating it in multiple rows, risking inconsistency; **Delete anomaly** — deleting one record inadvertently destroys another. Normalization separates entities into their own tables to eliminate all three.

---

**Q:** What is the grain in dimensional modeling?
**A:** The grain defines what one row in a fact table represents (e.g., "one row = one song play"). It must be defined before adding any columns. Every column must be true at that grain. Mixing grains in one table corrupts every aggregation.

---

**Q:** What is the difference between `RANK()` and `DENSE_RANK()`?
**A:** Both assign the same rank to tied values. `RANK()` skips the next rank after a tie: `1, 2, 2, 4`. `DENSE_RANK()` never skips: `1, 2, 2, 3`. Use `DENSE_RANK()` for "top N" queries where gaps in the ranking sequence would exclude valid results.

---

**Q:** What is the difference between SCD Type 1 and SCD Type 2?
**A:** Type 1 overwrites the current value — no history is kept (the old value is gone). Type 2 inserts a new row for each change with `effective_date`, `expiry_date`, and `is_current` — the full history is preserved. Type 2 is the default choice when analysts need to answer historical "what was it at the time of event X?" questions.

---

**Q:** Why use `DECIMAL` instead of `FLOAT` for monetary values?
**A:** `FLOAT` is a binary approximation — `0.1 + 0.2` may not equal `0.3`. Rounding errors accumulate when summing millions of transactions and represent real money. `DECIMAL(p, s)` stores exact values. Always use `DECIMAL` for money; minimum `DECIMAL(10,2)`, use `DECIMAL(19,4)` for financial systems.

---

**Q:** What does `NULL` mean in SQL?
**A:** `NULL` means **unknown** — not zero, not empty string, not false. `NULL = NULL` evaluates to `NULL` (not `TRUE`) — always use `IS NULL` / `IS NOT NULL`. `COUNT(*)` counts all rows including nulls; `COUNT(col)` skips `NULL` values. `SUM(col)` ignores `NULL` rows silently.

---

**Q:** What is the difference between a star schema and a snowflake schema?
**A:** In a star schema, dimension tables sit one join away from the fact table — fast queries, simple SQL. In a snowflake schema, dimensions are further normalised into sub-dimensions — saves storage but requires more joins. Star schema is preferred for most analytics workloads because query simplicity outweighs marginal storage savings.

---

*Checkpoint: [[Checkpoints/CP2 - SQL Proficiency|CP2]]*

*Previous: [[D1 - Foundations & Tooling]] | Next: [[D3 - Data Storage & Formats]]*