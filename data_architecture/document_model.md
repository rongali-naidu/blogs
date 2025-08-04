# I Wish I Knew: The Document Data Model is Just JSON

---

### Introduction: Let's Demystify the Document Model

People often present the document model as a fundamentally different data model, tied exclusively to NoSQL databases like MongoDB or DynamoDB. But here’s the truth:

> **The document model is simply about storing records as JSON—or any equivalent flexible format.**

That’s it. Each record is a self-contained document, typically in JSON. Instead of distributing attributes across rigid table columns (as in relational databases), the document model stores everything together in a single JSON blob.
While JSON is common, saying the document model is "simply" JSON is slightly oversimplified. Document databases often use BSON (Binary JSON) or other binary formats internally for performance and additional features.
Generally, document model based systems are implemented using DynamoDB or MongoDB. Let’s explore how this model works in practice with PostgreSQL, JSON relevance in AWS Data Lakes, and querying JSON in Athena and Redshift.


---

### What is the Document Data Model? (Plain English Definition)

In traditional relational databases, each record is stored as a row with fixed columns.

**Relational Example:**

| user\_id | name  | age |
| -------- | ----- | --- |
| u123     | Alice | 30  |

In the **document model**, the same record would be stored as:

```json
{
  "user_id": "u123",
  "name": "Alice",
  "age": 30,
  "skills": ["SQL", "Python"],
  "orders": [
    { "order_id": 123, "product": "Book", "qty": 2 },
    { "order_id": 124, "product": "Pen", "qty": 5 }
  ]
}
```

Each record is a **document**, usually in JSON format, and it can include nested fields, arrays, and variable attributes.

---

### Advantages of the Document Data Model

1. **Schema Flexibility**
   Easily handle records with different fields; useful when data structure evolves frequently.

2. **Natural Data Representation**
   Great for modeling real-world entities (e.g., orders with nested items); supports nested structures and arrays natively.

3. **Ease of Ingestion**
   Ideal for storing data from APIs, IoT, and NoSQL sources without transformation.

4. **Faster Development**
   Developers can iterate quickly without needing schema migrations.

5. **Portability Across Systems**
   JSON is widely supported across databases, data lakes, and analytics tools.

6. **Efficient for Certain Workloads**
   Reading/writing whole documents is efficient when most fields are used together.

---

### PostgreSQL: Embracing the Document Model with JSONB

PostgreSQL brings the power of the document model into a relational world using the [`JSONB`](https://www.postgresql.org/docs/current/datatype-json.html) data type. This allows you to store and query JSON documents directly inside a column.

**Create Table with JSONB:**

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  data JSONB
);

INSERT INTO users (data)
VALUES ('{
  "user_id": "u123",
  "name": "Alice",
  "age": 30,
  "skills": ["SQL", "Python"],
  "orders": [
    { "order_id": 123, "product": "Book", "qty": 2 },
    { "order_id": 124, "product": "Pen", "qty": 5 }
  ]
}');
```

**Query JSONB Data:**

```sql
SELECT data->>'name' AS name,
       data->'skills' AS skills
FROM users
WHERE (data->>'age')::int > 25;
```

PostgreSQL also supports **GIN indexes** on JSONB columns, making document queries efficient. See more: [PostgreSQL JSON Indexing](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING)

---

### Data Lake Relevance

In modern data lakes, JSON is everywhere:

* Data from **DynamoDB**, **IoT devices**, and APIs often lands in **JSON format**.
* Using **Kinesis Data Streams** + **Firehose**, you can stream this data directly into **S3**, which can be queried from Athena and Redshift.



---

### Athena: Query JSON Directly from S3

AWS Athena allows you to run SQL queries directly on JSON files stored in S3.

**Query JSON Fields: String Data type**

In this example, lets assume that `data` column is stored as string.

```sql
SELECT json_extract_scalar(data, '$.user_id') AS user_id,
       json_extract_scalar(data, '$.name') AS name
FROM raw_json_table;
```

**Flattening JSON Arrays in Athena:**
Using the same JSON document's `orders` array:

```sql
SELECT json_extract_scalar(data, '$.user_id') AS user_id,
       json_extract_scalar(order, '$.order_id') AS order_id,
       json_extract_scalar(order, '$.product') AS product,
       CAST(json_extract_scalar(order, '$.qty') AS int) AS qty
FROM raw_json_table,
     UNNEST(cast(json_parse(json_extract(data, '$.orders')) AS array<json>)) AS t(order);
```



**Query JSON Fields: STRUCT Data type**

Here the `data` column is stored as `struct` data type,

```sql
CREATE EXTERNAL TABLE user_events (
  data struct<
    user_id:string,
    name:string,
    age:int,
    skills:array<string>,
    orders:array<struct<
      order_id:int,
      product:string,
      qty:int
    >>
  >
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS PARQUET
LOCATION 's3://your-bucket/path/';
```


```sql
SELECT
  data.user_id AS user_id,
  data.name AS name,
  data.age AS age,
  data.skills[1] AS first_skill,
  order.order_id,
  order.product,
  order.qty
FROM user_events
CROSS JOIN UNNEST(data.orders) AS t(order)
WHERE data.age > 25;
```


**Tip:** We can use **CTAS** (Create Table As Select) in Athena to convert JSON into **Parquet** for faster querying.

**Example: Convert JSON to Parquet using CTAS**

```sql
CREATE TABLE processed_orders
WITH (
  format = 'PARQUET',
  external_location = 's3://your-bucket/processed-orders/',
  write_compression = 'SNAPPY'
) AS
SELECT json_extract_scalar(data, '$.user_id') AS user_id,
       json_extract_scalar(order, '$.order_id') AS order_id,
       json_extract_scalar(order, '$.product') AS product,
       CAST(json_extract_scalar(order, '$.qty') AS int) AS qty
FROM raw_json_table,
     UNNEST(cast(json_parse(json_extract(data, '$.orders')) AS array<json>)) AS t(order);
```

This saves the query output as **Parquet files in S3**, improving future query speed and reducing cost.

More on Athena JSON querying: [AWS Athena JSON Functions](https://docs.aws.amazon.com/athena/latest/ug/functions-json.html)
More on CTAS: [Athena CTAS Documentation](https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html)


### Querying JSON through Redshift Spectrum

Redshift also allows querying JSON files stored in **S3** using **Spectrum**, extending Redshift's capabilities to your Data Lake.

**Query JSON Fields in Spectrum:**

```sql
SELECT json_extract_path_text(data, 'user_id') AS user_id,
       json_extract_path_text(data, 'name') AS name
FROM spectrum_logs;
```

Spectrum enables Redshift to work with JSON data at scale, combining the flexibility of document storage with powerful SQL analytics.


---

### Redshift: Working with SUPER and FLATTEN

Amazon Redshift supports semi-structured data through the **[`SUPER`](https://docs.aws.amazon.com/redshift/latest/dg/super.html)** data type and **JSON functions**.

**Create Table with SUPER:**

```sql
CREATE TABLE logs (
  id INT,
  data SUPER
);
```

**Insert and Query JSON Data:**

```sql
SELECT data.user_id, data.name
FROM logs
WHERE data.age > 25;
```

**Flattening JSON Arrays in Redshift:**

```sql
SELECT data.user_id, item.order_id, item.product, item.qty
FROM logs,
     FLATTEN(data.orders) AS item;
```

---

### When and Why to Flatten JSON

While JSON is great for flexibility, flattening it into columns is often necessary for:

* **Performance:** Querying structured data (like Parquet) is faster.
* **Analytics:** BI tools and dashboards work better with tabular data.
