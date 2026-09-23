# 10 Underused SQL Features — PySpark DataFrame Edition

https://medium.com/@dataexpert/the-10-spark-interview-problems-i-kept-getting-wrong-14a1325309a5

> PySpark DataFrame API equivalents of ten "rarely taught" SQL features originally shown in PostgreSQL. Same examples and numbering as the Spark SQL file.
> Companion file: [`spark-sql-advanced-features.md`](spark-sql-advanced-features.md).
> Targets **PySpark 3.5**; APIs that need **Spark 4.x** or Delta/Iceberg are flagged.

---

## Support Matrix

| # | Feature (PostgreSQL) | PySpark DataFrame API |
|---|---|---|
| 1 | `FILTER (WHERE …)` | `F.count(F.when(cond, 1))`, `F.sum(F.when(cond, col))`, `F.count_if` (3.5+) |
| 2 | `LATERAL` join | `Window` + `row_number()` for top-N; `DataFrame.lateralJoin()` (**4.0+**); `explode` for arrays |
| 3 | `DISTINCT ON` | `row_number()` filter, `F.max_by` (3.3+), `F.max(F.struct(...))` — **not** `dropDuplicates` after `orderBy` |
| 4 | Window frames | `Window.rowsBetween()`, `Window.rangeBetween()` |
| 5 | `WITH RECURSIVE` | Iterative loop with `union` (+ checkpoint); `F.sequence` + `F.explode` for series; `spark.sql("WITH RECURSIVE …")` on **4.1+** |
| 6 | `GROUPING SETS / ROLLUP / CUBE` | `df.rollup()`, `df.cube()`, `df.groupingSets()` (**4.0+**), `F.grouping()`, `F.grouping_id()` |
| 7 | `MERGE` | `DeltaTable.merge()` (Delta); `spark.sql("MERGE …")` (Iceberg/Hudi); `DataFrame.mergeInto()` (**4.0+**) |
| 8 | `RETURNING` | ❌ — Delta `readChangeFeed`, `DeltaTable.history()`, Iceberg metadata tables |
| 9 | `VALUES` as a table | `spark.createDataFrame(...)` + `F.broadcast`, or `F.create_map` literal lookup |
| 10 | `IS DISTINCT FROM` | `col.eqNullSafe()`, `~col.eqNullSafe()`, `F.equal_null` (3.5+) |

---

## Sample Data Setup

```python
from datetime import date, datetime
from pyspark.sql import SparkSession, functions as F, Window

spark = SparkSession.builder.appName("advanced-df").getOrCreate()

orders = spark.createDataFrame([
    (1, 101, "Laptop",   "paid",     1200.00, datetime(2026, 9, 1, 10, 0)),
    (2, 101, "Mouse",    "paid",       25.00, datetime(2026, 9, 5, 11, 0)),
    (3, 101, "Monitor",  "refunded",  300.00, datetime(2026, 9, 10, 9, 30)),
    (4, 101, "Keyboard", "paid",       80.00, datetime(2026, 9, 15, 14, 0)),
    (5, 102, "Phone",    "pending",   900.00, datetime(2026, 9, 12, 16, 45)),
    (6, 102, "Case",     "paid",       20.00, datetime(2026, 9, 13, 8, 15)),
], "id INT, customer_id INT, title STRING, status STRING, total DOUBLE, created_at TIMESTAMP")

customers = spark.createDataFrame([(101, "Alice"), (102, "Bob"), (103, "Carol")],
                                  "id INT, name STRING")

sessions = spark.createDataFrame([
    (1, "active",  datetime(2026, 9, 20, 9, 0)),
    (1, "expired", datetime(2026, 9, 21, 10, 0)),
    (2, "active",  datetime(2026, 9, 19, 12, 0)),
], "user_id INT, status STRING, created_at TIMESTAMP")

daily_sales = spark.createDataFrame(
    [(date(2026, 9, d), r) for d, r in
     [(1, 100.0), (2, 150.0), (3, 90.0), (4, 200.0), (5, 120.0), (6, 170.0), (7, 130.0), (8, 210.0)]],
    "day DATE, revenue DOUBLE")

employees = spark.createDataFrame(
    [(1, "CEO", None), (2, "VP Eng", 1), (3, "VP Sales", 1), (4, "Eng Manager", 2), (5, "Engineer", 4)],
    "id INT, name STRING, manager_id INT")

sales = spark.createDataFrame(
    [("West", "Widget", 100), ("West", "Gadget", 150), ("East", "Widget", 200), ("East", "Gadget", 50)],
    "region STRING, product STRING, revenue INT")

users = spark.createDataFrame([
    (1, "a@x.com", "a@x.com"), (2, "b@x.com", "b2@x.com"),
    (3, None, "c@x.com"), (4, "d@x.com", None), (5, None, None),
], "id INT, old_email STRING, new_email STRING")
```

