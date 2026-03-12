# AWS Database Migration Service (DMS): Architecture, Data Flow, and Building a Production Pipeline with CDK

## Introduction

Modern data platforms rely heavily on **reliable data movement and replication**. Whether you are migrating databases to the cloud, building analytics pipelines, or enabling near-real-time data lakes, **AWS Database Migration Service (DMS)** plays a critical role.

AWS Database Migration Service (DMS) is a managed service that helps migrate databases to AWS with **minimal downtime** while also supporting **continuous data replication (CDC – Change Data Capture)**.

One common use case is **streaming database changes into an S3 data lake** for analytics pipelines. However, many engineers struggle to understand **how DMS processes data internally and why there is sometimes latency before data appears in S3**.

In this guide, we will:

* Understand **DMS architecture**
* Learn how **data flows through the replication pipeline**
* Build a **production-ready DMS pipeline using AWS CDK**
* Demystify **timing, batching, and delays**
* Learn how to **optimize the pipeline for your workload**

---

# Understanding DMS Architecture

At a high level, AWS DMS consists of three primary components:

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│  Source         │      │  Replication     │      │  Target         │
│  Endpoint       │─────▶│  Instance        │─────▶│  Endpoint       │
│  (PostgreSQL)   │      │  (EC2-based)     │      │  (S3)           │
└─────────────────┘      └──────────────────┘      └─────────────────┘
```

### Key Components

### 1. Source Endpoint

This is the **database DMS reads from**.

Example sources:

* PostgreSQL
* MySQL
* Oracle Database
* Microsoft SQL Server

DMS captures changes from **database logs** (like WAL in PostgreSQL).

---

### 2. Replication Instance

The **replication instance** is the compute engine that runs the migration process.

It is essentially an **AWS-managed EC2 instance** responsible for:

* Reading source database logs
* Processing changes
* Transforming data
* Writing data to the target system

Instance sizing matters because it directly impacts:

* throughput
* latency
* memory buffering

---

### 3. Target Endpoint

The **target endpoint** is where DMS writes the data.

A common modern data architecture writes to:

* Amazon S3

When writing to S3, DMS can generate files in formats like:

* CSV
* JSON
* Parquet

Parquet is commonly used because it works well with analytics tools like:

* Amazon Athena
* Amazon Redshift
* AWS Glue

---

### 4. Replication Task

A **replication task** defines:

* which tables to migrate
* migration mode
* transformation rules
* performance tuning settings

Three migration modes exist:

1. **Full Load** – copies entire tables
2. **CDC (Change Data Capture)** – replicates ongoing changes
3. **Full Load + CDC** – initial load followed by continuous replication

---

# The Data Flow Pipeline

Understanding **how data flows through DMS** is crucial for diagnosing delays and optimizing latency.

Below is the internal pipeline.

```
┌────────────────────────────────────────────────────────────────────┐
│                    DMS DATA FLOW PIPELINE                          │
└────────────────────────────────────────────────────────────────────┘

1. Database Change Occurs
         ↓
2. [SOURCE_CAPTURE] DMS captures change from WAL/Logs
         ↓ (1-5 seconds)
3. [CHANGE PROCESSING] Batch changes in memory
         ↓ (BatchApplyTimeoutMax: 1-30 seconds)
4. [TRANSFORMATION] Convert to target format (Parquet)
         ↓ (2-5 seconds for compression + format conversion)
5. [TARGET_APPLY] Wait for S3 CDC interval
         ↓ (cdcMaxBatchInterval: up to 60 seconds by default)
6. [S3 WRITE] Write file to S3
         ↓ (1-3 seconds)
