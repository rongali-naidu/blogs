# Why We Moved from Hive-Style Data Lakes to Apache Iceberg?

## Introduction

Our data lake journey started about four years ago when we designed and implemented a modern data platform based on AWS Modern Data Architecture and Data Mesh principles.

One of our key design decisions was adopting a **Data Lake First** approach. Data from various source systems would first land in the data lake before being consumed by downstream systems such as data warehouses, analytics platforms, and machine learning workloads.

At that time, we deliberately chose not to adopt Open Table Formats such as Apache Iceberg. We used other custom-build alternatives to Iceberg. The decision was not because Iceberg lacked technical merit. Rather, the ecosystem around Iceberg was still evolving. Support across AWS analytical services was limited, and the industry had not yet converged on a clear winner among competing table formats such as Iceberg, Hudi, and Delta Lake.

As architects, we wanted to avoid tightly coupling our platform to a technology that might not receive broad adoption across the AWS ecosystem in the long run.

Instead, we built our data lake using an architecture familiar to many AWS implementations:


```text
Athena / Spark / Trino
           │
           ▼
     AWS Glue Catalog
           │
           ▼
   Hive-style Partitions
           │
           ▼
    Parquet Files on S3
```

Data was stored as Parquet files in Amazon S3, organized using Hive-style partition structures, cataloged in AWS Glue, and queried through engines such as Athena and Spark. The underlying Parquet files could originate from both batch and streaming ingestion pipelines.

```text
s3://sales/
   year=2026/
      month=06/
         file1.parquet
         file2.parquet
```



The architecture was simple, open, cost-effective, and worked remarkably well for several years.

However, things started to change.

AWS gradually expanded Iceberg support across its analytics ecosystem. More services began supporting Iceberg natively, reducing the operational friction of adopting an Open Table Format. The tipping point for us came when AWS introduced **S3 Tables**, a managed table service built on top of Apache Iceberg. That was a strong signal that Iceberg was becoming a strategic direction within the AWS analytics ecosystem.

The uncertainty that previously existed around long-term adoption largely disappeared.

As we started incorporating Iceberg into our designs, I found myself revisiting many assumptions I had made while building traditional Hive-style data lakes.

Questions such as:

* If Parquet already stores data efficiently, why do we need Iceberg?
* What problems exist in a Hive-partitioned data lake that Iceberg solves?
* Why do query engines perform better with Iceberg tables?
* How does Iceberg support partition evolution without rewriting historical data?
* What role does AWS Glue play once Iceberg is introduced?
* How do Athena and Spark interact with Iceberg metadata?
* How do features such as ACID transactions, schema evolution, and time travel actually work under the hood?

Answering those questions helped me realize that Iceberg is not simply another file format or storage optimization.

It is a fundamentally different metadata architecture for managing data lakes.

This article is a summary of those questions and the architectural lessons I learned while understanding why many organizations are moving from traditional Hive-style data lakes to Apache Iceberg.


# Understanding an Iceberg Table

One of the biggest misconceptions about Apache Iceberg is that it is a replacement for Parquet.

It is not.

Iceberg still uses Parquet files to store the actual data. In fact, most Iceberg implementations continue to use Parquet as the underlying storage format because of its excellent compression, columnar storage, and query performance characteristics.

Iceberg also supports other file formats such as ORC and Avro. However, to keep the discussion focused, this article assumes Parquet as the underlying data format.

The real innovation of Iceberg is not how data is stored, but how data is managed.

Iceberg introduces a metadata layer above the data files that tracks:

* Table schemas
* Partition definitions
* Snapshot history
* File-level statistics
* Data file locations

This metadata layer becomes the authoritative source of truth for the table.

A simplified Iceberg table stored in Amazon S3 might look like this:

```text
s3://sales/

├── data/
│   ├── 00001.parquet
│   ├── 00002.parquet
│   ├── 00003.parquet
│   └── ...
│
└── metadata/
    ├── v1.metadata.json
    ├── v2.metadata.json
    ├── manifest-list-100.avro
    ├── manifest-list-101.avro
    ├── manifest-a.avro
    ├── manifest-b.avro
    └── ...
```

Notice that Iceberg separates **data files** from **metadata files**.

| Component        | Purpose                                                                  |
| ---------------- | ------------------------------------------------------------------------ |
| `data/*.parquet` | Stores the actual table data                                             |
| `metadata.json`  | Stores schema, partition specifications, snapshots, and table properties |
| `manifest list`  | Tracks manifests belonging to a snapshot                                 |
| `manifest files` | Tracks data files and their statistics                                   |

When an Iceberg table is registered in AWS Glue Catalog, Glue does not store the actual table metadata. Instead, it primarily stores a pointer to the current Iceberg metadata file.

```text
Glue Catalog Table
        │
        ▼
metadata_location =
s3://sales/metadata/v2.metadata.json
```