---

## 1. FILTER — Conditional Aggregates Without CASE

`F.when(cond, value)` returns NULL when the condition is false, and aggregates ignore NULLs, so this is the DataFrame equivalent of `FILTER`.

```python
result = orders.agg(
    F.count(F.when(F.col("status") == "paid", 1)).alias("paid"),
    F.count(F.when(F.col("status") == "refunded", 1)).alias("refunded"),
    F.sum(F.when(F.col("created_at") > F.current_timestamp() - F.expr("INTERVAL 30 DAYS"),
                 F.col("total"))).alias("last_30d_revenue"),
)
result.show()
```

**Spark 3.5+ shorthand and SQL-expression form**

```python
orders.agg(
    F.count_if(F.col("status") == "paid").alias("paid"),                       # 3.5+
    F.expr("sum(total) FILTER (WHERE status = 'paid')").alias("paid_total"),   # any version (3.0+)
).show()
```

**Per group, any aggregate**

```python
orders.groupBy("customer_id").agg(
    F.collect_list(F.when(F.col("status") == "paid", F.col("title"))).alias("paid_items"),
    F.avg(F.when(F.col("status") != "refunded", F.col("total"))).alias("avg_valid_order"),
).show(truncate=False)
```

> ⚠️ Use `F.count(F.when(...))`, **not** `F.sum(F.when(cond, 1).otherwise(0))` inside `count` — `F.count(F.when(cond, 1).otherwise(0))` counts every row because 0 is not NULL.

**Reusable KPI builder**

```python
statuses = ["paid", "pending", "refunded"]
orders.groupBy("customer_id").agg(
    *[F.count(F.when(F.col("status") == s, 1)).alias(f"{s}_cnt") for s in statuses]
).show()
```

(Alternatively `orders.groupBy("customer_id").pivot("status", statuses).count()` — passing the value list avoids an extra job to discover pivot values.)

---

## 2. LATERAL Joins — Top-N per Group

**Preferred: window + `row_number()`** (all versions; Spark 3.5 pushes `rn <= N` into the window operator)

```python
w = Window.partitionBy("customer_id").orderBy(F.col("created_at").desc())

top3 = (orders
        .withColumn("rn", F.row_number().over(w))
        .where(F.col("rn") <= 3)
        .drop("rn"))

# Inner-join semantics (CROSS JOIN LATERAL)
customers.join(top3, customers.id == top3.customer_id, "inner") \
         .select("name", "title", "created_at").show()

# Keep customers with zero orders (LEFT JOIN LATERAL ... ON true)
customers.join(top3, customers.id == top3.customer_id, "left") \
         .select("name", "title", "created_at").show()
```

**Spark 4.0+: native `lateralJoin`**

The right side references the left row via `.outer()`:

```python
c = customers.alias("c")
o = orders.alias("o")

recent = (o.where(F.col("o.customer_id") == F.col("c.id").outer())
           .orderBy(F.col("o.created_at").desc())
           .limit(3)
           .select("o.title", "o.created_at"))

c.lateralJoin(recent, how="left").select("c.name", "title", "created_at").show()
```

> `lateralJoin` is new in 4.0; check the API docs for your exact version. Correlated `limit`/`orderBy` support depends on the optimizer version. The window approach remains the safest choice for large data.

**Top-N as an array (one row per customer)**

```python
orders.groupBy("customer_id").agg(
    F.slice(
        F.sort_array(F.collect_list(F.struct("created_at", "title")), asc=False),
        1, 3
    ).alias("recent_3")
).show(truncate=False)
```

