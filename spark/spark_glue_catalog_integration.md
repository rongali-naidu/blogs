# From Hive to Glue: How Spark Sees Your Tables

In a modern data lake, metadata is as critical as the data itself. Spark, Hive, and cloud-native catalogs like **AWS Glue** rely on **metastores** to understand table schemas, partitions, and storage locations.

This blog explains:

* How Hive stores tables in the metastore
* How Spark queries Hive tables
* How Glue stores metadata
* How Spark sees Hive-compatible Glue tables via `AWSGlueDataCatalogHiveClientFactory` and `BaseCatalogToHiveConverter.convertTable()`

---

## 1️⃣ What Is a Metastore?

A **Metastore** is a centralized repository storing metadata *about data*, not the data itself.

It typically contains:

* Database and table names
* Column definitions and types
* Partition columns and values
* Physical data locations (S3, ADLS, GCS, HDFS)
* Table formats (Parquet, ORC, Delta, Iceberg, Hudi)

Example query:

```sql
SELECT * FROM datalake.sales WHERE year = 2025;
```

* Spark first consults the metastore to determine table location, schema, and partitions.
* Only then does it access the actual data files.

> The metastore gives structure and meaning to raw data files.

---

## 2️⃣ How Hive Stores Tables

### Step 1: Create Database and Table

Explicitly defining databases ensures proper metadata mapping:

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

### Step 2: Hive Metastore Storage

Hive stores metadata in **relational tables** in the metastore database:

| Hive Metastore Table | Purpose                                                 |
| -------------------- | ------------------------------------------------------- |
| `DBS`                | Database info (`datalake`)                              |
| `TBLS`               | Table info (`sales`)                                    |
| `COLUMNS_V2`         | Column names and types                                  |
| `PARTITIONS`         | Partition info (`month=1, month=2…`)                    |
| `SDS`                | Storage descriptors (location, file format, SerDe info) |

* If the Hive metastore database is deleted, tables become unusable.

---

## 3️⃣ How Spark Queries Hive Tables

Example:

```python
spark.catalog.listTables("datalake")
# or SQL:
SHOW TABLES IN datalake;
```

### Flow

1. Spark `SessionCatalog` → `HiveExternalCatalog`.
2. `HiveExternalCatalog` uses `HiveClient` → connects via **Thrift**.
3. Hive Metastore queries relational tables (`DBS`, `TBLS`, `SDS`) and returns **Hive Table objects**:

```java
org.apache.hadoop.hive.metastore.api.Table
dbName = "datalake"
tableName = "sales"
tableType = EXTERNAL_TABLE
storageDescriptor = { location: s3://company-data/sales/, columns: [...], format: Parquet }
partitionKeys = ["month"]
```

* **GitHub reference:** [Hive Table class (`Table.java`)](https://github.com/prongs/apache-hive/blob/master/metastore/src/gen/thrift/gen-javabean/org/apache/hadoop/hive/metastore/api/Table.java)

4. Spark converts these into `CatalogTable` objects for query execution.

---

## 4️⃣ How AWS Glue Stores Metadata

AWS Glue stores table metadata as **JSON-like objects**, not relational tables:

```json
{
  "Name": "sales",
  "DatabaseName": "datalake",
  "TableType": "EXTERNAL_TABLE",
  "StorageDescriptor": {
      "Columns": [
          {"Name": "order_id", "Type": "string"},
          {"Name": "amount", "Type": "double"},
          {"Name": "year", "Type": "int"}
      ],
      "Location": "s3://company-data/sales/",
      "InputFormat": "...",
      "OutputFormat": "...",
      "SerdeInfo": {"SerializationLibrary": "..."}
  },
  "PartitionKeys": [{"Name": "month", "Type": "int"}],
  "Parameters": {...}
}
```



## 5️⃣ How Spark Sees Glue Tables

### Step 1: Spark Configuration

```text
spark.hadoop.hive.metastore.client.factory.class =
com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory
```

* Replaces traditional ThriftHiveClient with Glue client.

### Step 2: Spark Calls Hive API

```python
SHOW TABLES IN datalake;
```

* Spark → `HiveExternalCatalog.getAllTables("datalake")` → Glue client → Glue API (`GetTables`)

### Step 3: Glue JSON → Hive Table

* `BaseCatalogToHiveConverter.convertTable(catalogTable, dbName)` maps Glue JSON → Hive Table.
* Only tables with **valid StorageDescriptor + columns** are Hive-compatible.

### Step 4: Hive Table → Spark CatalogTable

* Hive Table → Spark `CatalogTable`
* Queries (`SHOW TABLES`, `DESCRIBE TABLE`) now work as if using a traditional Hive Metastore.

---

## Hive Compatibility

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

---
