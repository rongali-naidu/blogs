# Apache Iceberg: The Table Format That Makes Data Lakes Feel Like Databases

## Introduction

When you think about a database, the idea of `INSERT`, `UPDATE`, `DELETE`, and transactional guarantees feels natural. But when you move to a data lake built on object storage like **Amazon S3**, the foundation looks very different. At the lowest level, you only have **files** — typically columnar formats such as **Parquet** or **ORC**. These formats are excellent for analytics, but they have no built-in notion of:

* Which files together make up a table.
* How to evolve a schema safely.
* How to apply updates or deletes without rewriting huge datasets.
* How to guarantee consistency when multiple jobs run in parallel.

This gap is exactly what **Apache Iceberg** fills.

Iceberg is not a query engine. It is a **table format specification and library** that query engines like **Spark, Flink, Trino, Athena, and Snowflake** plug into. The Iceberg library orchestrates how engines manage **metadata, snapshots, and data files** to provide:

* **ACID transactions** on object storage.
* **Schema evolution** without rewriting historical data.
* **Hidden partitioning** for simpler queries.
* **Time travel** to query data as of a snapshot.
* **Cross-engine compatibility**.

In short, Iceberg brings **database-like behavior** to your data lake.

---

## Why Iceberg?

If Parquet already compresses and stores data efficiently, why add Iceberg on top?

The answer: **file formats manage data within a single file**; they don’t manage how multiple files act together as a table.

Without Iceberg:

* A table is just “a folder of files” in S3.
* There is no atomicity if you overwrite multiple files — a query may read a half-written state.
* Updates and deletes require rewriting entire partitions.
* Schema changes are painful.

With Iceberg:

* A table is defined by a **metadata layer** (snapshots + manifests).
* Queries always see a **consistent snapshot**.
* Updates and deletes are tracked via **delete files**, not immediate rewrites.
* Schema can evolve gracefully.
* Snapshots allow **time travel** and **rollback**.
* Supports **concurrent writes** with **optimistic concurrency**. ([Apache Iceberg][1])

This is what elevates Iceberg above “just Parquet.”

---

## Iceberg Internals: Table Layout on S3

When you create an Iceberg table on S3, the storage layout looks like this:

```
orders/
 ├── data/                      <-- actual Parquet files
 │    ├── part-0000.parquet
 │    └── ...
 └── metadata/                  <-- Iceberg-managed files
      ├── v1.metadata.json      <-- table metadata (current snapshot, schema, props)
      ├── manifest-list-1.avro  <-- list of manifests for snapshot 1
      ├── manifest-*.avro       <-- file-level metadata
      └── version-hint.text     <-- pointer to latest version
```

Key components:

* **Metadata JSON**: root pointer for the table; tracks schema, partitioning, current snapshot ID.
* **Manifest Lists**: reference the manifests that belong to each snapshot.
* **Manifests**: track data files and delete files, with stats like min/max values.
* **Data Files**: Parquet (or ORC/Avro) files holding actual rows.
* **Delete Files**: special files that track row-level deletes.

Every operation (`INSERT`, `UPDATE`, `DELETE`) creates a **new snapshot** with its own manifests.

---

## Prerequisites: Spark + Iceberg + S3 Catalog

To use Iceberg with Spark on S3, you need:

1. **Spark** with the Iceberg runtime JAR.
2. **Iceberg library** (the spec + implementation).
3. **Catalog** configured to track tables (Glue, Hive Metastore, or custom REST).
4. **S3 bucket** where Iceberg will store data and metadata files.

Example Spark configuration for AWS Glue Catalog:

```bash
spark-sql \
  --packages org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.5.0 \
  --conf spark.sql.catalog.glue=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.glue.warehouse=s3://my-datalake/warehouse \
  --conf spark.sql.catalog.glue.catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog
```

