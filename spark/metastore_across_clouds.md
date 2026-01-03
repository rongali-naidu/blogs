# Metastore: From Hive Metastore to Cloud-Native Catalogs

In every data lake architecture, one component silently determines whether data is usable or merely a collection of files: **the Metastore**.
The metastore is the *Source of Truth* for schemas, partitions, and table locations. Query engines like **Apache Spark**, **Presto/Trino**, and **Hive** rely on it to understand what data exists and how to read it. Without a metastore, a data lake has storage—but no structure.

This blog explains:

* What a metastore is
* Why **Hive Metastore (HMS)** became the standard
* How traditional metastore setups work
* How cloud platforms (AWS, Azure, GCP) evolved metastores into managed catalogs
* How operations like **CREATE, ALTER, and DROP TABLE** differ across systems

---

## What Is a Metastore?

A **Metastore** is a centralized repository that stores metadata *about* data, not the data itself.

It typically contains:

* Database and table names
* Column definitions and data types
* Partition columns and values
* Physical data locations (S3, ADLS, GCS, HDFS)
* Table formats (Parquet, ORC, Iceberg, Delta, Hudi)

When a query engine executes:

```sql
SELECT * FROM sales WHERE year = 2025;
```

It first consults the metastore to determine:

* Where the table lives
* What schema applies
* Which partitions to read

Only then does it access the actual data files.

> **The metastore gives meaning to raw data files.**

---

## Hive Metastore: The Open-Source Foundation

The **Hive Metastore (HMS)** is the most widely adopted metastore implementation in the Hadoop ecosystem and remains the compatibility layer that Spark understands today.

HMS has two core components:

1. **Metastore Database**

   * Relational database (MySQL, PostgreSQL, SQL Server)
   * Stores schemas, partitions, and properties

2. **Metastore Service**

   * JVM-based service
   * Exposes metadata via the **Thrift protocol**

Query engines interact with the HMS service to resolve metadata before accessing data.

---

## How Query Engines Interact with the Metastore

A critical concept: **data never flows through the metastore**.

### Traditional Hive Metastore Flow

```text
Query Engine (Spark / Presto)
        |
        |  Thrift
        v
Hive Metastore Service
        |
        |  JDBC
        v
Metastore Database (Schemas & Partitions)

Query Engine
        |
        |  Object Storage / HDFS
        v
Actual Data Files
```

---

## Traditional Hive Metastore Setup

In a traditional deployment, teams build and operate the entire metastore stack.

### Components You Manage

* Hive Metastore Service
* SQL database
* Apache Spark
* Hadoop-compatible storage
* JDBC drivers

### Spark Connectivity

```text
hive.metastore.uris = thrift://<metastore-host>:9083
```

Spark communicates with HMS over Thrift, and HMS persists metadata in the SQL database. Metadata availability is therefore tied to service uptime and database health.

---

## Cloud-Native Metastore Implementations

Cloud providers kept the **Hive Metastore contract**, but fundamentally changed *how it is delivered*.


### Cloud-Native Metastore Flow

```text
Query Engine (Spark / Trino)
        |
        |  API Calls + Identity
        v
Cloud Metastore (Glue / Unity Catalog)

Query Engine
        |
        |  Object Storage
        v
Actual Data Files
```

> **Metastores are consulted, not traversed.**

---

### AWS Glue Data Catalog

* Serverless metadata service
* IAM-based access
* No Hive Metastore service or SQL database to manage

```text
spark.hadoop.hive.metastore.client.factory.class =
com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory
```

For cross-account catalogs:

```text
hive.metastore.glue.catalogid = 123456789012
```

### Identity Model

* Access is controlled by **IAM Roles**
* No database credentials
* Metadata persists independently of Spark clusters


---

## HMS vs AWS Glue: Operational Summary

| Category                         | Operation               | Hive Thrift Metastore (HMS)            | AWS Glue Data Catalog                  |
| -------------------------------- | ----------------------- | -------------------------------------- | -------------------------------------- |
| **Architecture**                 | Backend                 | Manually managed SQL DB                | Fully managed metadata store           |
|                                  | Connectivity            | Thrift URI (`thrift://ip:9083`)        | AWS Account ID / API-based             |
|                                  | Security                | Firewalls + DB credentials             | IAM & Glue resource policies           |
|                                  | Persistence             | Metadata lost if DB/service is deleted | Metadata survives cluster deletion     |
| **Table Lifecycle / Partitions** | CREATE TABLE            | Thrift call → SQL INSERT               | `glue:CreateTable` API                 |
|                                  | ALTER / UPDATE TABLE    | Thrift call → SQL UPDATE               | `glue:UpdateTable` API                 |
|                                  | DROP TABLE              | Thrift call → SQL DELETE               | `glue:DeleteTable` API (metadata-only) |
|                                  | Add / Manage Partitions | Manual or `MSCK REPAIR TABLE`          | Crawlers or APIs                       |

---

## Azure: SQL Metastore vs Unity Catalog

Azure supports **two generations** of metastore architectures.

### Azure Synapse (Traditional Cloud Model)

* Backend: Azure SQL Database
* Spark connects using a **Linked Service**
* Similar to traditional HMS, but infrastructure is managed

**Key Config**:

```text
spark.hadoop.hive.synapse.externalmetastore.linkedservice.name
```

---

### Azure Databricks: Unity Catalog (Modern Model)

Unity Catalog is Azure’s **cloud-native, account-level metastore**.

* Centralized across workspaces
* Built-in governance and access control
* No Thrift URIs or JDBC wiring

Spark automatically discovers Unity Catalog using the **workspace identity**.

> Metadata becomes a **shared platform service**, not a cluster dependency.

---

## Google Cloud: Dataproc Metastore

Google Cloud provides a managed Hive Metastore service that remains close to open-source behavior.

### Key Characteristics

* Dataproc Metastore is fully managed and serverless
* You attach it to Spark clusters by service name
* GCP injects required configuration automatically

Spark still uses:

```text
hive.metastore.uris = thrift://<managed-host>:9083
```

But **lifecycle, scaling, and availability** are fully managed by the platform.