From that entry point, Iceberg libraries embedded within Athena, Spark, or Trino can discover the current snapshot, associated manifests, and ultimately the data files that belong to the table.

Conceptually, the metadata hierarchy looks like this:

```text
Glue Catalog Table
        │
        ▼
metadata.json
        │
        ▼
Current Snapshot
        │
        ▼
Manifest List
        │
        ▼
Manifest Files
        │
        ▼
Parquet Data Files
```

Each layer has a specific responsibility.

### Glue Catalog Table

The Glue table acts primarily as a discovery mechanism. Its most important responsibility is storing the location of the current Iceberg metadata file.

### metadata.json

The metadata file acts as the table entry point.

It contains:

* Table schema
* Partition specifications
* Snapshot history
* Current snapshot pointer
* Table properties

Whenever a table change occurs (schema evolution, partition evolution, compaction, data writes, etc.), Iceberg creates a new version of this metadata file.

### Snapshot

A snapshot represents a consistent version of the table at a specific point in time.

Every successful commit creates a new snapshot.

```text
Snapshot 100
Snapshot 101
Snapshot 102
```

Readers always query a specific snapshot.

This snapshot-based architecture forms the foundation for ACID transactions, consistent reads, rollback, and time travel.

### Manifest List

Each snapshot points to a manifest list.

Think of a manifest list as an index that tells Iceberg which manifest files belong to a particular snapshot.

### Manifest Files

Manifest files contain metadata about groups of data files.

Each manifest entry contains information such as:

* Data file location
* Partition values
* Record count
* File size
* Null counts
* Column min/max values

A manifest entry conceptually looks like:

```text
File:
s3://sales/data/fileA.parquet

Partition:
event_date=2026-06-01

Record Count:
1,000,000

Min(event_date):
2026-06-01

Max(event_date):
2026-06-01
```

Notice that the manifest contains both the file location and statistics about the file. These statistics are later used by query engines for file pruning without opening every Parquet file.

### Data Files

These are the actual Parquet files containing user data.

```text
fileA.parquet
fileB.parquet
fileC.parquet
```

The data files remain relatively simple.

Most of the intelligence—schema evolution, partition tracking, snapshot management, file statistics, and query planning support—lives in the Iceberg metadata layer.


---
# Now lets understand the Problems addressed by Iceberg

# Problem 1: Directory-Based File Discovery Doesn't Scale

In a traditional Hive-style table, discovering files requires traversing storage directories.

```text
Glue
   │
   ▼
Partition Path
   │
   ▼
S3 LIST
   │
   ▼
Files
```

Imagine a partition containing 50,000 Parquet files.

Before Athena can plan the query, it must discover what files exist.

This requires S3 LIST requests.

Amazon S3 returns a maximum of 1,000 object keys per response.

Therefore:

```text
50,000 files
÷
1,000 keys per request

=
50 LIST requests
```

The engine cannot jump directly to file number 42,387.

It must repeatedly request additional pages.

Only after discovering files can planning begin.

In large tables containing millions of files, query planning itself becomes expensive.

## How Iceberg Solves It

Iceberg eliminates directory-based file discovery.

Instead, query engines start from the current snapshot and read manifest metadata.

The critical innovation is that Iceberg stores file-level statistics in manifest entries.

These statistics include:

* Partition values
* Record counts
* Null counts
* Column min values
* Column max values

Consider:

```sql
SELECT *
FROM sales
WHERE event_date='2026-06-01'
```

Manifest metadata might contain:

```text
File A
Min Date = 2026-01-01
Max Date = 2026-01-31

File B
Min Date = 2026-06-01
Max Date = 2026-06-30

File C
Min Date = 2026-07-01
Max Date = 2026-07-31
```

During query planning, Athena or Spark can immediately determine:

```text
File A → Skip
File B → Read
File C → Skip
```

The important distinction is that the query engine makes this pruning decision using statistics exposed through Iceberg metadata.

It does not need to open thousands of Parquet files and inspect their footers before pruning can occur.

This dramatically reduces planning overhead.

---

# Problem 2: Hive Partitions Couple Metadata to Folder Structure

Hive stores partition information inside directory paths.

```text
s3://sales/
   year=2026/
      month=06/
```

The directory structure itself becomes metadata.

This creates a tight coupling between:

* Physical layout
* Partition strategy
* Query planning

Changing partition strategies becomes difficult.

## How Iceberg Solves It

Iceberg treats S3 as a dumb object store.

Files can live anywhere.

```text
s3://sales/data/a1.parquet
s3://sales/data/b2.parquet
s3://sales/data/c3.parquet
```

Partition information lives entirely in metadata.

Manifest entries store partition values separately from file paths.

```text
File:
a1.parquet

Partition:
event_date=2026-06-01
```

