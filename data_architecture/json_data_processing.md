
# Mastering JSON Processing Across Athena, Redshift Spectrum, Redshift SUPER, and Spark SQL

JSON has become a universal format for semi-structured data, powering modern applications, data lakes, and analytics. 
Whether you are querying JSON files directly from S3 or storing JSON inside relational systems, understanding how to handle JSON efficiently is key.


## Sample JSON Document

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

---

## 1) Athena: JSON Stored as STRING

### Query JSON fields from string column

```sql
WITH raw_json_table AS (
  SELECT
    '{"user_id":"u123","name":"Alice","age":30,"skills":["SQL","Python"],"orders":[{"order_id":123,"product":"Book","qty":2},{"order_id":124,"product":"Pen","qty":5}]}' AS data
)
SELECT
  json_extract_scalar(data, '$.user_id') AS user_id,
  json_extract_scalar(data, '$.name') AS name
FROM raw_json_table;
```

### Flatten nested array (`orders`)

```sql
WITH raw_json_table AS (
  SELECT
    '{"user_id":"u123","name":"Alice","age":30,"skills":["SQL","Python"],"orders":[{"order_id":123,"product":"Book","qty":2},{"order_id":124,"product":"Pen","qty":5}]}' AS data
)
SELECT
  json_extract_scalar(data, '$.user_id') AS user_id,
  json_extract_scalar(orders, '$.order_id') AS order_id,
  json_extract_scalar(orders, '$.product') AS product,
  CAST(json_extract_scalar(orders, '$.qty') AS int) AS qty
FROM raw_json_table,
     UNNEST(
       CAST(
         json_parse(
           json_format(json_extract(data, '$.orders'))
         ) AS array<json>
       )
     ) AS t(orders);
```

---

## 2) Athena: JSON Stored as STRUCT (via Glue schema on Parquet)

```sql
WITH user_events AS (
  SELECT CAST(
    ROW(
      'u123',
      'Alice',
      30,
      ARRAY['SQL', 'Python'],
      ARRAY[
        CAST(ROW(123, 'Book', 2) AS ROW(order_id INT, product VARCHAR, qty INT)),
        CAST(ROW(124, 'Pen', 5) AS ROW(order_id INT, product VARCHAR, qty INT))
      ]
    ) AS ROW(
      user_id VARCHAR,
      name VARCHAR,
      age INT,
      skills ARRAY<VARCHAR>,
      orders ARRAY<ROW(order_id INT, product VARCHAR, qty INT)>
    )
  ) AS data
)
SELECT
  data.user_id,
  data.name,
  data.age,
  order.order_id,
  order.product,
  order.qty
FROM user_events
CROSS JOIN UNNEST(data.orders) AS t(order);
```


## 3) Redshift Spectrum: JSON Stored as STRING


### Query JSON fields

```sql
WITH spectrum_logs(data) AS (
  SELECT '{"user_id":"u123","name":"Alice","age":30,"skills":["SQL","Python"],"orders":[{"order_id":123,"product":"Book","qty":2},{"order_id":124,"product":"Pen","qty":5}]}'::varchar AS data
)
SELECT
  json_extract_path_text(data, 'user_id') AS user_id,
  json_extract_path_text(data, 'name') AS name
FROM spectrum_logs;
```

---


---

## 5) Redshift Native Table: JSON Stored as SUPER

### Create native Redshift table with SUPER type


```sql
-- Create a temp table with SUPER type to simulate
CREATE TEMP TABLE user_events (data SUPER);

INSERT INTO user_events VALUES (
  JSON_PARSE('{
    "user_id":"u123",
    "name":"Alice",
    "age":30,
    "skills":["SQL","Python"],
    "orders":[
      {"order_id":123,"product":"Book","qty":2},
      {"order_id":124,"product":"Pen","qty":5}
    ]
  }')
);

SELECT
  data.user_id AS user_id,
  data.name AS name
FROM user_events

```

---

## 6) Spark SQL: JSON Stored as STRING (from S3 DataFrame)

### Query JSON fields

```sql
SELECT m.a FROM (
   SELECT from_json(MESSAGE, 'a INT, b DOUBLE') AS m FROM my_table
) WHERE m.b > 1.0
```
