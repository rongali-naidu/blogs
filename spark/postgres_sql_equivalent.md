# 10 Underused SQL Features — Spark SQL Edition

https://medium.com/@dataexpert/the-10-spark-interview-problems-i-kept-getting-wrong-14a1325309a5

> Spark SQL equivalents of ten "rarely taught" SQL features originally shown in PostgreSQL. Each section gives the Spark SQL version, what's supported natively vs. what needs a workaround, and gotchas that matter at scale.
> Companion file: [`pyspark-df-advanced-features.md`](pyspark-df-advanced-features.md) (same examples using the DataFrame API).
> Tested syntax targets **Spark 3.5**; features that need **Spark 4.x** or a table format (Delta/Iceberg/Hudi) are flagged.

---

## Support Matrix

| # | Feature (PostgreSQL) | Spark SQL | Spark equivalent / notes |
|---|---|---|---|
| 1 | `FILTER (WHERE …)` on aggregates | ✅ Native (3.0+) | Also `count_if()` |
| 2 | `LATERAL` joins | ✅ Native (3.2+, broadened in 3.4/3.5) | For top-N, `ROW_NUMBER()` is usually faster; `LATERAL VIEW explode` for arrays |
| 3 | `DISTINCT ON` | ❌ | `ROW_NUMBER()` filter, `max_by()`, or `max(struct(...))` |
| 4 | Window frames `ROWS/RANGE BETWEEN` | ✅ Native | `RANGE` with `INTERVAL` on date/timestamp order |
| 5 | `WITH RECURSIVE` | ✅ **Spark 4.1+** only | Earlier: `sequence()` + `explode` for series; iterative joins for hierarchies |
| 6 | `GROUPING SETS / ROLLUP / CUBE` | ✅ Native | `grouping()`, `grouping_id()` |
| 7 | `MERGE` | ✅ With **Delta / Iceberg / Hudi** tables | Not for plain Parquet/Hive tables |
| 8 | `RETURNING` | ❌ | Delta Change Data Feed / Iceberg changelog, `DESCRIBE HISTORY` metrics |
| 9 | `VALUES` as a table | ✅ Native | `UPDATE … FROM` not supported → use `MERGE` |
| 10 | `IS DISTINCT FROM` | ✅ Native | Also `<=>` (null-safe equal), `equal_null()` |

---

## Sample Data Setup

Run once to create the temp views used throughout.

```sql
CREATE OR REPLACE TEMP VIEW orders AS
SELECT * FROM VALUES
  (1, 101, 'Laptop',   'paid',     1200.00, TIMESTAMP '2026-09-01 10:00:00'),
  (2, 101, 'Mouse',    'paid',       25.00, TIMESTAMP '2026-09-05 11:00:00'),
  (3, 101, 'Monitor',  'refunded',  300.00, TIMESTAMP '2026-09-10 09:30:00'),
  (4, 101, 'Keyboard', 'paid',       80.00, TIMESTAMP '2026-09-15 14:00:00'),
  (5, 102, 'Phone',    'pending',   900.00, TIMESTAMP '2026-09-12 16:45:00'),
  (6, 102, 'Case',     'paid',       20.00, TIMESTAMP '2026-09-13 08:15:00')
AS orders(id, customer_id, title, status, total, created_at);

CREATE OR REPLACE TEMP VIEW customers AS
SELECT * FROM VALUES (101, 'Alice'), (102, 'Bob'), (103, 'Carol')
AS customers(id, name);

CREATE OR REPLACE TEMP VIEW sessions AS
SELECT * FROM VALUES
  (1, 'active',  TIMESTAMP '2026-09-20 09:00:00'),
  (1, 'expired', TIMESTAMP '2026-09-21 10:00:00'),
  (2, 'active',  TIMESTAMP '2026-09-19 12:00:00')
AS sessions(user_id, status, created_at);

CREATE OR REPLACE TEMP VIEW daily_sales AS
SELECT * FROM VALUES
  (DATE '2026-09-01', 100.0), (DATE '2026-09-02', 150.0), (DATE '2026-09-03', 90.0),
  (DATE '2026-09-04', 200.0), (DATE '2026-09-05', 120.0), (DATE '2026-09-06', 170.0),
  (DATE '2026-09-07', 130.0), (DATE '2026-09-08', 210.0)
AS daily_sales(day, revenue);

CREATE OR REPLACE TEMP VIEW employees AS
SELECT * FROM VALUES
  (1, 'CEO', CAST(NULL AS INT)), (2, 'VP Eng', 1), (3, 'VP Sales', 1),
  (4, 'Eng Manager', 2), (5, 'Engineer', 4)
AS employees(id, name, manager_id);

CREATE OR REPLACE TEMP VIEW sales AS
SELECT * FROM VALUES
  ('West', 'Widget', 100), ('West', 'Gadget', 150),
  ('East', 'Widget', 200), ('East', 'Gadget',  50)
AS sales(region, product, revenue);

CREATE OR REPLACE TEMP VIEW users AS
SELECT * FROM VALUES
  (1, 'a@x.com', 'a@x.com'),
  (2, 'b@x.com', 'b2@x.com'),
  (3, CAST(NULL AS STRING), 'c@x.com'),
  (4, 'd@x.com', CAST(NULL AS STRING)),
  (5, CAST(NULL AS STRING), CAST(NULL AS STRING))
AS users(id, old_email, new_email);
```

