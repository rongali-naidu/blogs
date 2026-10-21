
# What Does *Hive-Compatible* AWS Glue Table Really Mean?

## Context

I often come across the term **“Hive-compatible Glue table”**, but I could not find a clear, authoritative definition explaining *what exactly makes a Glue table Hive-compatible*.

Most explanations stop at *“Spark can read Glue tables”* — but **why** that works, and **where compatibility is enforced**, is rarely explained.

This blog walks through:

* How Spark queries metadata
* How Hive defines the metastore contract
* How AWS Glue plugs into that contract
* Where *Hive compatibility* is actually decided

---

## What Is a Metastore?

A **metastore** is a centralized repository that stores **metadata**, not data.

It typically stores:

* Database and table names
* Column definitions and data types
* Partition columns and values
* Physical locations (S3, ADLS, GCS, HDFS)
* Storage formats and SerDe information

### Example

```sql
SELECT * FROM datalake.sales WHERE year = 2025;
```

Before reading any files:

1. Spark queries the **metastore**
2. Resolves schema, partitions, and location
3. Only then reads data files


---

## How Hive Stores Metadata

### Creating a Hive Table

```sql
CREATE DATABASE IF NOT EXISTS datalake;

CREATE EXTERNAL TABLE datalake.sales (
    order_id STRING,
    amount DOUBLE,
    year INT
)
PARTITIONED BY (month INT)
STORED AS PARQUET
LOCATION 's3://company-data/sales/';
```

### Hive Metastore Storage Model

Hive stores metadata in relational tables inside its metastore database:

| Hive Table   | Purpose                                      |
| ------------ | -------------------------------------------- |
| `DBS`        | Database metadata                            |
| `TBLS`       | Table metadata                               |
| `COLUMNS_V2` | Column definitions                           |
| `PARTITIONS` | Partition values                             |
| `SDS`        | Storage descriptor (location, format, SerDe) |

If the Hive metastore database is lost, **tables become unusable**, even though the data still exists.

---

## How Spark Queries Hive Tables

### Spark Catalog Flow

```sql
SHOW TABLES IN datalake;
```

Spark executes this through the following layers:

```
Spark SQL
  → SessionCatalog
      → HiveExternalCatalog
```

### Spark Package (Ownership: Spark)

```
org.apache.spark.sql.hive.HiveExternalCatalog
```

Source:
[https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/HiveExternalCatalog.scala](https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/HiveExternalCatalog.scala)

Important:

* This class is **Spark code**
* It does **not** know about Glue
* It delegates metadata calls to a **Hive client**

---

## The Hive Client Used by Spark

### Spark Hive Client Interface (Ownership: Spark)

```
org.apache.spark.sql.hive.client.HiveClient
```

Source:
[https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/client/HiveClient.scala](https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/client/HiveClient.scala)

Spark programs against this interface, not against Hive or Glue directly.

---

## How Spark Chooses a Metastore Implementation

The decision is made **inside Spark’s Hive client implementation**:

```
org.apache.spark.sql.hive.client.HiveClientImpl
```

Source:
[https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/client/HiveClientImpl.scala](https://github.com/apache/spark/blob/master/sql/hive/src/main/scala/org/apache/spark/sql/hive/client/HiveClientImpl.scala)

Spark delegates metastore creation to **Hive**, using this Hive configuration:

```properties
hive.metastore.client.factory.class
```

---

## Hive Owns the Metastore Factory Contract

### Hive Factory Interface (Ownership: Hive)

```
org.apache.hadoop.hive.metastore.HiveMetaStoreClientFactory
```

Source:
[https://github.com/apache/hive/blob/master/standalone-metastore/metastore-common/src/main/java/org/apache/hadoop/hive/metastore/HiveMetaStoreClientFactory.java](https://github.com/apache/hive/blob/master/standalone-metastore/metastore-common/src/main/java/org/apache/hadoop/hive/metastore/HiveMetaStoreClientFactory.java)

This interface defines how a metastore client must be created:

```java
IMetaStoreClient createMetaStoreClient(HiveConf conf);
```

---

### Default Hive Implementation

By default, Hive uses:

```
org.apache.hadoop.hive.metastore.SessionHiveMetaStoreClientFactory
```

which creates:

```
org.apache.hadoop.hive.metastore.HiveMetaStoreClient
```

These talk to the traditional Thrift-based Hive Metastore backed by an RDBMS.

---

## How AWS Glue Integrates (Critical Section)

AWS Glue integrates **by implementing Hive’s factory interface**.

### Glue Factory (Ownership: AWS)

```
com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory
```

Source:
[https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-hive3-client/src/main/java/com/amazonaws/glue/catalog/metastore/AWSGlueDataCatalogHiveClientFactory.java](https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-hive3-client/src/main/java/com/amazonaws/glue/catalog/metastore/AWSGlueDataCatalogHiveClientFactory.java)

This class:

* Implements `HiveMetaStoreClientFactory`
* Is loaded by Spark **via Hive**
* Produces a Glue-backed Hive client

---

### Glue Hive Client

```
com.amazonaws.glue.catalog.metastore.GlueHiveMetaStoreClient
```

Source:
[https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-hive3-client/src/main/java/com/amazonaws/glue/catalog/metastore/GlueHiveMetaStoreClient.java](https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-hive3-client/src/main/java/com/amazonaws/glue/catalog/metastore/GlueHiveMetaStoreClient.java)

This client:

* Implements `IMetaStoreClient` (Hive interface)
* Calls AWS Glue APIs
* Converts Glue metadata internally
* Returns **Hive objects**

---

## Why Conversion to Hive `Table` Is Mandatory

Regardless of backend, Spark expects:

```
org.apache.hadoop.hive.metastore.api.Table
```

Source:
[https://github.com/apache/hive/blob/master/standalone-metastore/metastore-common/src/main/java/org/apache/hadoop/hive/metastore/api/Table.java](https://github.com/apache/hive/blob/master/standalone-metastore/metastore-common/src/main/java/org/apache/hadoop/hive/metastore/api/Table.java)

Glue converts its metadata using:

```
com.amazonaws.glue.catalog.converters.BaseCatalogToHiveConverter
```

Source:
[https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-client-common/src/main/java/com/amazonaws/glue/catalog/converters/BaseCatalogToHiveConverter.java](https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/master/aws-glue-datacatalog-client-common/src/main/java/com/amazonaws/glue/catalog/converters/BaseCatalogToHiveConverter.java)

This conversion is the **true Hive compatibility gate**.

---

## So What Does *Hive-Compatible Glue Table* Mean?

A **Glue table is Hive-compatible** if it can be successfully converted into a Hive Table object using:

```java
BaseCatalogToHiveConverter.convertTable(catalogTable, dbName)
```

**Requirements:**

1. Non-null `StorageDescriptor`
2. Non-empty `StorageDescriptor.Columns`

* Tables missing these fields exist in Glue but **cannot be queried via Spark/Hive**.
* This is the **operational definition of Hive compatibility**, not a formal AWS spec.

**References:**

* [Glue convertTable method](https://github.com/awslabs/aws-glue-data-catalog-client-for-apache-hive-metastore/blob/53d09f0c97edb913b02e00904b6620ea7468e8f5/aws-glue-datacatalog-client-common/src/main/java/com/amazonaws/glue/catalog/converters/BaseCatalogToHiveConverter.java#L58)
* [Hive Table API class](https://github.com/prongs/apache-hive/blob/master/metastore/src/gen/thrift/gen-javabean/org/apache/hadoop/hive/metastore/api/Table.java)