Query engines perform partition pruning using metadata rather than directory names.

This completely decouples partitioning from storage layout.

---

# Problem 3: Partition Evolution Requires Massive Rewrites

Suppose a table originally uses:

```sql
month(event_time)
```

Years later, daily partitioning becomes more appropriate.

Traditional Hive architectures often require:

* Rewriting data
* Rebuilding folders
* Updating partition registrations

Potentially petabytes of data must be moved.

## How Iceberg Solves It

Partition specifications are stored in metadata.

For example:

```text
Spec ID 1

month(event_time)
```

Later:

```text
Spec ID 2

day(event_time)
```

When partitioning changes:

* Old data files remain untouched
* New files use the new specification
* Iceberg metadata tracks both specifications

Athena and Spark evaluate partition pruning using the appropriate specification for each file.

No historical rewrite is required.

---

# Problem 4: Readers Can Observe Partial Writes

Traditional Hive tables expose files as soon as they appear.

```text
Write File 1
Write File 2
Write File 3
...
```

A query arriving midway may see incomplete data.

## How Iceberg Solves It

Iceberg introduces snapshot-based transactions.

```text
Write Files
      │
      ▼
Create Snapshot
      │
      ▼
Atomic Commit
```

Readers move from:

```text
Snapshot 101
```

to

```text
Snapshot 102
```

instantly.

They never see a partially committed state.

---

# Problem 5: Fragile Schema Evolution and Query Failures

Schema evolution is one of the most common challenges in long-lived data lakes.

As business requirements evolve, tables frequently need to:

* Add new columns
* Drop obsolete columns
* Rename existing columns

In traditional Hive-managed tables, these changes can be surprisingly fragile. Query engines often rely on column names or physical column positions when interpreting data files. As schemas evolve, this can lead to query failures, compatibility issues, or even incorrect data being mapped to the wrong column.

For example, if an old file contains:

```text
customer_id
email
```

and a newer schema contains:

```text
customer_id
phone
```

the engine must somehow determine whether the second column represents `email` or `phone`.

Historically, this has been a common source of operational complexity in large data lakes.


## How Iceberg Solves It: Column IDs and Schema Translation

Iceberg completely abandons name-based and position-based schema matching.

Instead, every column is assigned a permanent, unique integer identifier.

```text
ID 1 → customer_id
ID 2 → email
```

These IDs are stored in Iceberg metadata and embedded into the underlying Parquet file metadata when data files are written.

Column identity is determined by the ID rather than the column name or physical position within the schema.

Consider the following schema evolution:

```text
Version 1

ID 1 → customer_id
ID 2 → email
```

Later:

```text
Version 2

ID 1 → customer_id
ID 3 → phone
```

Notice that:

* ID 1 remains unchanged
* ID 2 is retired
* ID 3 is newly introduced

The original Parquet files remain completely untouched.

### Dropping a Column

When a column is dropped:

```sql
ALTER TABLE customers
DROP COLUMN email;
```

Iceberg updates the table metadata to indicate that ID 2 is no longer part of the active schema.

Historical Parquet files still physically contain values associated with ID 2. However, during query execution, the Iceberg library interprets the table metadata and instructs the underlying file reader to ignore that column.

```text
Historical File
────────────────────
ID 1 → customer_id
ID 2 → email

Current Schema
────────────────────
ID 1 → customer_id

Result
────────────────────
customer_id
```

No data files are rewritten.

### Adding a Column

When a new column is added:

```sql
ALTER TABLE customers
ADD COLUMN phone STRING;
```

Iceberg assigns a brand-new identifier.

```text
ID 3 → phone
```

Older files naturally do not contain values for ID 3 because the column did not exist when those files were written.

During query execution, the Iceberg library acts as a schema translation layer between the query engine and the underlying Parquet files. It compares the current table schema with the schema stored in each file and resolves any differences before data is returned to the engine.

Conceptually:

```text
Current Table Schema
────────────────────
ID 1 → customer_id
ID 3 → phone

Historical File
────────────────────
ID 1 → customer_id
ID 2 → email
```

When the Iceberg library encounters a request for ID 3 while reading an older file, it recognizes that the column does not exist in that file and transparently projects NULL values for the missing field.

The query result becomes:

```text
customer_id   phone
-----------   -----
101           NULL
102           NULL
```

even though the original Parquet files were never modified.


# Problem 6: Data Lakes Were Built for Appends, Not Updates

Traditional data lakes are optimized for:

```text
INSERT
INSERT
INSERT
```

Real-world systems often require:

```text
UPDATE
DELETE
MERGE
```

Examples include:

* GDPR compliance
* CDC ingestion
* Customer corrections

## How Iceberg Solves It

