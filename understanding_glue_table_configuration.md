# Understanding AWS Glue Input Format, Output Format, and JsonSerDe

AWS Glue is a powerful, serverless data integration service that enables you to extract, transform, and load (ETL) data across a variety of AWS services. When working with AWS Glue tables, understanding **InputFormat**, **OutputFormat**, and **SerDe** (Serializer/Deserializer) is crucial for efficient data processing. This blog explores how these components interact from end to end, including how AWS Glue interacts with Amazon S3, and clarifies the distinction between **table properties** and **SerDe properties** with practical examples.

---

## 1. What Are InputFormat, OutputFormat, and JsonSerDe?

### **InputFormat**
- Example:   org.apache.hadoop.mapred.TextInputFormat
- Responsible for reading data from storage (e.g., S3).
- Parses raw data into records that can be processed by Glue.

### **OutputFormat**
- Example:     org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat [Ignore Key doesnt refer to the keys in JSON Docs or column names..they refer to the Key in K,V format of how Hadoop reads like record indetifier]
- Writes processed data back to storage.
- Controls how Glue outputs the results of transformations.

### **JsonSerDe**
- Example:     org.openx.data.jsonserde.JsonSerDe
- A SerDe (Serializer/Deserializer) for parsing  the records.
- Glue uses **org.openx.data.jsonserde.JsonSerDe** to handle JSON.

---

## 2. How AWS Glue Reads and Writes Data

When AWS Glue interacts with a data source (e.g., S3), it relies on the Hadoop ecosystem’s **InputFormat** and **OutputFormat** classes. These classes define how the data is read from and written to storage.

### **Step-by-Step Data Flow**

1. **InputFormat** reads data from S3 (via the **[S3AFileSystem](https://github.com/apache/hadoop/blob/trunk/hadoop-tools/hadoop-aws/src/main/java/org/apache/hadoop/fs/s3a/S3AFileSystem.java?utm_source=chatgpt.com)** library).
2. **JsonSerDe** deserializes the raw data into structured records.
3. AWS Glue processes the data using Spark or Hive.
4. **OutputFormat** serializes the records back to a specified format and writes them to S3.
5. In case of Athena, it displays the data. 
### **Interaction with S3A Library**

InputFormat uses the `org.apache.hadoop.fs.s3a.S3AFileSystem` class to interact with S3. This class handles:

- **Connecting to S3**: Using AWS SDK to authenticate and retrieve objects.
- **Optimizing Performance**: Supporting multipart uploads and parallel reads.
- **Handling Data Consistency**: Ensuring correct data retrieval with eventual consistency.

When a Glue job starts, the **InputFormat** communicates with **S3AFileSystem** to list, split, and read objects in parallel.

---

## 3. Example: Parsing JSON in AWS Glue

### JSON File in S3

```json
{"id": 1, "name": "Alice", "age": 30}
{"id": 2, "name": "Bob", "age": 25}
```

### Glue Table Definition

```sql
CREATE EXTERNAL TABLE people (
    id INT,
    name STRING,
    age INT
)
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://your-bucket/json_data/'
SERDE 'org.openx.data.jsonserde.JsonSerDe'
```

### How It Works

1. **InputFormat**: TextInputFormat reads line-by-line.
2. **SerDe**: JsonSerDe parses each line into key-value pairs.
3. **OutputFormat**: HiveIgnoreKeyTextOutputFormat outputs results, ignoring the key part.

### Querying in Athena or Glue

```sql
SELECT * FROM people;
```

This query reads the JSON, parses it using JsonSerDe, and outputs it in tabular form.

---

## 4. Difference Between Table Properties and SerDe Properties

| Feature           | Table Properties                           | SerDe Properties                             |
| ----------------- | ------------------------------------------ | -------------------------------------------- |
| **Purpose**       | Configure table-level behavior.            | Customize how data is parsed and serialized. |
| **Scope**         | Applies to the entire Glue table.          | Specific to how data is read and written.    |
| **Examples**      | `classification`, `skip.header.line.count` | `ignore.malformed.json`, `timestamp.format`  |
| **Example Usage** | Set compression or file format.            | Control JSON parsing behavior.               |

### Example Configuration

```sql
TBLPROPERTIES (
  'classification'='json',
  'skip.header.line.count'='1'
)
WITH SERDEPROPERTIES (
  'ignore.malformed.json'='true',
  'dots.in.keys'='false'
);
```

### Examples for JSON, CSV, Parquet, and Avro

1. **JSON Table Configuration**

```sql
CREATE EXTERNAL TABLE json_table (
    id INT,
    data STRING
)
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://your-bucket/json/'
SERDE 'org.openx.data.jsonserde.JsonSerDe'
TBLPROPERTIES ('classification'='json');
```

2. **CSV Table Configuration**

```sql
CREATE EXTERNAL TABLE csv_table (
    id INT,
    name STRING
)
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://your-bucket/csv/'
TBLPROPERTIES ('classification'='csv', 'skip.header.line.count'='1');
```

3. **Parquet Table Configuration**

```sql
CREATE EXTERNAL TABLE parquet_table (
    id INT,
    name STRING
)
STORED AS PARQUET
LOCATION 's3://your-bucket/parquet/'
TBLPROPERTIES ('classification'='parquet');
```

4. **Avro Table Configuration**

```sql
CREATE EXTERNAL TABLE avro_table (
    id INT,
    name STRING
)
STORED AS INPUTFORMAT 'org.apache.hadoop.hive.ql.io.avro.AvroContainerInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.avro.AvroContainerOutputFormat'
LOCATION 's3://your-bucket/avro/'
SERDE 'org.apache.hadoop.hive.serde2.avro.AvroSerDe'
TBLPROPERTIES ('classification'='avro');
```

---

## 5. How Different AWS Services Use InputFormat, OutputFormat, and SerDe

--Note: Verify below point...looks like SPark uses internal inputformat
* 

| Service            | Usage of InputFormat/OutputFormat/SerDe                          |
|--------------------|------------------------------------------------------------------|
| **AWS Glue**       | Uses InputFormat to read from S3, SerDe to parse data, and OutputFormat to write back to S3. |
| **Amazon Athena**  | Leverages InputFormat and SerDe to query structured/unstructured data in S3. |
| **Redshift Spectrum** | Uses InputFormat and SerDe to query external data in S3 as Redshift tables. |
| **Amazon EMR**     | Uses InputFormat and SerDe for Hive and Spark jobs to process large datasets. |
| **Spark on AWS**   | Utilizes Hadoop InputFormat to read S3 data and SerDe for parsing during Spark SQL execution. |

---

## 6. Summary

Understanding how **InputFormat**, **OutputFormat**, and **JsonSerDe** work in AWS Glue is key to efficient data processing. Here’s a recap:

- **InputFormat**: Reads raw data (e.g., from S3).
- **OutputFormat**: Writes processed data.
- **JsonSerDe**: Parses JSON records into tabular format.
- **S3AFileSystem**: Manages S3 access under the hood.