Good when N is small and groups are bounded. `collect_list` pulls the whole group into memory, so avoid it for huge groups.

**Arrays: `LATERAL VIEW explode` equivalent**

```python
items = orders.groupBy("customer_id").agg(F.collect_list("title").alias("items"))
items.select("customer_id", F.explode("items").alias("item")).show()
items.select("customer_id", F.explode_outer("items").alias("item"))   # keeps empty/null arrays
items.select("customer_id", F.posexplode("items").alias("pos", "item"))
```

---

## 3. DISTINCT ON — Latest Row per Group

**A. `row_number()` (full rows, deterministic with a tie-breaker)**

```python
w = Window.partitionBy("user_id").orderBy(F.col("created_at").desc(), F.col("status"))

latest = (sessions
          .withColumn("rn", F.row_number().over(w))
          .where("rn = 1")
          .drop("rn"))
latest.show()
```

**B. `max_by` (PySpark 3.3+) — cheapest for a few columns**

```python
sessions.groupBy("user_id").agg(
    F.max_by("status", "created_at").alias("latest_status"),
    F.max("created_at").alias("latest_at"),
).show()
```

**C. `max(struct)` — all columns in one hash aggregation**

```python
(sessions
 .groupBy("user_id")
 .agg(F.max(F.struct("created_at", "status")).alias("latest"))
 .select("user_id", "latest.*")
 .show())
```

> ⚠️ **Common bug:** `df.orderBy("created_at", ascending=False).dropDuplicates(["user_id"])` is **not guaranteed** to keep the latest row. `dropDuplicates` keeps an arbitrary row per key after the shuffle. Use one of the patterns above.

---

## 4. Window Frames — ROWS BETWEEN

```python
running = Window.orderBy("day").rowsBetween(Window.unboundedPreceding, Window.currentRow)
last7_rows = Window.orderBy("day").rowsBetween(-6, Window.currentRow)

daily_sales.select(
    "day", "revenue",
    F.sum("revenue").over(running).alias("running_total"),
    F.avg("revenue").over(last7_rows).alias("moving_avg_7_rows"),
).show()
```

**Calendar-based window with `rangeBetween`**

`rangeBetween` offsets apply to the numeric value of the order column, so order by epoch seconds (or day number) for time ranges:

```python
DAY = 86400
last7_days = (Window
              .orderBy(F.col("day").cast("timestamp").cast("long"))
              .rangeBetween(-6 * DAY, 0))

daily_sales.withColumn("moving_avg_7_calendar_days",
                       F.avg("revenue").over(last7_days)).show()

# Alternative: order by an integer day number
daily_sales.withColumn("day_num", F.datediff("day", F.lit("1970-01-01"))) \
    .withColumn("avg_7d", F.avg("revenue").over(Window.orderBy("day_num").rangeBetween(-6, 0))) \
    .show()
```

**Per-partition frames, LAG, LAST_VALUE**

```python
by_cust = Window.partitionBy("customer_id").orderBy("created_at")
whole_cust = Window.partitionBy("customer_id")
full_frame = by_cust.rowsBetween(Window.unboundedPreceding, Window.unboundedFollowing)

orders.select(
    "customer_id", "created_at", "total",
    F.sum("total").over(by_cust.rowsBetween(Window.unboundedPreceding, 0)).alias("cust_running_total"),
    F.sum("total").over(whole_cust).alias("cust_total"),
    F.lag("total").over(by_cust).alias("prev_order_total"),
    F.last("total").over(full_frame).alias("last_order_total"),
).show()
```

> **Gotchas**
> - Default frame with `orderBy` is `rangeBetween(unboundedPreceding, currentRow)`; ties share a value. Use `rowsBetween` explicitly when duplicates exist.
> - `F.last(...)` over an ordered window without a full frame returns the current row.
> - `Window.orderBy(...)` with **no `partitionBy`** sends all rows to one task — you'll see `WARN WindowExec: No Partition Defined for Window operation!`.

---

## 5. Recursive CTEs — Trees and Generated Series

**Date series (no recursion needed)**

