## Handling Duplicates in Apache Spark: distinct(), dropDuplicates(), vs. Window Functions

When engineering data pipelines in Apache Spark, removing duplicate records is a daily task. However, choosing the wrong strategy can lead to unpredictable data loss or severe performance bottlenecks.
This post breaks down when to use Spark's native distinct(), dropDuplicates(), and when you must upgrade to SQL-style Window Functions (row_number() / rank()) to safely deduplicate data while keeping the exact row you want.

------------------------------
## 1. Global Deduplication: distinct()

The simplest way to remove duplicates in Spark is df.distinct().
## How it works

distinct() looks at the entire row. It only removes records if every single column matches perfectly with another row.

# Drops rows only if all columns are identical

```
df_all_unique = df.distinct()
```


* Use it only when: You need to drop exact 1:1 duplicate rows across your entire dataset.
* Avoid it when: You want to find duplicates based on a subset of columns (like a user_id) while ignoring differences in other columns (like a timestamp).

------------------------------
## 2. Subset Deduplication: dropDuplicates()
Spark provides df.dropDuplicates() to give you more control than distinct(). It allows you to specify a subset of columns to evaluate for uniqueness.
## How it works
You pass an optional list of columns that define a "duplicate." Spark hashes these columns, groups them across your cluster, and retains only one row per group.

# Drops duplicates looking only at the user_id column
```
df_deduped = df.dropDuplicates(["user_id"])
```

## The Catch: Non-Deterministic Behavior
dropDuplicates() is non-deterministic when your subset of columns doesn't uniquely identify the rest of the row. Because Spark processes data across a distributed cluster, it simply keeps the first row it encounters in a given partition.

* Use it only when: You do not care which specific row is preserved, or if the rows are completely identical across every single column.
* Avoid it when: You need to keep a specific row (e.g., the latest update or the highest transaction amount).

------------------------------
## 3. The Precision Fix: Window Functions (row_number())
If you want to identify duplicates based on certain columns but keep a specific row based on a condition (like the latest timestamp or highest price), dropDuplicates() falls short. You need to mimic the SQL RANK() or ROW_NUMBER() pattern using Spark’s Window API.
## Code Example: Keeping the Latest Record

```
from pyspark.sql import Window
from pyspark.sql import functions as F

# 1. Define the window: Group by duplicate keys, order by your selection criteria
window_spec = Window.partitionBy("user_id").orderBy(F.col("timestamp").desc())

# 2. Assign a sequential row number 
# Note: row_number() is safer than rank() here because it guarantees exactly one row gets '1'
df_with_rank = df.withColumn("row_num", F.row_number().over(window_spec))

# 3. Filter to keep only the top selected row, then drop the helper column
final_df = df_with_rank.filter(F.col("row_num") == 1).drop("row_num")

```

------------------------------
## Quick Comparison

| Feature | distinct() | dropDuplicates() | Window Functions (row_number) |
|---|---|---|---|
| Deduplication Scope | Full Row | Column Subset | Column Subset |
| Row Selection Control | ❌ None. (Rows must match 1:1) | ❌ None. Keeps an arbitrary row. | Full Control. Uses orderBy to pick the exact row. |
| Performance | ⚡ Fast. | ⚡ Faster. Uses efficient hashing. | 🐢 Slower. Requires a full data shuffle into window partitions. |
| Best Used For | Removing exact literal duplicate rows. | Subsets where any matching row will do. | Event-driven data, changelogs, and business-critical ordering. |

## Conclusion
Use distinct() or dropDuplicates() as your default for fast, basic cleaning. When business logic dictates exactly which record must survive, accept the performance trade-off and reach for Window.partitionBy().
If you want to tailor this further, let me know:

* Are you looking for ways to optimize the shuffle performance of the Window function approach?
* Do you want to add secondary sorting criteria in case of tied columns?