7. Data Available in S3
```

This pipeline reveals an important insight:

**DMS is not a streaming system. It is a batch-based CDC pipeline.**

This means changes are grouped before being written to the target.

---

# Two Critical Timing Stages

Understanding **where delays happen** is key to optimizing DMS.

## Stage 1 — Change Processing (In-Memory Batching)

After capturing database changes, DMS batches them **in memory**.

This stage is controlled by **ChangeProcessingTuning settings**.

Important parameters include:

```
BatchApplyTimeoutMin
BatchApplyTimeoutMax
BatchApplyMemoryLimit
BatchSplitSize
```

These settings control:

* how long DMS waits before flushing batches
* how large the in-memory batches can grow
* how frequently batches are applied

Originally, these were designed for **database-to-database replication**, not S3 file generation.

---

## Stage 2 — S3 Write Timing (File Creation)

Even after batches are processed, DMS still waits before writing to S3.

This behavior is controlled by **S3 CDC settings**.

Key parameters include:

```
cdcMinFileSize
cdcMaxBatchInterval
```

These determine:

* how large a CDC file should be
* how long DMS waits before writing it

For example:

```
cdcMaxBatchInterval = 60 seconds
```

This means DMS may **wait up to 60 seconds before writing a file**, even if data is ready.

This is often the **main cause of latency in S3 pipelines**.

---

# Building the Infrastructure with AWS CDK

Infrastructure as Code makes DMS deployments **repeatable, automated, and production-ready**.

We can define the entire pipeline using:

AWS Cloud Development Kit (CDK)

The architecture we will deploy:

```
PostgreSQL  →  DMS Replication Instance  →  S3 Data Lake
```

## Step 1 — Create the S3 Bucket

```ts
const bucket = new s3.Bucket(this, "DmsTargetBucket", {
  bucketName: "dms-cdc-data-lake",
  removalPolicy: RemovalPolicy.RETAIN
});
```

---

## Step 2 — Create the Replication Instance

```ts
const replicationInstance = new dms.CfnReplicationInstance(
  this,
  "DmsReplicationInstance",
  {
    replicationInstanceClass: "dms.t3.medium",
    allocatedStorage: 100,
    publiclyAccessible: false
  }
);
```

---

## Step 3 — Define Source Endpoint

```ts
const sourceEndpoint = new dms.CfnEndpoint(this, "PostgresSource", {
  endpointType: "source",
  engineName: "postgres",
  serverName: "db-host",
  port: 5432,
  username: "dms_user",
  password: "password",
  databaseName: "appdb"
});
```

---

## Step 4 — Define S3 Target Endpoint

```ts
const targetEndpoint = new dms.CfnEndpoint(this, "S3Target", {
  endpointType: "target",
  engineName: "s3",
  s3Settings: {
    bucketName: bucket.bucketName,
    dataFormat: "parquet",
    compressionType: "gzip"
  }
});
```

---

## Step 5 — Create Replication Task

```ts
const replicationTask = new dms.CfnReplicationTask(this, "DmsTask", {
  migrationType: "cdc",
  sourceEndpointArn: sourceEndpoint.ref,
  targetEndpointArn: targetEndpoint.ref,
  replicationInstanceArn: replicationInstance.ref,
  tableMappings: JSON.stringify({
    rules: [
      {
        "rule-type": "selection",
        "rule-id": "1",
        "rule-name": "1",
        "object-locator": {
          "schema-name": "%",
          "table-name": "%"
        },
        "rule-action": "include"
      }
    ]
  })
});
```

This creates a **fully automated CDC pipeline**.

---

# Understanding Timing and Delays

Many engineers expect **near real-time replication**, but several factors affect latency:

### Typical latency breakdown

| Stage              | Time     |
| ------------------ | -------- |
| Capture from WAL   | 1–5 sec  |
| Change batching    | 1–30 sec |
| Parquet conversion | 2–5 sec  |
| S3 CDC interval    | 0–60 sec |
| S3 write           | 1–3 sec  |

Total typical latency:

**5 seconds → 90 seconds**

depending on configuration.

---

# Optimizing for Your Use Case

The best configuration depends on your goals.

## Low Latency Analytics

Reduce batching delay.

Example:

```
cdcMaxBatchInterval = 5
BatchApplyTimeoutMax = 5
cdcMinFileSize = 1MB
```

Pros:

* faster updates in S3

Cons:

* more small files

---

## Cost-Optimized Data Lake

Increase batching.

Example:

```
cdcMaxBatchInterval = 300
cdcMinFileSize = 64MB
```

Pros:

* fewer files
* better query performance

Cons:

* higher latency

---

## High-Throughput Workloads

Use:

* larger replication instances
* larger batch sizes
* parallel apply settings

---

# Final Thoughts

AWS Database Migration Service is far more than just a migration tool. It is a **powerful CDC engine capable of feeding modern data lakes and analytics platforms**.

However, its **batch-oriented architecture means latency is influenced by multiple timing gates**, including:

* change processing buffers
* transformation overhead
* S3 batch intervals

By understanding these internal stages—and deploying infrastructure using **AWS Cloud Development Kit**—you can build a **robust, production-grade replication pipeline** that balances:

* latency
* throughput
* cost
* data lake performance