```python
dates = spark.range(1).select(
    F.explode(F.sequence(F.lit("2026-09-01").cast("date"),
                         F.lit("2026-09-10").cast("date"),
                         F.expr("INTERVAL 1 DAY"))).alias("day"))

# Fill gaps in a report
dates.join(daily_sales, "day", "left") \
     .withColumn("revenue", F.coalesce("revenue", F.lit(0.0))) \
     .orderBy("day").show()
```

**Hierarchy — iterative loop (Spark 3.x / 4.0)**

```python
def org_chart(emp_df, max_depth=20):
    level = emp_df.where(F.col("manager_id").isNull()) \
                  .select("id", "name", "manager_id", F.lit(1).alias("depth"))
    result = level

    for _ in range(max_depth - 1):
        next_level = (emp_df.alias("e")
                      .join(level.alias("p"), F.col("e.manager_id") == F.col("p.id"))
                      .select("e.id", "e.name", "e.manager_id", (F.col("p.depth") + 1).alias("depth")))
        next_level = next_level.localCheckpoint()        # truncate lineage; keeps plans small
        if next_level.isEmpty():                          # 3.3+
            break
        result = result.unionByName(next_level)
        level = next_level

    return result

(org_chart(employees)
 .select(F.concat(F.repeat(F.lit("  "), F.col("depth") - 1), F.col("name")).alias("chart"), "depth")
 .orderBy("depth")
 .show(truncate=False))
```

Notes:
- Each iteration triggers a job (`isEmpty`), so this suits hierarchies with modest depth.
- **Checkpointing** (`localCheckpoint()` or `checkpoint()` with a checkpoint dir) prevents the query plan from growing with every loop iteration.
- For graph problems (paths, connected components), consider **GraphFrames**.

**Spark 4.1+: just use SQL**

```python
employees.createOrReplaceTempView("employees")
spark.sql("""
WITH RECURSIVE org_chart AS (
  SELECT id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, oc.depth + 1
  FROM employees e JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY depth
""").show()
```

---

## 6. GROUPING SETS, ROLLUP, and CUBE

```python
# ROLLUP: (region, product), (region), ()
(sales.rollup("region", "product")
      .agg(F.sum("revenue").alias("revenue"),
           F.grouping_id().alias("level"))
      .orderBy("level", "region", "product")
      .show())

# CUBE: every combination
sales.cube("region", "product").agg(F.sum("revenue").alias("revenue")).show()
```

**Label subtotal rows**

```python
(sales.rollup("region", "product")
      .agg(F.sum("revenue").alias("revenue"),
           F.grouping("region").alias("g_region"),
           F.grouping("product").alias("g_product"))
      .select(
          F.when(F.col("g_region") == 1, "ALL REGIONS").otherwise(F.col("region")).alias("region"),
          F.when(F.col("g_product") == 1, "ALL PRODUCTS").otherwise(F.col("product")).alias("product"),
          "revenue")
      .show())
```

**Custom grouping sets**

```python
# Spark 4.0+: DataFrame.groupingSets
sales.groupingSets([["region", "product"], ["region"], []], "region", "product") \
     .agg(F.sum("revenue").alias("revenue")).show()

# Any version: via SQL
sales.createOrReplaceTempView("sales")
spark.sql("""
SELECT region, product, SUM(revenue) AS revenue
FROM sales
GROUP BY GROUPING SETS ((region, product), (region), ())
""").show()
```

> `CUBE` over n columns creates 2ⁿ copies of each row (`Expand` operator) before the shuffle. Keep n small or pre-aggregate first.

---

## 7. MERGE — Upsert

**Delta Lake Python API**

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "retail.inventory")   # or DeltaTable.forPath(spark, "s3://.../inventory")
source = spark.table("staging.incoming_shipment")

(target.alias("t")
 .merge(source.alias("s"), "t.sku = s.sku")
 .whenMatchedDelete(condition="s.quantity = 0")
 .whenMatchedUpdate(set={"quantity": "t.quantity + s.quantity"})
 .whenNotMatchedInsert(values={"sku": "s.sku", "quantity": "s.quantity"})
 .execute())
```

Full sync with deletes (Delta 2.3+):

```python
(target.alias("t")
 .merge(source.alias("s"), "t.sku = s.sku")
 .whenMatchedUpdateAll()
 .whenNotMatchedInsertAll()
 .whenNotMatchedBySourceDelete()
 .execute())
