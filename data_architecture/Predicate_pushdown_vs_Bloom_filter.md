# The Fastest Way to Process Data Is to Avoid Reading Unnecessary Data

Reading data is expensive.

Before a query can process a row, data often needs to:

```text
Be read from storage
↓
Be transferred over the network
↓
Be loaded into memory
↓
Be deserialized into a usable format
↓
Be processed
```

Therefore, one of the most effective optimization strategies in modern data systems is:

> Eliminate data that cannot contribute to the answer as early as possible.

Modern data platforms such as Spark, Iceberg, Delta Lake, and Parquet achieve this using techniques like partition pruning, predicate pushdown, column pruning, and Bloom filters.

---

# Example Dataset

Assume we have a large events table:

```sql
events(
    user_id,
    movie_id,
    event_date,
    watch_seconds,
    country,
    device,
    event_time
)
```

stored as Parquet files and partitioned by `event_date`.

---

# 1. Partition Pruning

Suppose the physical layout looks like:

```text
events/
    event_date=2026-06-01/
    event_date=2026-06-02/
    event_date=2026-06-03/
```

Now consider the query:

```sql
SELECT *
FROM events
WHERE event_date = '2026-06-02';
```

The query engine can immediately determine that only one partition is relevant:

```text
Read:
    event_date=2026-06-02

Skip:
    event_date=2026-06-01
    event_date=2026-06-03
```

This optimization is called **partition pruning**.

The key idea is that entire partitions can be eliminated before any data files are read.

In Iceberg, the same optimization exists even if the directory structure is hidden. Instead of relying on folder names, Iceberg stores partition information in metadata and uses that metadata to determine which files belong to the requested partition.

---

# 2. Predicate Pushdown

Once Spark determines which files need to be examined, it can push filter conditions closer to the storage layer.

Consider the query:

```sql
SELECT *
FROM events
WHERE user_id = 150;
```

Instead of reading all data and filtering later, Spark passes the filter:

```text
user_id = 150
```

to the Parquet reader.

This optimization is called **predicate pushdown**.

The goal is simple:

> Allow the storage reader to eliminate irrelevant data before it is read into memory.

However, the Parquet reader still needs a way to determine which parts of the file can be skipped. This is where metadata becomes important.

---

# 3. Min/Max Statistics: One Way Predicate Pushdown Works

Inside a Parquet file, metadata may contain:

```text
Row Group 1
user_id min=1
user_id max=100

Row Group 2
user_id min=101
user_id max=200
```

For the query:

```sql
WHERE user_id = 150
```

the reader can reason:

```text
Row Group 1:
150 not in [1,100]
→ Skip

Row Group 2:
150 in [101,200]
→ Read
```

The reader never reads Row Group 1 because the metadata proves that it cannot contain the requested value.

This reduces:

```text
Storage I/O
↓
Network I/O
↓
Memory Usage
↓
CPU for Deserialization
```

Min/max statistics are one of the most common mechanisms used by predicate pushdown.

---

# 4. Column Pruning

Parquet stores data **column by column** rather than row by row.

Suppose a table contains:

```text
user_id
movie_id
event_time
country
device
watch_seconds
```

and the query is:

```sql
SELECT movie_id
FROM events
WHERE user_id = 12345;
```

Notice that the query only needs:

```text
movie_id
user_id
```

It does not need:

```text
event_time
country
device
watch_seconds
```

Since Parquet stores columns separately, Spark can tell the Parquet reader:

```text
Read:
    movie_id
    user_id

Skip:
    event_time
    country
    device
    watch_seconds
```

This optimization is called **column pruning**.

### Example

Imagine each column occupies:

```text
user_id         100 MB
movie_id        100 MB
event_time      200 MB
country          50 MB
device           50 MB
watch_seconds   500 MB
```

Total:

```text
1,000 MB
```

If the query only needs:

```text
user_id
movie_id
```

Spark may read only:

```text
200 MB
```

instead of:

```text
1,000 MB
```

before any processing even begins.

---

# 5. Why Min/Max Statistics Are Sometimes Not Enough

Suppose a row group contains only:

```text
1
10
100
1000
10000
```

Metadata records:

```text
min = 1
max = 10000
```

Now consider:

```sql
WHERE user_id = 5555
```

The metadata says:

```text
5555 is between 1 and 10000
```

Therefore the reader cannot skip the row group.

However, the value 5555 does not actually exist in the data.

Min/max statistics are too coarse to detect this.

---

# 6. Bloom Filters

A Bloom filter is an additional metadata structure that helps answer a more precise question:

> Can this value possibly exist in this block of data?

Unlike a normal lookup table, a Bloom filter does not store actual values.

For example, if a row group contains:

```text
1
10
100
1000
10000
```

a Bloom filter does not store:

```text
{1, 10, 100, 1000, 10000}
```

Instead, it stores a compact hash-based bit structure that represents those values using very little space.

Conceptually:

```text
Row Group
├── Data
└── Metadata
      ├── Min/Max Statistics
      └── Bloom Filter
```

---

# How Bloom Filters Work

During data writing:

```text
Value
  ↓
Hash Function(s)
  ↓
Set Bits in Bloom Filter
```

During reading:

```text
Query Value
  ↓
Hash Function(s)
  ↓
Check Bits
```

The Bloom filter can answer:

```text
Definitely Not Present
```

or

```text
Maybe Present
```

It cannot answer:

```text
Definitely Present
```

because Bloom filters allow false positives.

---

# Bloom Filters and Predicate Pushdown

Bloom filters are another mechanism that a storage reader can use when performing predicate pushdown.

Suppose our row group contains:

```text
1
10
100
1000
10000
```

Query:

```sql
WHERE user_id = 5555
```

Min/max statistics say:

```text
5555 is within [1,10000]
```

So the reader cannot skip the row group.

However, the Bloom filter says:

```text
5555 definitely not present
```

Now the reader can skip the entire row group without reading any data pages.

This is why Bloom filters are particularly useful for equality lookups such as:

```sql
WHERE user_id = ?
WHERE account_id = ?
WHERE email = ?
```

where exact-value lookups are common.

---

# Putting It All Together

Imagine the query:

```sql
SELECT movie_id
FROM events
WHERE event_date = '2026-06-02'
  AND user_id = 12345;
```

The engine may optimize the read in multiple stages.

### Step 1: Partition Pruning

```text
Read only:
event_date=2026-06-02
```

### Step 2: Predicate Pushdown

Pass:

```text
user_id = 12345
```

to the Parquet reader.

### Step 3: Metadata-Based Elimination

The reader may use:

```text
Min/Max Statistics
Bloom Filters
```

to eliminate files, row groups, or pages that cannot contain the requested value.

### Step 4: Column Pruning

```text
Read only:
user_id
movie_id

Skip:
country
device
watch_seconds
event_time
```

### Step 5: Read Remaining Data

Only the small subset of data that might contribute to the answer is read and processed.

---

# Key Takeaway

The goal is simple:

> Eliminate data that cannot contribute to the answer as early as possible.

The major optimization strategies are:

```text
Partition Pruning
    ↓
Uses partition metadata
    ↓
Skip partitions

Predicate Pushdown
    ↓
Push filter conditions to storage reader
    ↓
Reader may use:
    • Min/Max Statistics
    • Bloom Filters
    • Dictionary Metadata
    ↓
Skip files, row groups, or pages

Column Pruning
    ↓
Read only required columns
```

The common theme across all modern analytical systems is:

> Don't spend resources reading, transferring, storing in memory, or processing data that cannot affect the final answer.
