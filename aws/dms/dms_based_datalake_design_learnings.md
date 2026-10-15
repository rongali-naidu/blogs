# Building a Lakehouse with AWS DMS : Design Choices & Learnings

**Context:** An application stores data in Amazon Aurora MySQL, and we want to bring that data into a data lakehouse.

## Overall Design Flow
    ┌─────────────────────────────┐
    │   Aurora MySQL (source app) │   classic binlog
    └──────────────┬──────────────┘
                   │  full-load + CDC (near real time)
                   ▼
    ┌─────────────────────────────┐
    │          AWS DMS            │   BatchApplyEnabled = false (S3 target)
    └──────────────┬──────────────┘   LOB handling set explicitly
                   │  Parquet, date-partitioned
                   ▼
    ┌─────────────────────────────┐
    │   S3  —  RAW / BRONZE       │   landing zone (not queried directly)
    └──────────────┬──────────────┘
                   │  Glue ETL — cron schedule + job bookmarks
                   │  (clean, transform, PK MERGE)
                   ▼
    ┌─────────────────────────────┐
    │  Iceberg on S3 — SILVER     │   ACID; merge-on-read; scheduled compaction
    └──────────────┬──────────────┘
                   │  aggregate
                   ▼
    ┌─────────────────────────────┐
    │  Iceberg on S3     — GOLD   │   materialized views / aggregated datasets
    └──────────────┬──────────────┘
                   ▼
         BI / Analytics / Consumers

Architecture patterns used:
- **ELT** — extract raw data and load it, then transform.
- **Medallion** — Raw (Bronze) → Curated (Silver) → Aggregated (Gold).
- **Lakehouse** — ACID operations via Apache Iceberg tables.
- **Batch + near-real-time** — DMS applies changes near-real-time (CDC); Glue ETL runs on a cron schedule with job bookmarks, so cadence can shift from near-real-time to daily as needs change.

Note on Iceberg MVs: Apache Iceberg does not provide materialized views natively — the  Iceberg spec only defines views (logical, no stored data); materialized views are still a proposal. The AWS Glue Data Catalog provides managed materialized-view support: it stores the pre-computed query result as an Iceberg table and handles change-detection and refresh (scheduled or incremental). It works only on the AWS-optimized Spark runtime (Glue/EMR/Athena, Spark ≥ 3.5.6) — not open-source Spark. More details: https://aws.amazon.com/blogs/big-data/introducing-apache-iceberg-materialized-views-in-aws-glue-data-catalog/

## Why AWS DMS over RDS "Zero-ETL" (as of 2026-09-08)

1. **AWS RDS Zero-ETL doesn't support an S3 / S3 Tables target yet** — it currently targets the SageMaker Lakehouse (SMLH) / Redshift-managed catalog. Our target is open-format Iceberg on S3. For More on AWS Zero-ETL refer https://aws.amazon.com/what-is/zero-etl/
2. **You can't run DMS and Zero-ETL on the same source at once.** Both rely on the binlog for CDC, but they need *different* binlog modes — DMS uses the classic binlog, while Aurora Zero-ETL requires enhanced binlog (`aurora_enhanced_binlog=1`). Switching the source to enhanced binlog can disrupt a running DMS task, so the two can't coexist on the same source.
3. **We left the door open** to adopt Zero-ETL once it supports Iceberg on S3 — at which point the binlog-mode switch would be sequenced after DMS is retired.
4. **Note**: Like Zero-ETL, DMS lands data into the Bronze layer with no custom ETL pipelines

## Key DMS Learnings

### 1. "Batch apply" is not supported for an S3 target — and can silently lose data
Production incident. With batch apply enabled against S3, the replicator used a primary-key bulk large-object apply path that **failed on a table with a large-object (LOB) column and no PRIMARY KEY** (only a unique key). It failed during **incremental/CDC apply**, not during the initial full load — so it passed full-load and broke later.

- **Fix:** disable batch apply for S3 targets (`BatchApplyEnabled=false`).
- **Per AWS:** "BatchApply is not supported for an S3 endpoint. Using Batch Apply … for an S3 target might result in loss of data."
  (https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.S3.html)
- **Watch for:** tables with LOB columns and no primary key — this is the exact shape that triggered the failure. Audit your source for it before enabling CDC.

### 2. Set large-object (LOB) handling EXPLICITLY — defaults silently truncate
The implicit default (especially with batch apply) is limited-LOB mode capped low (~tens of KB) — **larger values are silently truncated while the task still reports success.** A correctness landmine.

- **Fix:** set LOB handling explicitly — support LOBs, limited-size mode (not full-LOB, so you keep fast change-apply), and a max-LOB size set well above your largest realistic value so nothing truncates.