---

## 1. FILTER — Conditional Aggregates Without CASE

✅ **Supported natively** (Spark 3.0+).

```sql
SELECT
  COUNT(*)   FILTER (WHERE status = 'paid')     AS paid,
  COUNT(*)   FILTER (WHERE status = 'refunded') AS refunded,
  SUM(total) FILTER (WHERE created_at > current_timestamp() - INTERVAL 30 DAYS) AS last_30d_revenue
FROM orders;
```

Works with any aggregate, including `collect_list`, `avg`, `max_by`:

```sql
SELECT
  customer_id,
  collect_list(title) FILTER (WHERE status = 'paid') AS paid_items,
  AVG(total)          FILTER (WHERE status <> 'refunded') AS avg_valid_order
FROM orders
GROUP BY customer_id;
```

**Spark extras**

```sql
SELECT
  count_if(status = 'paid') AS paid,                 -- shorthand for COUNT(*) FILTER (...)
  SUM(IF(status = 'paid', total, 0)) AS paid_total   -- CASE/IF style, also fine
FROM orders;
```

**Scenario — pivot-style KPIs in one scan:** instead of three `GROUP BY` queries (one per status) joined together, a single aggregation with `FILTER` scans the table once and avoids extra shuffles.

---

## 2. LATERAL Joins — A "for loop" Inside the Query

✅ **Supported natively.** Lateral subqueries exist since Spark 3.2, and support for correlation (non-equality predicates, correlated `LIMIT`, joins inside the subquery) was broadened in 3.4/3.5.

```sql
-- Top 3 most recent orders per customer
SELECT c.name, recent.title, recent.created_at
FROM customers c
JOIN LATERAL (
    SELECT o.title, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id          -- references the outer table
    ORDER BY o.created_at DESC
    LIMIT 3
) AS recent;

-- Keep customers with zero orders (Carol)
SELECT c.name, recent.title
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.title FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC
    LIMIT 3
) AS recent ON true;
```

> Correlated `LIMIT`/`ORDER BY` inside lateral subqueries depends on your Spark version. If you get an error about an unsupported correlated subquery, use the window approach below.

**Preferred at scale — window function top-N**

Spark decorrelates lateral subqueries into joins, but the window approach is the most predictable plan. Spark 3.5 added a **window group limit** optimization that pushes `rn <= N` into the window operator, so it doesn't sort/materialize whole partitions.

```sql
SELECT c.name, o.title, o.created_at
FROM customers c
LEFT JOIN (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
  FROM orders
) o
  ON o.customer_id = c.id AND o.rn <= 3;
```

**Related: Hive-style `LATERAL VIEW`** (for arrays/maps, not correlated subqueries)

