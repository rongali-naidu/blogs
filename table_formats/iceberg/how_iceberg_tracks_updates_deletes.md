# 1) What is Iceberg

Iceberg is a *metadata/transaction layer* over Parquet/ORC/Avro on S3. Every write (append/update/delete/compact) produces a new **snapshot** (root metadata → manifest list → manifests → data/delete files). Reads consult that metadata to pick files and apply deletes so queries see a consistent snapshot. ([Apache Iceberg][1])

---

# 2) prerequisites: Spark + Iceberg + S3 (catalog)

Common pattern: configure a Spark catalog that points at an S3 warehouse (HadoopCatalog or Glue/Hive catalog). Example Spark conf (set before you run SQL):

```sql
-- in spark-submit or spark-shell before CREATE TABLE
spark.conf.set("spark.sql.catalog.my_catalog","org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.my_catalog.type","hadoop")               -- or 'hive'/'glue'
spark.conf.set("spark.sql.catalog.my_catalog.warehouse","s3://my-bucket/iceberg/")
-- also set aws credentials / s3 endpoint as usual for your environment
```

Now you can `CREATE TABLE my_catalog.db.my_table USING iceberg ...`. ([Apache Iceberg][2])

---

# 3) create table → what lands in S3 (file layout)

When you create an Iceberg table the engine writes a **root metadata file** under `s3://my-bucket/iceberg/db/my_table/metadata/` (files like `v1.metadata.json` or `00000-...metadata.json`) and you’ll start seeing a `data/` folder for Parquet files you write. Typical layout:

```
s3://my-bucket/iceberg/db/my_table/
  ├─ data/                     <-- Parquet/ORC/Avro data files
  │   ├─ data-00001-....parquet
  │   └─ ...
  └─ metadata/
      ├─ v1.metadata.json      <-- root metadata (current snapshot id, schema, manifest-list pointer)
      ├─ manifest-list-*.avro  <-- lists manifests for the snapshot
      ├─ manifest-*.avro       <-- lists of data-files (and stats) or delete-files
      └─ version-hint.text
```

* `vN.metadata.json` (root metadata) is the single source of truth for the table’s current snapshot and schema.
* A **manifest-list** (Avro) enumerates the manifests used by that snapshot.
* A **manifest** (Avro) contains one row per *data file* (or per delete file manifest) with partition values and per-column min/max counts used for pruning. ([Dremio][3])

---

# 4) append/write flow (what Spark does)

If you `INSERT INTO my_catalog.db.t VALUES (...)` or write a DataFrame:

1. Spark writes new Parquet files under `data/` (physical data files).
2. Iceberg writes a manifest file that lists those new data files + file metrics (row count, min/max).
3. Iceberg writes a new manifest-list + new `vN.metadata.json` that points to the manifest-list → this commit is atomic (snapshot).
   So a write creates: new data file(s) + manifest(s) + new root metadata (snapshot). Readers will see the new snapshot once commit completes. ([Apache Iceberg][1])

---

# 5) read flow (how a Spark query finds files and applies deletes)

When Spark runs `SELECT`:

1. **Open metadata**: Spark’s Iceberg reader opens the latest `vN.metadata.json` to get the current snapshot and manifest-list.
2. **Manifest pruning**: Using manifest statistics (partition min/max), it prunes manifests and then prunes data files inside manifests — avoids scanning irrelevant files.
3. **Collect delete manifests**: If there are delete manifests (equality/position), the reader learns which delete files apply. (Manifests may be of type “data manifest” or “delete manifest”.) ([Apache Iceberg][1])
4. **Read data files and apply deletes**: As the data file rows stream, the engine applies equality filters or skip rows indicated by position deletes so the final output reflects the snapshot.

This is why queries always return a consistent snapshot (atomic commit semantics at the metadata level). ([Apache Iceberg][1])

---

# 6) updates & deletes — how Iceberg tracks them (equality vs position)