```

**Iceberg / Hudi (any Spark 3.x): SQL through PySpark**

```python
source.createOrReplaceTempView("incoming_shipment")
spark.sql("""
MERGE INTO glue_catalog.retail.inventory t
USING incoming_shipment s
ON t.sku = s.sku
WHEN MATCHED AND s.quantity = 0 THEN DELETE
WHEN MATCHED THEN UPDATE SET t.quantity = t.quantity + s.quantity
WHEN NOT MATCHED THEN INSERT (sku, quantity) VALUES (s.sku, s.quantity)
""")
```

**Spark 4.0+: `DataFrame.mergeInto`** (table must be a format that supports MERGE)

```python
(source.alias("s")
 .mergeInto("glue_catalog.retail.inventory", F.expr("inventory.sku = s.sku"))
 .whenMatched(F.expr("s.quantity = 0")).delete()
 .whenMatched().update({"quantity": F.expr("inventory.quantity + s.quantity")})
 .whenNotMatched().insertAll()
 .merge())
```

> Verify the exact builder methods against the 4.x API docs; for Delta, the `DeltaTable` API remains the most widely used.

**Always deduplicate the source first**

```python
w = Window.partitionBy("sku").orderBy(F.col("received_at").desc())
source_dedup = source.withColumn("rn", F.row_number().over(w)).where("rn = 1").drop("rn")
```

**No table format? Partition-overwrite upsert**

```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

existing = spark.read.parquet("s3://bucket/inventory/").where(F.col("dt").isin(affected_dates))
merged = (existing.alias("e")
          .join(source_dedup.alias("s"), ["dt", "sku"], "full_outer")
          .select("dt", "sku",
                  F.coalesce(F.col("s.quantity"), F.col("e.quantity")).alias("quantity")))
merged.write.mode("overwrite").partitionBy("dt").parquet("s3://bucket/inventory/")
```

---

## 8. RETURNING — Getting Affected Rows

❌ No equivalent. Use table-format metadata instead.

```python
from delta.tables import DeltaTable

# Metrics from the last operation (rows inserted/updated/deleted)
dt = DeltaTable.forName(spark, "retail.inventory")
dt.history(1).select("version", "operation", "operationMetrics").show(truncate=False)

# Delta Change Data Feed (after enabling delta.enableChangeDataFeed = true)
changes = (spark.read.format("delta")
           .option("readChangeFeed", "true")
           .option("startingVersion", 5)
           .table("retail.inventory"))
changes.where("_change_type IN ('insert', 'update_postimage', 'delete')").show()

# Iceberg: snapshot summaries
spark.table("glue_catalog.retail.inventory.snapshots") \
     .select("snapshot_id", "operation", "summary").show(truncate=False)
```

**Generate IDs in the DataFrame so you already "have" them**

```python
new_users = (staged_users
             .withColumn("id", F.expr("uuid()"))
             .withColumn("created_at", F.current_timestamp()))
new_users.write.mode("append").saveAsTable("app.users")
new_users.select("id", "email").show()     # no second query; cache first if reused
```

> `F.monotonically_increasing_id()` is unique but **not consecutive** and not stable across recomputation — don't use it as a persistent surrogate key without materializing the result.

---

## 9. VALUES as a Table — Inline Lookups

**Small DataFrame + broadcast join**

```python
labels = spark.createDataFrame(
    [("paid", "Paid ✅"), ("pending", "Awaiting payment"), ("refunded", "Refunded ↩")],
    "status STRING, display_name STRING")

orders.join(F.broadcast(labels), "status", "left") \
      .select("id", "status", "display_name").show()
```

**Literal map lookup (no join)**

```python
from itertools import chain

label_map = {"paid": "Paid ✅", "pending": "Awaiting payment", "refunded": "Refunded ↩"}
mapping = F.create_map(*[F.lit(x) for x in chain(*label_map.items())])

orders.withColumn("display_name", mapping[F.col("status")]).show()
# Missing keys → NULL; wrap with F.coalesce(..., F.lit("Unknown")) for a default
```

**Bulk update from inline values (Delta)**

```python
price_updates = spark.createDataFrame([(1, 19.99), (2, 24.50), (3, 8.00)], "id INT, new_price DOUBLE")