```sql
SELECT customer_id, item
FROM (SELECT customer_id, collect_list(title) AS items FROM orders GROUP BY customer_id)
LATERAL VIEW explode(items) t AS item;

-- Modern equivalent: table-valued generator in FROM
SELECT o.customer_id, t.item
FROM (SELECT customer_id, collect_list(title) AS items FROM orders GROUP BY customer_id) o,
     LATERAL explode(o.items) AS t(item);
```

**Dialect map:** PostgreSQL `LATERAL` = SQL Server `CROSS/OUTER APPLY` = Spark `JOIN LATERAL / LEFT JOIN LATERAL … ON true`.

---

## 3. DISTINCT ON — "Latest Row per Group"

❌ **Not supported.** Three Spark patterns:

**A. `ROW_NUMBER()` (general, returns full rows)**

```sql
SELECT user_id, status, created_at
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM sessions
)
WHERE rn = 1;
```

**B. `max_by()` / `min_by()` (single aggregation, no window)**

```sql
SELECT
  user_id,
  max_by(status, created_at) AS latest_status,
  max(created_at)            AS latest_at
FROM sessions
GROUP BY user_id;
```

**C. `max(struct(...))` (all columns, one aggregation)**

Structs compare field by field, so putting the ordering column first returns the whole "latest" row.

```sql
SELECT user_id, latest.status, latest.created_at
FROM (
  SELECT user_id, max(struct(created_at, status)) AS latest
  FROM sessions
  GROUP BY user_id
);
```

| Pattern | Pros | Cons |
|---|---|---|
| `ROW_NUMBER()` | Any columns, ties controllable, top-N | Window + sort per partition |
| `max_by` | Very cheap hash aggregation | One column per call; ties are non-deterministic |
| `max(struct)` | All columns in one aggregation | Ordering column must be first in the struct |

> **Gotchas**
> - Use `ROW_NUMBER` (exactly one row) vs `RANK`/`DENSE_RANK` (keeps ties) deliberately. Add a tie-breaker column (e.g., `id DESC`) for deterministic results.
> - Databricks SQL supports `QUALIFY rn = 1` to filter window results without a subquery; check whether your Spark runtime supports it before relying on it.

---

## 4. Window Frames — ROWS BETWEEN Is Where the Power Is

✅ **Supported natively.**

```sql
SELECT
  day,
  revenue,
  SUM(revenue) OVER (
      ORDER BY day
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total,
  AVG(revenue) OVER (
      ORDER BY day
      ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7_rows
FROM daily_sales;
```

**Calendar-based window (handles missing days correctly) — `RANGE` with an interval**

```sql
SELECT
  day,
  revenue,
  AVG(revenue) OVER (
      ORDER BY day
      RANGE BETWEEN INTERVAL 6 DAYS PRECEDING AND CURRENT ROW
  ) AS moving_avg_7_calendar_days
FROM daily_sales;
```

`ROWS 6 PRECEDING` = last 7 **rows**; if a day is missing, it reaches back further than 7 days. `RANGE INTERVAL 6 DAYS` = last 7 **calendar days**.

**Other frame patterns**

```sql
SELECT
  customer_id, created_at, total,
  SUM(total)  OVER (PARTITION BY customer_id ORDER BY created_at
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)           AS cust_running_total,
  SUM(total)  OVER (PARTITION BY customer_id)                                   AS cust_total,          -- whole partition
  LAG(total)  OVER (PARTITION BY customer_id ORDER BY created_at)               AS prev_order_total,
  LAST_VALUE(total) OVER (PARTITION BY customer_id ORDER BY created_at
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)   AS last_order_total
FROM orders;
```