Iceberg supports two delete encodings: **equality deletes** and **position deletes**. The spec and Spark implementations use these two approaches depending on the operation and engine behavior. ([Apache Iceberg][4])

* **Equality delete**: delete file contains column value(s) (e.g., `id = 123`). At read time the engine filters those values out. Small to medium-scale deletes or predicates keyed by business columns often use equality deletes.

  Delete file example (conceptual):

  ```
  -- equality-delete.parquet
  id
  ---
  123
  ```

* **Position delete**: delete file contains `(data_file_path, row_position)` pairs. The engine assigns positions while reading a data file (row 0, row 1, ...). A position delete says “skip row N in data-0001.parquet.” This is very surgical and avoids rewriting large files immediately, but it requires the engine to compute positions when creating the delete file and to check positions at read time. Example:

  ```
  -- position-delete.parquet
  data_file                pos
  --------------------------------
  s3://.../data-0001.parquet  1
  ```

  (Row positions are the physical order as read from the file.) ([Apache Iceberg][4])

**How Spark chooses**: Spark’s Iceberg integration will scan candidate data files (using manifest pruning) to find matching rows. For `UPDATE`/`DELETE` operations Spark commonly emits **position deletes** (it can identify exact row offsets during the scan) and then may also write new data files with updated rows (CoW). For MERGE/UPSERT/CDC workflows you might see equality deletes depending on how keys are supplied. ([Apache Iceberg][2])

---

# 7) concrete example — DELETE using Spark SQL (what gets written)

```sql
-- delete a single id
DELETE FROM my_catalog.db.people WHERE id = 2;
```

Under the hood (simplified):

1. Iceberg consults manifests to prune files that could contain `id=2`.
2. Spark scans those candidate Parquet files and determines the row *position(s)* where `id=2` lives.
3. Spark writes a **position delete file** with entries `(file_path, pos)` for each matching row.
4. Spark writes a new root metadata file (new snapshot) that references the new delete file (the delete file appears in manifests or a delete-manifest).
5. The original data file remains on S3 until compaction/expire/GC. The delete file is small (just keys) and marks rows logically deleted. ([Apache Iceberg][4])

At subsequent reads the reader will skip the row(s) from `data-0001.parquet` because the delete file says to skip position N.

---

# 8) compaction / rewrite — why and when to run it

Position deletes are great short-term (cheap metadata commit) but over time you get:

* many small data files (tiny-file problem), and
* many delete files that readers must check (read overhead).

**Compaction** (rewrite) merges data files and applies delete files physically so the rewritten data contains the “live” rows only — then delete files can be removed. Compaction tasks:

* Bin-pack small files into larger ones (target file size — often recommended 50–100MB+).
* Merge delete files with data files (apply deletes, drop deleted rows).
* Optionally recluster/sort (z-order, sort) to improve read locality. ([AWS Documentation][5])

**How to run in Spark** (procedures):

```sql
-- simple compaction (bin-pack)
CALL my_catalog.system.rewrite_data_files('db.my_table');

-- compaction with options
CALL my_catalog.system.rewrite_data_files(
  table => 'db.my_table',
  options => map('min-input-files','2', 'remove-dangling-deletes','true')
);
```

Use `rewrite_manifests` to rewrite manifests for scan efficiency, and `expire_snapshots` / `remove_orphan_files` to garbage-collect old snapshots and unreferenced files. These are built-in Spark procedures in Iceberg. ([Apache Iceberg][6])

**Heuristics / triggers**

* If average data file size << target (e.g., <<100MB) → run compaction (bin-pack).
* If query latency grows due to many delete files or manifest bloat → rewrite manifests & compact.
* Run compaction on *historical* partitions (e.g., older than N hours) to avoid conflicting with high-velocity streaming writes. AWS guidance recommends >100MB target and careful partition scoping for compaction. ([AWS Documentation][5])

---

# 9) snapshot GC / expiring snapshots

Iceberg keeps old snapshots for time travel and safety. These old snapshots keep references to old data files, so physical deletion is safe only after you expire snapshots.