For more details on setting up Spark with Iceberg, refer to the [Spark Quickstart Guide](https://iceberg.apache.org/spark-quickstart/).

---

## Creating a Table: What Lands in S3

```sql
CREATE TABLE glue.orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DOUBLE,
    order_date TIMESTAMP
) USING iceberg
PARTITIONED BY (days(order_date));
```

In S3:

* A new folder `warehouse/orders/` is created.
* Under `metadata/`, Iceberg writes `v1.metadata.json`, a manifest list, and manifests.
* The `data/` folder is empty until rows are inserted.

---

## Write Flow: Appends and Inserts

When you `INSERT` rows:

1. Spark writes Parquet files under `data/`.
2. Iceberg generates new manifest entries pointing to those files.
3. A new **snapshot** is committed by writing `v2.metadata.json`.
4. The catalog pointer is atomically updated to reference the new metadata.

This ensures readers see either the old snapshot or the new one — never an inconsistent mix.

---

## Read Flow: How Queries Resolve Files

When Spark reads an Iceberg table:

1. It loads the **current metadata.json**.
2. It finds the **active snapshot**.
3. It loads manifests → lists candidate data files.
4. If there are delete files, Spark applies them at read time:

   * **Equality deletes**: filter out rows where column values match.
   * **Position deletes**: drop rows at specific `(file, row_position)` coordinates.

The result is as if the table had been physically updated.

---

## Updates & Deletes: Equality vs Position Deletes

Iceberg never edits Parquet files in place. Instead, it creates **delete files**.

* **Equality Deletes**: store a filter condition like `(order_id=123)`. Any row in data files with that value is excluded.
* **Position Deletes**: store `(file_path, row_position)` to surgically remove specific rows.

Position deletes are more precise but require the engine (e.g., Spark) to know row positions when writing.

For more information on delete file types, refer to the [Equality Delete Files Specification](https://iceberg.apache.org/spec/?h=equality).

---

## Concrete Example: DELETE in Spark SQL

```sql
DELETE FROM glue.orders WHERE order_id = 1001;
```

What happens:

1. Spark scans manifests to locate files that *might* contain `order_id=1001`.
2. It finds the row in `data-0003.parquet` at position 532.
3. Spark writes a **position delete file**:

```
file_path             row_position
----------------------------------
data-0003.parquet     532
```

4. Iceberg commits a new snapshot pointing to old data files + the new delete file.

At query time, Spark skips row 532 when reading `data-0003.parquet`.

Later, compaction can rewrite `data-0003.parquet` without row 532 and drop the delete file.

---

## Compaction / Rewrite

Over time, too many small data files and delete files degrade performance. Iceberg provides:

* **Data file compaction** (`rewrite_data_files`): merge small Parquet files, apply deletes, rewrite partitions.
* **Manifest rewrite**: reduce manifest count for faster planning.

Compaction is usually scheduled as a background job. For more details, refer to the [Rewrite Data Files Documentation](https://iceberg.apache.org/javadoc/1.4.1/org/apache/iceberg/actions/RewriteDataFiles.html).

---

## Snapshot Expiration / Garbage Collection

Each snapshot references data + delete files. To save storage:

* Use `expire_snapshots` to drop old snapshots.
* Run `remove_orphan_files` to clean S3 files not tracked in metadata.

This ensures your lake doesn’t grow endlessly. For more information, refer to the [Maintenance Documentation](https://iceberg.apache.org/docs/1.5.1/maintenance/).

---

## Lifecycle of an UPDATE in Practice

Let’s walk through an update step by step:

1. **Initial State**: Table points to snapshot 10 with 3 Parquet files.
2. **UPDATE issued**: Spark identifies candidate files.
3. **Row identified**: Spark determines `(file, row_position)` for updates.
4. **Delete written**: Spark writes a position delete file for the old row.
5. **Insert written**: Spark writes a new Parquet file with the updated row.
6. **New snapshot committed**: Snapshot 11 = old data files + new data file + delete file.
7. **Query behavior**: Readers see only the updated row, not the deleted one.
8. **Compaction**: Eventually rewrites files to absorb deletes.
9. **Snapshot expiration**: Old snapshots (like 10) can be dropped once retention policy allows.

This is the Iceberg way: **logical ACID at metadata + file level, with background maintenance to stay efficient**.

