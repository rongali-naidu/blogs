# Getting Familiar with Apache Iceberg: The Basics, Examples, and the Truth About ACID

Apache Iceberg is one of the fastest-growing table formats in the data lake ecosystem. If you are a data engineer working with S3, Hadoop, or cloud object storage, chances are you’ve already come across it in conversations about **data lakes vs. lakehouses**.

In this blog, let’s break down Iceberg basics, point to useful resources, and clarify an often-misunderstood part of Iceberg: its support for **ACID transactions**. Spoiler — the “ACID” here is not the same as in a relational database engine.

---

## Why Iceberg?

If you already have your data stored in **Parquet, ORC, or Avro files**, why do you need a table format like Iceberg?

The answer lies in what these file formats **don’t** give you:

* **ACID Support**: You cannot safely run updates, deletes, or inserts directly against Parquet/ORC files while multiple readers/writers are working on them. Iceberg introduces a metadata layer that ensures **atomic snapshot-based commits**, so queries always see a consistent view of the data.
* **Schema Evolution**: Parquet allows schema changes, but managing them manually across thousands of files is painful. Iceberg handles column adds, drops, renames, and reorders gracefully.
* **Hidden Partitioning**: No more hardcoding partition columns into queries — Iceberg tracks partitioning internally.
* **Time Travel**: Query your table “as of” a previous snapshot, just like a versioned database.
* **Compatibility**: Iceberg works across engines like Spark, Trino, Flink, Presto, Athena, Snowflake, and Redshift Spectrum.




## How Iceberg Stores Data

Apache Iceberg is a table format with libraries that integrate into data processing engines (like Spark, Flink, Trino, Athena). These libraries handle how the engine orchestrates INSERT, UPDATE, and DELETE operations by managing table metadata (snapshots, manifests, schema, partitioning) and coordinating with the underlying data files (Parquet, ORC, Avro on S3, HDFS, etc.)
* **Data Files** → Contain actual rows in Parquet/ORC/Avro.
* **Delete Files** → Track rows that should be excluded (row-level deletes).
* **Manifests** → Point to groups of data files, along with stats (min/max values, counts).
* **Snapshot Metadata** → Records which manifests (and hence which files) make up a consistent table state.

Because queries always resolve to a **snapshot**, readers never see partial writes or inconsistent data.


## Example: Creating an Iceberg Table

Here’s a Spark SQL example:

```sql
-- Create an Iceberg table
CREATE TABLE my_catalog.sales (
    id BIGINT,
    amount DOUBLE,
    region STRING
) USING iceberg
PARTITIONED BY (region);

-- Insert data
INSERT INTO my_catalog.sales VALUES (1, 100.5, 'US'), (2, 250.0, 'EU');

-- Time travel: Query an older snapshot
SELECT * FROM my_catalog.sales.snapshot_id('123456789');
```

---

## The ACID Question: Are Iceberg Transactions Truly ACID?

Iceberg markets itself as **ACID-compliant**, but it’s important to unpack what that really means.

Unlike a relational database (Postgres, SQL Server, Oracle), Iceberg doesn’t manage **row-level locks or in-place updates**. Instead, it achieves **snapshot isolation at the file level**:

1. **Updates and Deletes**:

   * Iceberg never modifies Parquet/ORC files directly.
   * Updates create new files with modified rows; deletes create *delete files* that mark rows invalid.
   * Old files remain until garbage collected.

2. **Readers See a Consistent Snapshot**:

   * Queries always reference a snapshot (set of manifests and files), so they never see partial results.

3. **Compaction (a.k.a. Rewrite)**:

   * Updates/deletes leave behind lots of small or expired files.
   * Iceberg relies on **compaction jobs** to merge valid rows into optimized files and discard expired ones.

So, Iceberg gives you **transactional ACID-like guarantees** — but at the **table/file level**, not the row/page level like a database engine.

---

## Learning Resources

Here are some excellent links to go deeper:

* [Apache Iceberg Official Docs](https://iceberg.apache.org/docs/latest/)
* [Netflix Blog: Iceberg — A Table Format for Huge Data Sets](https://netflixtechblog.com/iceberg-a-table-format-for-huge-analytic-datasets-2e6bbc58158e)
* [Trino Iceberg Connector](https://trino.io/docs/current/connector/iceberg.html)
* [AWS Iceberg Integration (Athena & Glue)](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
* [Iceberg GitHub](https://github.com/apache/iceberg)