(DeltaTable.forName(spark, "retail.products").alias("p")
 .merge(price_updates.alias("v"), "p.id = v.id")
 .whenMatchedUpdate(set={"price": "v.new_price"})
 .execute())
```

---

## 10. IS DISTINCT FROM — Null-Safe Comparison

```python
# ❌ misses NULL cases (ids 3 and 4)
users.where(F.col("old_email") != F.col("new_email")).show()

# ✅ IS DISTINCT FROM (ids 2, 3, 4)
users.where(~F.col("old_email").eqNullSafe(F.col("new_email"))).show()

# IS NOT DISTINCT FROM / <=> (ids 1 and 5)
users.where(F.col("old_email").eqNullSafe(F.col("new_email"))).show()
users.where(F.equal_null("old_email", "new_email")).show()     # 3.5+
```

**Null-safe join**

```python
a.join(b, a["k"].eqNullSafe(b["k"]), "inner")
```

**Scenario — change detection across many columns**

```python
from functools import reduce

compare_cols = ["email", "phone", "segment"]

s = spark.table("staging.customer").alias("s")
d = spark.table("dw.dim_customer").where("is_current").alias("d")

changed_cond = reduce(
    lambda acc, c: acc | ~F.col(f"s.{c}").eqNullSafe(F.col(f"d.{c}")),
    compare_cols[1:],
    ~F.col(f"s.{compare_cols[0]}").eqNullSafe(F.col(f"d.{compare_cols[0]}")),
)

changed = s.join(d, "customer_id").where(changed_cond).select("s.*")
```

**Row-hash alternative (NULL-safe)**

```python
def row_hash(cols):
    return F.sha2(F.concat_ws("||", *[F.coalesce(F.col(c).cast("string"), F.lit("<null>")) for c in cols]), 256)

staged = spark.table("staging.customer").withColumn("row_hash", row_hash(compare_cols))
```

> `concat_ws` silently **skips NULLs**, so without the `coalesce`, `("a", NULL)` and `(NULL, "a")` produce the same hash.

---

## Quick Reference

| Need | PySpark |
|---|---|
| Conditional aggregate | `F.count(F.when(cond, 1))`, `F.sum(F.when(cond, col))`, `F.count_if` |
| Top-N per group | `row_number().over(Window.partitionBy(..).orderBy(..))` + filter; `lateralJoin` (4.0+) |
| Latest row per group | `F.max_by`, `F.max(F.struct(ts, ...))`, `row_number() == 1` |
| Running total / moving avg | `Window.rowsBetween(...)` |
| Calendar window | `Window.orderBy(epoch).rangeBetween(-n*86400, 0)` |
| Date series | `F.explode(F.sequence(start, end, F.expr("INTERVAL 1 DAY")))` |
| Hierarchy | Iterative join loop + `localCheckpoint()`; SQL `WITH RECURSIVE` on 4.1+ |
| Subtotals | `df.rollup`, `df.cube`, `df.groupingSets` (4.0+), `F.grouping_id()` |
| Upsert | `DeltaTable.merge(...)`, SQL `MERGE` for Iceberg, `df.mergeInto` (4.0+) |
| What changed | `readChangeFeed`, `DeltaTable.history()`, Iceberg metadata tables |
| Inline lookup | `spark.createDataFrame` + `F.broadcast`, or `F.create_map` |
| Null-safe compare | `eqNullSafe`, `~eqNullSafe`, `F.equal_null` |

---

## References

- [PySpark API — `pyspark.sql.functions`](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html)
- [PySpark API — `Window`](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/window.html)
- [PySpark API — `DataFrame.lateralJoin` (4.0+)](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.lateralJoin.html)
- [PySpark API — `DataFrame` (rollup, cube, groupingSets, mergeInto)](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html)
- [Spark 4.1.0 release notes (recursive CTE)](https://spark.apache.org/releases/spark-release-4.1.0.html)
- [Delta Lake — Python MERGE API & Change Data Feed](https://docs.delta.io/latest/delta-update.html)
- [Apache Iceberg — Spark writes (MERGE INTO)](https://iceberg.apache.org/docs/latest/spark-writes/)
- [GraphFrames](https://graphframes.github.io/graphframes/docs/_site/index.html)