Apache Iceberg introduces native row-level mutations while preserving the scalability, performance, and cost efficiency of open object storage.
At first glance, this may seem surprising because cloud object stores such as Amazon S3 do not support in-place row updates. Objects can only be created, replaced, or deleted.
It is important to understand that Iceberg does **not** modify individual rows directly within Parquet files. The underlying storage operations are still executed at the file level. Instead, Iceberg provides row-level semantics through metadata and strategic file management.

Iceberg supports two primary approaches:

### Copy-on-Write (CoW)

In the Copy-on-Write model, Iceberg identifies the data files containing the affected rows and rewrites only those files during the commit operation.

```text
Original Data File
        │
        ▼
Modify Rows
        │
        ▼
Create New Data File
        │
        ▼
Update Iceberg Metadata
```

For example, if a table contains 1,000 Parquet files and an update affects rows stored in only 3 files, Iceberg rewrites just those 3 files while leaving the remaining 997 files untouched.

This approach provides excellent read performance because readers only need to access the latest data files.

### Merge-on-Read (MoR)

For workloads with frequent updates and deletes, repeatedly rewriting data files can become expensive.

The Merge-on-Read model avoids immediate data file rewrites by recording row-level changes in separate metadata files, commonly referred to as **Delete Files**.

```text
Original Data File
        │
        ├──► Data File
        │
        └──► Delete File
```

During query execution, the engine reads both the data files and the associated delete files and merges them logically at runtime.

```text
Data File
      +
Delete File
      ↓
Merged Result Set
```

This significantly reduces write amplification and improves write throughput for update-heavy workloads.

The trade-off is that readers may need to perform additional work to merge the data and delete files during query execution.

### Choosing Between CoW and MoR

| Characteristic    | Copy-on-Write (CoW)  | Merge-on-Read (MoR)    |
| ----------------- | -------------------- | ---------------------- |
| Read Performance  | Faster               | Slightly Slower        |
| Write Performance | Slower               | Faster                 |
| Storage Overhead  | Lower                | Higher                 |
| Best For          | Read-heavy analytics | Update-heavy workloads |

By combining metadata-driven table management with these storage strategies, Iceberg brings many of the capabilities traditionally associated with data warehouses—such as UPDATE, DELETE, and MERGE—directly into the data lake while continuing to leverage open formats and low-cost object storage.


---

# Problem 7: No Built-In Time Travel

Traditional Hive tables expose only the latest version of data.

```text
Old Data
    ↓
Overwrite
    ↓
New Data
```

The previous version is often difficult to recover.

## How Iceberg Solves It

Every commit creates a snapshot.

```text
Snapshot 100
      ↓
Snapshot 101
      ↓
Snapshot 102
```

Previous snapshots remain available for:

* Auditing
* Recovery
* Debugging
* Reproducibility

This capability is commonly known as Time Travel.

---


# Challenges Introduced by Iceberg

While Iceberg solves many of the limitations of traditional Hive-style data lakes, it also introduces new operational responsibilities.

Unlike Hive tables, where most metadata is managed through partitions and the catalog, Iceberg relies heavily on snapshots, manifests, and metadata files. To operate Iceberg efficiently, teams need a solid understanding of how these metadata structures grow and how table maintenance works.

One common challenge is the **small files problem**.

Row-level updates and deletes are not performed directly on Parquet files. Instead, Iceberg often creates additional delete files and metadata entries. Similarly, streaming workloads can generate large numbers of small data files.

Over time, this can increase:

* Metadata overhead
* Query planning time
* File scan costs
* Overall query latency

To maintain performance, Iceberg tables typically require periodic maintenance operations such as:

* Data file compaction
* Manifest compaction
* Snapshot expiration
* Cleanup of orphaned data and metadata files

Snapshot expiration deserves special attention. Iceberg retains historical snapshots to support capabilities such as time travel and rollback. Without a retention strategy, old snapshots and their associated data files can accumulate over time, increasing storage costs and metadata complexity.

## AWS Options for Iceberg Maintenance

Teams managing Iceberg tables directly are typically responsible for scheduling and operating these maintenance workflows themselves.

AWS now provides additional options to simplify this operational burden.

**AWS Glue** supports Iceberg table optimization and maintenance operations, allowing teams to automate activities such as compaction and snapshot management while still retaining control over table configuration and maintenance schedules.

For teams that prefer a fully managed experience, **Amazon S3 Tables** goes a step further by automatically handling many of these maintenance activities behind the scenes.

The trade-off is control versus simplicity:

| Approach                       | Control | Operational Effort |
| ------------------------------ | ------- | ------------------ |
| Self-Managed Iceberg           | High    | High               |
| Iceberg + AWS Glue Maintenance | Medium  | Medium             |
| Amazon S3 Tables               | Lower   | Low                |

For organizations with highly specialized workloads, direct control over maintenance strategies may be valuable. For many teams, however, AWS Glue or S3 Tables can significantly reduce the operational complexity of running Iceberg at scale.

