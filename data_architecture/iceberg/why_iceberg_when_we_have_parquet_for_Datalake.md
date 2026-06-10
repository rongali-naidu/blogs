# Why We Moved from Hive-Style Data Lakes to Apache Iceberg?

## Introduction

Our data lake journey started about four years ago when we designed and implemented a modern data platform based on AWS Modern Data Architecture and Data Mesh principles.

One of our key design decisions was adopting a **Data Lake First** approach. Data from various source systems would first land in the data lake before being consumed by downstream systems such as data warehouses, analytics platforms, and machine learning workloads.

At that time, we deliberately chose not to adopt Open Table Formats such as Apache Iceberg.

The decision was not because Iceberg lacked technical merit. Rather, the ecosystem around Iceberg was still evolving. Support across AWS analytical services was limited, and the industry had not yet converged on a clear winner among competing table formats such as Iceberg, Hudi, and Delta Lake.

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

Table schemas
Partition definitions
Snapshot history
File-level statistics
Data file locations

This metadata layer becomes the authoritative source of truth for the table.

When an Iceberg table is stored in Amazon S3 and registered in AWS Glue Catalog, a simplified view of the architecture looks like this:



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
Glue table points to the S3 location of the metdata.json file. 

### metadata.json

The metadata file acts as the table entry point.

It contains:

* Table schema
* Partition specifications
* Snapshot history
* Current snapshot pointer
* Table properties

### Snapshot

A snapshot represents a consistent version of the table.

Every successful commit creates a new snapshot.

```text
Snapshot 100
Snapshot 101
Snapshot 102
```

Readers always query a specific snapshot.

This is the foundation of ACID transactions and time travel.

### Manifest List

A snapshot points to a manifest list.

Think of it as an index containing references to multiple manifests.

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

### Data Files

These are the actual Parquet files containing user data.

```text
fileA.parquet
fileB.parquet
fileC.parquet
```

The data files remain relatively dumb.

Most of the intelligence lives in Iceberg metadata.

---

# How Athena or Spark Queries an Iceberg Table

In traditional Hive tables:

```text
Athena
   │
   ▼
Glue Catalog
   │
   ▼
Partitions
   │
   ▼
Files
```

Glue stores much of the metadata required to discover files.

With Iceberg, the flow changes.

```text
Athena Query
      │
      ▼
Glue Catalog
      │
      ▼
metadata_location
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
Relevant Data Files
```

Notice the important difference.

Glue is no longer the source of truth.

Glue primarily acts as a discovery mechanism.

Its most important job is telling Athena where the current Iceberg metadata file lives.

For example:

```text
metadata_location=
s3://sales/metadata/v123.metadata.json
```

The authoritative metadata now resides inside Iceberg itself.

Once Athena reads the metadata hierarchy, it has everything needed to plan the query.

This architectural shift is the foundation for nearly every capability Iceberg provides.

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

# Problem 5: Schema Evolution Is Fragile

Traditional systems often rely on names or positions.

```text
Version 1

customer_id
email

Version 2

customer_id
phone
```

This creates compatibility challenges.

## How Iceberg Solves It

Iceberg assigns permanent column IDs.

```text
ID 1 → customer_id

ID 2 → email

ID 3 → phone
```

Identity is determined by ID rather than position.

This enables safe:

* Renames
* Adds
* Drops

without rewriting historical files.
Iceberg changes the manifiest files . It removes the dropped columns and add the new columns in the manifest files without rewriting the data file.s

---

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

Iceberg introduces row-level operations while preserving the scalability of object storage.This enables modern data warehouse workflows directly on the data lake.
But it will be important to note that the CRUD operations are still at file-level even though Iceberg simulates record-level updates through CoW or RoW approach.

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

# The Real Innovation

Most discussions about Iceberg focus on features:

* ACID transactions
* Time travel
* Schema evolution
* Partition evolution

But these features are consequences of a deeper architectural change.

Hive-style architecture:

```text
Glue
   ↓
Directories
   ↓
Files
```

Iceberg architecture:

```text
Glue
   ↓
metadata.json
   ↓
Snapshots
   ↓
Manifest Lists
   ↓
Manifest Files
   ↓
Files
```

The source of truth moves from storage directories to metadata structures.

Once that shift happens, efficient query planning, transactions, partition evolution, schema evolution, and time travel become natural outcomes of the architecture rather than isolated features.

That architectural shift is the real problem Apache Iceberg was built to solve.
