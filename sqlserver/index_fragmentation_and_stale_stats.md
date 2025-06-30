
# Top Database Performance Concerns: Index Fragmentation and Stale Statistics

## PART 1: Index Fragmentation

### What Is Index Fragmentation?

**Index fragmentation** happens when the logical order of index pages no longer matches the physical order on disk.

Think of it like:

* A book's pages getting out of order.
* A filing cabinet with scattered files.

This usually happens due to frequent **INSERTs**, **UPDATEs**, and **DELETEs** in tables.

[Index Fragmentation Explained](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver17)



###  How Index Fragmentation Affects Performance

* **Inefficient I/O**: SQL Server has to read more pages to get what it needs.
* **Slower queries**: Especially with **range scans** or large reads.
* **Increased CPU and memory usage**: Due to more data shuffling.


### Quick Fixes

1. **Reorganize** (`ALTER INDEX ... REORGANIZE`)

   * Lightweight, online, page-by-page cleanup
   * Best for **30–60% fragmentation**

2. **Rebuild** (`ALTER INDEX ... REBUILD`)

   * Completely rebuilds the index from scratch (can be [online or offline](https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online?view=sql-server-ver16))
   * Best for **>60% fragmentation**

[ALTER INDEX Syntax & Options](https://learn.microsoft.com/sql/t-sql/statements/alter-index-transact-sql?view=sql-server-ver16)



## PART 2: Statistics

### What Are Statistics?

**Statistics** are small, internal summary objects that tell SQL Server about **data distribution** in your tables (value frequency, ranges, nulls, etc.).

They're used by the **query optimizer** to decide:

* What indexes to use
* What join order to follow
* How to estimate row counts

[Statistics Overview](https://learn.microsoft.com/sql/relational-databases/statistics/statistics?view=sql-server-ver16)



### How Are Statistics Created?

SQL Server uses **named statistics**. Each stat has a unique name.

* When you create an index, SQL Server **automatically creates** statistics for the indexed column(s), and the stat name is the **same as the index name**.
* For **non-indexed columns**, SQL Server creates statistics **automatically** (if `AUTO_CREATE_STATISTICS` is ON) when the column is queried. These have names like `_WA_Sys_<column_id>_<object_id_hash>`.
* You can also [manually create statistics](https://learn.microsoft.com/sql/t-sql/statements/create-statistics-transact-sql?view=sql-server-ver16).

```sql
SELECT 
    s.name AS StatisticName,
    c.name AS ColumnName,
    t.name AS TableName
FROM 
    sys.stats s
JOIN 
    sys.stats_columns sc ON s.stats_id = sc.stats_id AND s.object_id = sc.object_id
JOIN 
    sys.columns c ON c.column_id = sc.column_id AND c.object_id = sc.object_id
JOIN 
    sys.tables t ON t.object_id = s.object_id
WHERE 
    t.name = 'YourTableName';
```



### How Stale Statistics Affects Performance

* **Stale statistics = bad query plans**
* Bad plans can lead to:

  * Full table scans instead of seeks
  * Wrong join methods (e.g., Nested Loop instead of Hash)
  * Memory spills, timeouts, or slowness

Even a well-indexed table can perform poorly if **statistics are outdated**.



### What Does `'ALL'`, `'INDEX'`, `'COLUMNS'` Mean?

* **`ALL`**: Update **both index statistics** and **column (non-index) statistics**
* **`INDEX`**: Update only statistics that were created **with indexes**
* **`COLUMNS`**: Update only **auto-created statistics** on **non-indexed columns**

[sp\_updatestats System Procedure](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-updatestats-transact-sql?view=sql-server-ver16)

> SQL Server auto-creates column statistics for frequently queried non-indexed columns. `'ALL'` ensures nothing is missed.



### Quick Fixes

Use `UPDATE STATISTICS`:

```sql
UPDATE STATISTICS MyTable;          -- Updates all stats
UPDATE STATISTICS MyTable MyStat;   -- Updates specific stat
```

[UPDATE STATISTICS Syntax](https://learn.microsoft.com/sql/t-sql/statements/update-statistics-transact-sql?view=sql-server-ver16)

Use `WITH FULLSCAN` or sampling:

* `FULLSCAN` = most accurate but slower
* Default = faster, uses sampling



## Why These Two Are Top Concerns for DBAs

| Concern       | Why It Matters                                 |
| ------------- | ---------------------------------------------- |
| Fragmentation | Impacts **I/O performance and read speed**     |
| Statistics    | Impacts **query plan accuracy and efficiency** |

Together, they directly affect:

* Query performance
* CPU and memory usage
* System stability and scalability



## Solution:  Ola Hallengren’s  `IndexOptimize` Automation Script

Ola Hallengren’s [`IndexOptimize`](https://github.com/olahallengren/sql-server-maintenance-solution) procedure is a **smart, automated** solution that:

* Scans all or selected databases/tables
* Checks fragmentation levels and chooses the best action
* Updates statistics **only when needed**
* Logs all actions for review or audit



## Example Setup Explained

This SQL script can be scheduled as [SQL Server Agent Job](https://learn.microsoft.com/sql/ssms/agent/sql-server-agent?view=sql-server-ver16)

```sql
EXECUTE dbo.IndexOptimize 
  @Databases = 'USER_DATABASES',
  @FragmentationLow = NULL,
  @FragmentationMedium = 'INDEX_REORGANIZE,INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
  @FragmentationHigh = 'INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
  @FragmentationLevel1 = 30,
  @FragmentationLevel2 = 60,
  @UpdateStatistics = 'ALL',
  @OnlyModifiedStatistics = 'Y';
```

### What This Does

1. **`@Databases = 'USER_DATABASES'`**
   → Targets **all user-created databases**

2. **Fragmentation Settings**:

   * **<30%**: Skip (low fragmentation = `NULL`)
   * **30–60%**: Try `REORGANIZE`, fallback to `REBUILD ONLINE`, then `REBUILD OFFLINE`
   * **≥60%**: Try `REBUILD ONLINE`, fallback to `REBUILD OFFLINE`

3. **Statistics Settings**:

   * `@UpdateStatistics = 'ALL'` → Update both index and column stats
   * `@OnlyModifiedStatistics = 'Y'` → Update only those stats that actually changed



## What You Achieve With This Setup

* Smart index maintenance (no unnecessary rebuilds)
* Minimal downtime (online rebuilds preferred)
* Efficient statistics refresh
* Better query performance with lower resource usage
* Safe for production use
* Logged history of what was optimized