Spark procedures to clean up:

```sql
-- expire snapshots older than timestamp (or keep last N)
CALL my_catalog.system.expire_snapshots(
  table => 'db.my_table',
  older_than => TIMESTAMP '2025-09-01 00:00:00',
  retain_last => 3,
  clean_expired_metadata => true
);

-- remove unreferenced files (orphan files)
CALL my_catalog.system.remove_orphan_files(table => 'db.my_table', older_than => TIMESTAMP '2025-09-01 00:00:00');
```

Important: expiring snapshots + removing orphan files is *not* automatic — you must schedule these maintenance tasks; otherwise S3 storage will still contain old data and metadata. ([Apache Iceberg][7])

---

# 10) lifecycle of an UPDATE in practice (summary)

1. `UPDATE` finds affected data files via manifest pruning.
2. Spark scans those files, writes new data files (with updated rows) and writes position-delete files that reference old file positions (or writes equality deletes for some patterns).
3. Iceberg creates a new snapshot (new metadata root + manifest-list).
4. Readers see the new snapshot atomically. Old files remain until compaction/expire.
5. Compaction/rewrite later physically removes deleted rows and drops delete files. ([Apache Iceberg][4])

---

# 11) practical tips / best practices

* **Set `write.target-file-size-bytes`** (table property) so writers produce files near your desired size; this reduces the need for early compaction. ([Tabular][8])
* **Compact historical partitions** (e.g., `where => 'date < current_date() - interval 1 day'`) — avoids conflicts with hot ingestion. ([Repost][9])
* **Schedule** `expire_snapshots` and `remove_orphan_files` regularly to reclaim S3 space. ([Apache Iceberg][7])
* **Monitor** manifest counts and delete-file counts (Iceberg metadata tables like `table.snapshots`, `table.manifests`, `table.files` are useful). ([dataos.info][10])

---

# 12) quick list of authoritative reading (handy links)

* Iceberg spec (manifests, snapshots, delete types). ([Apache Iceberg][1])
* Spark + Iceberg quickstart (examples & config). ([Apache Iceberg][2])
* Spark procedures: `rewrite_data_files`, `expire_snapshots`, `remove_orphan_files`. ([Apache Iceberg][6])
* AWS prescriptive guidance on compaction & best practices. ([AWS Documentation][5])


[1]: https://iceberg.apache.org/spec/?utm_source=chatgpt.com "Spec - Apache Iceberg™"
[2]: https://iceberg.apache.org/spark-quickstart/?utm_source=chatgpt.com "Spark and Iceberg Quickstart"
[3]: https://www.dremio.com/blog/a-hands-on-look-at-the-structure-of-an-apache-iceberg-table/?utm_source=chatgpt.com "A Hands-On Look at the Structure of an Apache Iceberg Table"
[4]: https://iceberg.apache.org/spec/?h=equality&utm_source=chatgpt.com "Equality Delete Files - Spec - Apache Iceberg™"
[5]: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-compaction.html?utm_source=chatgpt.com "Maintaining tables by using compaction - AWS Prescriptive Guidance"
[6]: https://iceberg.apache.org/docs/latest/spark-procedures/?utm_source=chatgpt.com "Procedures - Apache Iceberg™"
[7]: https://iceberg.apache.org/docs/nightly/spark-procedures/?h=remove&utm_source=chatgpt.com "Spark Procedures - Apache Iceberg"
[8]: https://www.tabular.io/blog/table-maintenance-the-key-to-keeping-your-iceberg-tables-healthy-and-performant/?utm_source=chatgpt.com "The Key To Keeping Your Iceberg Tables Healthy and Performant"
[9]: https://repost.aws/knowledge-center/glue-optimize-iceberg-tables-data-storage-query?utm_source=chatgpt.com "Optimize Iceberg tables for efficient data storage and queries"
[10]: https://dataos.info/resources/lakehouse/iceberg_metadata_tables/?utm_source=chatgpt.com "Iceberg Metadata Tables - All things DataOS"
