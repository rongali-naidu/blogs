# Bridging Python CSV Skills to Spark and Athena Internals: InputFormat, SerDe & OutputFormat

## Introduction

If you’re a Python developer, you’ve probably worked with CSV files using the built-in `csv` module or `pandas`. It feels simple: open the file, read rows, parse fields, and get to work. But when you step into the world of **Big Data** tools like **Apache Spark** or **AWS Athena**, you suddenly see terms like **InputFormat**, **OutputFormat**, and **SerDe**. These can feel intimidating at first.

This blog aims to bridge that gap. We’ll start with familiar CSV handling in Python, then map those concepts to how Spark and Athena process data under the hood. By the end, you’ll understand why these extra components exist and how they power large-scale distributed systems.

---

## 1. Reading CSV in Python

Let’s start with a simple CSV file stored in S3:

**Example File (customers.csv)**

```csv
customer_id,name,age
1,John Doe,30
2,Jane Smith,25
```

Using Python’s built-in `csv` module:

```python
import csv
import boto3

# Download file from S3
s3 = boto3.client("s3")
s3.download_file("my-bucket", "customers.csv", "customers.csv")

# Read CSV locally
with open("customers.csv", newline="") as csvfile:
    reader = csv.DictReader(csvfile)
    for row in reader:
        print(row)
```

Output:

```python
{'customer_id': '1', 'name': 'John Doe', 'age': '30'}
{'customer_id': '2', 'name': 'Jane Smith', 'age': '25'}
```

Here’s what happened:

* Python opened the file (I/O)
* The `csv` module parsed rows and split columns (parsing/serialization)
* You got a Python dictionary (usable object)

This is straightforward in Python—but what about Athena?

---

## 2. Defining the Same Data in Athena

Now let’s define this CSV file in Athena as an external table:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS customers (
  customer_id STRING,
  name STRING,
  age INT
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = ',',
  'quoteChar' = '"'
)
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-bucket/';
```

At first glance, this seems more complicated. But here’s how it maps back to Python’s `csv` handling:

* **InputFormat** → Like Python’s `open()` — tells Athena how to read raw files (e.g., text, parquet, ORC).
* **SerDe (Serializer/Deserializer)** → Like Python’s `csv.DictReader` — tells Athena how to parse raw text into structured rows/columns.
* **OutputFormat** → Like Python’s `writer.writerow()` — defines how data would be written back to storage (more relevant in Hive/Spark writes).

---

## 3. Why Do We Need InputFormat, OutputFormat, and SerDe?

The reason Spark, Hive, and Athena need these abstractions is **scale and flexibility**:

* Python assumes you have the whole file locally and small enough to fit in memory.
* Athena and Spark may be reading **billions of rows across thousands of files in S3**.
* Each abstraction plays a role:

  * **InputFormat** handles distributed file reading across nodes.
  * **SerDe** handles different file encodings (CSV, JSON, Parquet, Avro, etc.).
  * **OutputFormat** ensures that if results are written, they conform to a standard format.

---

## 4. Walking Through with a Sample Record

Take one line from the CSV:

```csv
1,John Doe,30
```

### In Python:

* `open()` reads the bytes.
* `csv` module splits by `,`.
* Returns `{ 'customer_id': '1', 'name': 'John Doe', 'age': '30' }`.

### In Athena:

* **InputFormat** loads the raw text from S3.
* **SerDe** splits by `,` and applies schema (`STRING, STRING, INT`).
* Athena query engine returns a structured row: `(1, 'John Doe', 30)`.

---

## 5. Extending Beyond CSV

Now imagine switching file formats:

* In Python → you’d import `json` or `pyarrow.parquet` instead of `csv`.
* In Athena → you just swap the SerDe and InputFormat to JSON/Parquet equivalents.

For example, a Parquet table in Athena:

```sql
ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
```

This is how big data systems generalize parsing across many formats.

---

## Conclusion

If you know how Python’s `csv` module works, you already understand the basics of Athena’s SerDe, InputFormat, and OutputFormat:

* `open()` ↔ InputFormat
* `csv.reader()` / `DictReader` ↔ SerDe
* `writer.writerow()` ↔ OutputFormat

The difference is **scale**. Athena and Spark need to split work across many nodes, handle petabytes of data, and support dozens of formats. That’s why these extra building blocks exist.


