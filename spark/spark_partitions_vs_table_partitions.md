## Don't Confuse Spark Execution Partitions with Table-Level Partitions
### 1. **Spark Partition (Execution-Time Partition)**
**This is what Spark tasks operate on**.
* A Spark **partition** is a **chunk of data assigned to one task**.
* Defined during **read time** and based on:
* File splits (e.g., multiple Parquet/CSV blocks),
 * Number of cores/executors,
 * `repartition()`, `coalesce()`, etc.
Example:
```python
df = spark.read.parquet("s3://my-bucket/data/")
print(df.rdd.getNumPartitions())
```
This might return 200 - meaning **200 Spark partitions**, one for each task.
These partitions are:
* **Used for parallelism**
* Temporary - defined **per job**
* **Not related to Glue table partitioning**
 - -
### 2. **Glue Catalog Partition (Table Partitioning in Metadata)**
These are **logical partitions defined at the table level** - like folders in S3.
Example:
A Glue table partitioned by `year`, `month`, `region` means S3 layout like:
```
s3://bucket/my_table/year=2024/month=06/region=us/
```
Glue **knows these as metadata partitions**, and Spark can **prune** them based on queries:
```python
df = spark.read.table("my_db.my_table").filter("year = 2024 AND month = 06")
```
Benefits:
* **Partition pruning** → only reads matching folders
* Can greatly reduce I/O
But they **don't define execution partitions** - even if 1 folder is read, Spark might create many Spark partitions depending on data size.