> **Gotchas**
> - Same default as the standard: with `ORDER BY` and no frame, the frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (ties grouped). Be explicit with `ROWS` when the order column has duplicates.
> - `LAST_VALUE` without an explicit `UNBOUNDED FOLLOWING` frame returns the current row, a classic bug.
> - **Spark-specific:** a window with **no `PARTITION BY`** moves all data to a **single partition/task** (you'll see a warning). Fine for small daily aggregates; dangerous on large tables.

---

## 5. Recursive CTEs — Trees, Graphs, and Generated Series

✅ **Spark 4.1+ only.** `WITH RECURSIVE` was added in Spark 4.1 (SPARK-24497). Spark 3.x and 4.0 do not support it.

**Spark 4.1+**

```sql
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL
  UNION ALL
    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT concat(repeat('  ', depth - 1), name) AS chart, depth
FROM org_chart
ORDER BY depth;
```

Keep a terminating condition (e.g., `WHERE depth < 20`) to guard against cycles in dirty hierarchy data.

**Date series — no recursion needed (all versions)**

```sql
SELECT explode(sequence(DATE '2026-01-01', DATE '2026-01-31', INTERVAL 1 DAY)) AS d;

-- Fill gaps in a report
WITH dates AS (
  SELECT explode(sequence(DATE '2026-09-01', DATE '2026-09-10', INTERVAL 1 DAY)) AS day
)
SELECT d.day, COALESCE(s.revenue, 0) AS revenue
FROM dates d
LEFT JOIN daily_sales s ON s.day = d.day
ORDER BY d.day;
```

Also available: `range(start, end)` as a table-valued function, e.g. `SELECT * FROM range(1, 11)`.

**Hierarchies on Spark 3.x / 4.0 — fixed-depth self joins (when depth is known and small)**

```sql
SELECT e1.name AS level1, e2.name AS level2, e3.name AS level3, e4.name AS level4
FROM employees e1
LEFT JOIN employees e2 ON e2.manager_id = e1.id
LEFT JOIN employees e3 ON e3.manager_id = e2.id
LEFT JOIN employees e4 ON e4.manager_id = e3.id
WHERE e1.manager_id IS NULL;
```

For unknown depth on older versions, use an iterative DataFrame loop (see the PySpark file) or GraphFrames.

---

## 6. GROUPING SETS, ROLLUP, and CUBE — Multiple Levels in One Pass

✅ **Supported natively.**

```sql
SELECT region, product, SUM(revenue) AS revenue
FROM sales
GROUP BY GROUPING SETS (
    (region, product),   -- detail
    (region),            -- subtotal per region
    ()                   -- grand total
);

SELECT region, product, SUM(revenue) FROM sales GROUP BY ROLLUP (region, product);
SELECT region, product, SUM(revenue) FROM sales GROUP BY CUBE (region, product);
```

**Label subtotal rows with `grouping()` / `grouping_id()`**

```sql
SELECT
  CASE WHEN grouping(region)  = 1 THEN 'ALL REGIONS'  ELSE region  END AS region,
  CASE WHEN grouping(product) = 1 THEN 'ALL PRODUCTS' ELSE product END AS product,
  grouping_id(region, product) AS level,        -- 0 = detail, 1 = region subtotal, 3 = grand total
  SUM(revenue) AS revenue
FROM sales
GROUP BY ROLLUP (region, product)
ORDER BY level, region, product;
```

Spark also accepts the Hive-style syntax: `GROUP BY region, product WITH ROLLUP`.

**How Spark executes it:** an `Expand` operator emits one copy of each row per grouping set, then a single aggregation runs. One scan, but row count is multiplied by the number of sets — `CUBE` over many columns (2ⁿ sets) can explode data volume before the shuffle.

---

## 7. MERGE — Upsert, Properly

✅ **Supported with table formats that implement row-level operations:** Delta Lake, Apache Iceberg, Apache Hudi. ❌ Not on plain Parquet/ORC/Hive tables or temp views.

```sql
-- Delta or Iceberg target table
MERGE INTO inventory AS target
USING incoming_shipment AS source
   ON target.sku = source.sku
WHEN MATCHED AND source.quantity = 0 THEN
    DELETE
WHEN MATCHED THEN
    UPDATE SET target.quantity = target.quantity + source.quantity
WHEN NOT MATCHED THEN
    INSERT (sku, quantity) VALUES (source.sku, source.quantity);
```

**Full sync — delete target rows missing from the source** (Delta 2.3+/Spark 3.4+ syntax; supported in Iceberg with recent versions)

```sql
MERGE INTO dim_customer t
USING staged_customer s
  ON t.customer_id = s.customer_id
WHEN MATCHED AND t.row_hash <> s.row_hash THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

**Setup example (Iceberg on AWS Glue Catalog)**

```sql
CREATE TABLE glue_catalog.retail.inventory (sku STRING, quantity INT)
USING iceberg;
```

> **Gotchas**
> - **Duplicate source keys** fail the merge (`multiple source rows matched`). Deduplicate the source first (Section 3 patterns).
> - Add partition predicates to the `ON` clause (e.g., `AND target.dt = source.dt`) so the merge prunes files instead of scanning the whole target.
> - Iceberg: choose copy-on-write vs merge-on-read (`write.merge.mode`) based on read vs write frequency.

**No table format? Upsert pattern for plain Parquet:** full outer join of existing + incoming per partition, then `INSERT OVERWRITE` with `spark.sql.sources.partitionOverwriteMode=dynamic`.

---

## 8. RETURNING — Stop Querying Twice

❌ **Not supported.** Spark isn't an OLTP engine; DML returns no affected rows (Delta/Iceberg return operation metrics instead).

**Getting "what changed"**

```sql
-- Delta: operation metrics (rows inserted/updated/deleted)
DESCRIBE HISTORY inventory;

-- Delta Change Data Feed (enable once)
ALTER TABLE inventory SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
SELECT * FROM table_changes('inventory', 5);        -- changes since version 5

-- Iceberg: changelog view
CALL glue_catalog.system.create_changelog_view(table => 'retail.inventory');
SELECT * FROM inventory_changes;

-- Iceberg: snapshot summaries
SELECT snapshot_id, operation, summary FROM glue_catalog.retail.inventory.snapshots;
```

**Generated IDs without a second query**

```sql
-- Delta identity column (Delta 3.x / Databricks)
CREATE TABLE users_delta (
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  email STRING,
  created_at TIMESTAMP
) USING DELTA;

-- Or generate keys yourself at write time
SELECT uuid() AS id, email, current_timestamp() AS created_at FROM staged_users;
```

**"Move rows between tables" (the archive example)**

Spark has no multi-table transactions, so make it **idempotent** instead of atomic:

```sql
-- 1. Copy (idempotent via MERGE on the key)
MERGE INTO sessions_archive a
USING (SELECT * FROM sessions WHERE created_at < current_timestamp() - INTERVAL 90 DAYS) s
ON a.session_id = s.session_id
WHEN NOT MATCHED THEN INSERT *;

-- 2. Delete only what's now archived
DELETE FROM sessions
WHERE created_at < current_timestamp() - INTERVAL 90 DAYS
  AND session_id IN (SELECT session_id FROM sessions_archive);
```

If step 2 fails, re-running both steps is safe.

---

## 9. VALUES as a Table — Inline Lookups and Bulk Ops

✅ **Supported natively** (inline tables).

```sql
SELECT o.id, o.status, labels.display_name
FROM orders o
JOIN (VALUES
    ('paid',     'Paid ✅'),
    ('pending',  'Awaiting payment'),
    ('refunded', 'Refunded ↩')
) AS labels(status, display_name)
  ON o.status = labels.status;
```

Spark also allows `FROM VALUES ... AS t(cols)` without the parentheses and a `SELECT` without `FROM` for one-row tests.

**Bulk update — `UPDATE … FROM` isn't supported; use MERGE (Delta/Iceberg)**

```sql
MERGE INTO products p
USING (VALUES (1, 19.99), (2, 24.50), (3, 8.00)) AS v(id, new_price)
  ON p.id = v.id
WHEN MATCHED THEN UPDATE SET p.price = v.new_price;
```

**Alternative: literal map lookup (no join at all)**

```sql
SELECT id, status,
       map('paid', 'Paid ✅', 'pending', 'Awaiting payment', 'refunded', 'Refunded ↩')[status] AS display_name
FROM orders;
```

> Inline tables are tiny, so Spark broadcasts them automatically — no shuffle for the join.

---

## 10. IS DISTINCT FROM — Null-Safe Comparison

✅ **Supported natively**, plus two Spark shorthands.

```sql
-- ❌ misses rows where either side is NULL (ids 3 and 4)
SELECT * FROM users WHERE old_email <> new_email;

-- ✅ catches NULL → 'x', 'x' → NULL, and 'x' → 'y' (ids 2, 3, 4)
SELECT * FROM users WHERE old_email IS DISTINCT FROM new_email;

-- Null-safe equality (ids 1 and 5)
SELECT * FROM users WHERE old_email IS NOT DISTINCT FROM new_email;
SELECT * FROM users WHERE old_email <=> new_email;          -- same, Spark/MySQL operator
SELECT * FROM users WHERE equal_null(old_email, new_email); -- function form (3.4+)
```

| Expression | `'a' vs 'a'` | `'a' vs 'b'` | `NULL vs 'a'` | `NULL vs NULL` |
|---|---|---|---|---|
| `=` | true | false | NULL | NULL |
| `<>` | false | true | NULL | NULL |
| `<=>` / `IS NOT DISTINCT FROM` | true | false | false | **true** |
| `IS DISTINCT FROM` | false | true | **true** | false |

**Null-safe joins** (join keys that can be NULL)

```sql
SELECT * FROM a JOIN b ON a.k <=> b.k;
```

**Scenario — CDC / SCD2 change detection**

```sql
SELECT s.customer_id
FROM staged_customer s
JOIN dim_customer d
  ON s.customer_id = d.customer_id AND d.is_current
WHERE s.email   IS DISTINCT FROM d.email
   OR s.phone   IS DISTINCT FROM d.phone
   OR s.segment IS DISTINCT FROM d.segment;
```

Or compare a row hash. Beware that `concat_ws` **skips NULLs**, so `('a', NULL)` and `(NULL, 'a')` hash the same. Coalesce to a sentinel first:

```sql
SELECT sha2(concat_ws('||',
         coalesce(email, '<null>'),
         coalesce(phone, '<null>'),
         coalesce(segment, '<null>')), 256) AS row_hash
FROM staged_customer;
```

---

## Quick Reference

| Need | Spark SQL |
|---|---|
| Conditional aggregate | `COUNT(*) FILTER (WHERE …)`, `count_if(…)` |
| Top-N per group | `ROW_NUMBER() … WHERE rn <= N`, or `JOIN LATERAL (… LIMIT N)` |
| Latest row per group | `max_by(col, ts)`, `max(struct(ts, …))`, `ROW_NUMBER() = 1` |
| Running total / moving avg | `SUM/AVG … OVER (ORDER BY … ROWS BETWEEN …)` |
| Calendar window | `RANGE BETWEEN INTERVAL n DAYS PRECEDING AND CURRENT ROW` |
| Date series | `explode(sequence(start, end, INTERVAL 1 DAY))` |
| Hierarchy | `WITH RECURSIVE` (4.1+), else iterative joins |
| Subtotals | `GROUPING SETS / ROLLUP / CUBE` + `grouping()` |
| Upsert | `MERGE INTO` on Delta/Iceberg/Hudi |
| What changed | Delta CDF / Iceberg changelog, `DESCRIBE HISTORY` |
| Inline lookup | `JOIN (VALUES …) AS t(cols)` or `map(...)[key]` |
| Null-safe compare | `IS [NOT] DISTINCT FROM`, `<=>`, `equal_null()` |

---

## References

- [Spark SQL Reference — SELECT, joins, subqueries](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select.html)
- [Aggregate functions & FILTER clause](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-groupby.html)
- [LATERAL subquery](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-lateral-subquery.html)
- [LATERAL VIEW](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-lateral-view.html)
- [Window functions](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-window.html)
- [Inline tables (VALUES)](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-inline-table.html)
- [Built-in functions (max_by, count_if, sequence, equal_null, …)](https://spark.apache.org/docs/latest/api/sql/index.html)
- [Spark 4.1.0 release notes (recursive CTE)](https://spark.apache.org/releases/spark-release-4.1.0.html)
- [Spark 3.5.0 release notes (window group limit for top-k)](https://spark.apache.org/releases/spark-release-3-5-0)
- [Delta Lake — MERGE, Change Data Feed, identity columns](https://docs.delta.io/latest/delta-update.html)
- [Apache Iceberg — Spark writes (MERGE INTO) and procedures](https://iceberg.apache.org/docs/latest/spark-writes/)
