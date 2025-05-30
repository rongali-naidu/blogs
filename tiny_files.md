
# **The Tiny Files Problem: A Silent Bottleneck in Real-Time Data Lakes**



Real-time data lakes promise instant insights, agile decision-making, and a responsive data platform. But lurking behind many real-time architectures is an underappreciated problem that gradually erodes performance and scalability: **the tiny files problem**.

As someone who has spent years working on data ingestion pipelines, streaming architectures, and lakehouse designs, I’ve seen this pattern repeat: teams nail the data ingestion story using tools like Kinesis Firehose, Spark Streaming, or Flink—only to discover that downstream queries slow down, jobs get unstable, and costs spike. Often, the root cause boils down to **an explosion of tiny files**.

Let’s dive into what this problem is, how it manifests, why it's especially painful in cloud-native architectures like AWS, and what practical strategies exist to deal with it.

---

## **1. What Is the Tiny Files Problem?**

A "tiny file" is a file that is too small to justify the overhead required to read, store, or process it efficiently. In cloud data lakes, especially those built on Amazon S3 or similar object stores, real-time or micro-batch ingestion often creates **thousands of small Parquet or JSON files per day**.

Here’s how it happens:

* Firehose buffers 5MB or 1-minute worth of data → one tiny Parquet file.
* Spark or Flink writes micro-batches every 10 seconds → each batch becomes a small file.
* Concurrent writers append data → more fragmentation.

**Result:** Your data lake ends up with **millions of small files**, scattered across partitions.

---

## **2. Why Are Tiny Files a Problem?**

While tiny files help ingest data with low latency, they create major bottlenecks later:

### **a. Query Performance Degradation**

* Engines like Athena, Redshift Spectrum, and Presto must open each file individually.
* Each file adds I/O overhead, metadata scanning, and CPU cost.
* Partition pruning and predicate pushdown are rendered ineffective when scanning thousands of files.

### **b. Increased Cost**

* You pay per query or per MB scanned. Tiny files make engines read metadata and footers repeatedly.
* Serverless tools like Athena incur **higher cost per query**, even when actual data scanned is low.

### **c. Metadata Explosion**

* Glue catalogs or Hive metastore struggle to track hundreds of thousands of file entries.
* Listing files (e.g., via S3 or Hadoop’s file systems) becomes painfully slow.

### **d. Job Failures and Operational Complexity**

* Spark or Flink jobs reading millions of files often fail or hit scalability limits.
* Recovery and retries are expensive and unpredictable.

---

## **3. Why It’s Worse in Real-Time Data Lakes**

Batch systems generate predictable and compact files. Real-time pipelines don’t.

Real-time ingestion by nature produces **frequent, small updates**:

* Every event stream triggers a new write.
* Each write targets a specific partition (e.g., by hour or minute).
* Writers don’t coordinate globally.

This leads to **file fragmentation**, especially on high-volume or high-frequency tables.

### Example:

You ingest events into S3 via Firehose every 60 seconds. With 50 partitions (e.g., by appId, region, and time), you get 50 files per minute → **72,000 files/day**. Within a week: **half a million tiny files.**

---

## **3.5. How Tiny Files Affect Apache Spark Workloads**

Apache Spark is one of the most powerful tools for processing big data—but its design makes it particularly vulnerable to the tiny files problem.

### **Spark's Execution Model:**

* Spark divides input into **splits**, where each split maps to a file or file segment.
* Each split becomes a task. Thousands of tiny files → thousands of tasks.

### **Impact on Spark:**

* **Excessive task scheduling** → Spark driver slows down or crashes.
* **High GC pressure** on executors from metadata and shuffle management.
* **Longer job times** due to stage delays and underutilized resources.
* **Skewed shuffles** when small files are unevenly distributed across partitions.

### **What You Can Do in Spark:**

* Use `coalesce()` or `repartition()` to reduce output files.
* Trigger **compaction jobs** periodically using Spark or Glue.
* Combine **micro-batch writes** with compaction-aware sinks.
* Where possible, use **merge-on-read** table formats (Iceberg, Hudi) to separate ingest from query optimization.

---

## **4. AWS-Specific Considerations**

In AWS ecosystems, the tiny files problem is exacerbated by:

* **Firehose** default buffering (5MB or 1-minute) → too small for analytic workloads.
* **Athena**'s pricing model: \$ per TB scanned, with no optimization for metadata-heavy reads.
* **Glue Catalog** tracking S3 files → bloated metadata, slower crawls, higher job start times.
* **Redshift Spectrum** accessing S3 tables → similar penalties for tiny files.

---

## **5. How Table Formats Like Iceberg Help**

Apache Iceberg (and cousins like Delta Lake and Hudi) tackle tiny files natively:

* They **track metadata at the table level**, not the filesystem level.
* Support **automatic compaction** and **file rewriting**.
* Store **manifests** to optimize reads and eliminate file scanning.

In Iceberg, you can trigger:

```bash
CALL iceberg.system.rewrite_data_files('my_table');
```

This consolidates small files while keeping table snapshots intact. The system handles file listing, atomic updates, and metadata sync.

---

## **6. Alternatives and Mitigations**

Not ready to introduce a full table format like Iceberg? You still have options.

### **1. Use Table Formats with Built-In Compaction**

* **Iceberg**: snapshot-aware compaction.
* **Delta Lake**: `OPTIMIZE` command.
* **Hudi**: background compaction in MOR mode.

These are great for managed lakehouse-style tables.

---

### **2. Separate “Live” Table + Historical Compaction Pattern**

This is my go-to for many AWS-based projects:

#### **Pattern:**

* Ingest real-time data into a “live” S3 location:

  ```
  s3://my-bucket/events/live/  ← contains tiny files
  ```
* Periodically (e.g., every hour), run a Spark/Glue job to:

  * Read and compact into larger files (100MB+).
  * Write to a “historical” location:

    ```
    s3://my-bucket/events/historical/
    ```
  * Clean up the live folder.

#### **Querying:**

* Use a view to union both layers:

  ```sql
  SELECT * FROM events_historical
  UNION ALL
  SELECT * FROM events_live
  ```

**Benefits:**

* Keeps ingestion fast.
* Keeps queries fast.
* Works with raw Parquet or ORC—no need for Iceberg.

---

### **3. Pre-Aggregation or Windowing on Ingestion**

Use Kinesis Data Analytics, Spark Structured Streaming, or Flink to:

* Buffer records in short windows (e.g., 5 minutes).
* Aggregate or roll up data before writing.
* Reduce write frequency and improve file sizes.

---

### **4. Stream to Staging, Then Compact**

Instead of writing directly to final destinations:

* Stream raw events to `s3://bucket/staging/`
* Periodically compact to `s3://bucket/curated/`
* Make curated layer queryable, skip staging for consumers.

---

## **Conclusion**

The tiny files problem is silent, but deadly. It starts small—literally—but grows into a serious performance and cost bottleneck. If you care about the long-term sustainability of your data lake or lakehouse, **you must invest in compaction strategies**.

Whether you embrace table formats like Iceberg or implement layered compaction manually, the goal is the same: **optimize for reads, not just writes**.

Because a fast ingest pipeline is only half the story. The real success comes when your analysts, ML models, and BI tools can get answers **fast and reliably**.
