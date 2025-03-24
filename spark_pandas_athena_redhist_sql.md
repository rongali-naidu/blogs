Here's a comprehensive guide to performing equivalent data operations across **Spark DataFrame**, **Pandas DataFrame**, **Spark SQL**, **Athena SQL**, and **Redshift SQL**.

### **1. Data Creation**
<table>
<tr>
  <td> Spark DataFrame </td> 
  <td> Pandas DataFrame </td>
  <td> Spark SQL </td>  
  <td> Athena SQL   </td>  
  <td> Redshift SQL   </td>    
</tr>
  
<tr>
  <td>
  
  ```python
  from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DateType
  from pyspark.sql.functions import col, asc, desc, lit
  import datetime
  
  # Employee Schema and Data
  employees_schema = StructType([
      StructField("first_name", StringType(), True),
      StructField("last_name", StringType(), True),
      StructField("joining_date", DateType(), True),
      StructField("department_id", IntegerType(), True),
      StructField("salary", IntegerType(), True)
  ])
  
  employees_data = [
      ['Alice', 'Smith', datetime.date(2015, 6, 21), 1, 75000],
      ['Bob', 'Johnson', datetime.date(2018, 9, 15), 2, 65000],
      ['Charlie', 'Williams', datetime.date(2020, 3, 10), 3, 60000],
      ['Diana', 'Brown', datetime.date(2017, 12, 5), 4, 80000],
      ['Ethan', 'Davis', datetime.date(2019, 7, 22), 2, 72000],
      ['Fiona', 'Miller', datetime.date(2021, 1, 30), 3, 58000],
      ['George', 'Wilson', datetime.date(2016, 11, 18), 1, 90000]
  ]
  
  # Department Schema and Data
  departments_schema = StructType([
      StructField("department_id", IntegerType(), True),
      StructField("department_name", StringType(), True)
  ])
  
  departments_data = [
      [1, 'Engineering'],
      [2, 'Marketing'],
      [3, 'Finance'],
      [4, 'Human Resources'],
      [5, 'Operations']
  ]
  
  # Create DataFrames
  employees_df = spark.createDataFrame(data=employees_data, schema=employees_schema)
  departments_df = spark.createDataFrame(data=departments_data, schema=departments_schema)
  
  # Display DataFrames
  employees_df.show()
  departments_df.show()
  ```
  </td>
  <td>
    
  ```python

import pandas as pd
import numpy as np
import datetime

# Employee Data and Schema
employees_data = [
    ['Alice', 'Smith', datetime.date(2015, 6, 21), 1, 75000],
    ['Bob', 'Johnson', datetime.date(2018, 9, 15), 2, 65000],
    ['Charlie', 'Williams', datetime.date(2020, 3, 10), 3, 60000],
    ['Diana', 'Brown', datetime.date(2017, 12, 5), 4, 80000],
    ['Ethan', 'Davis', datetime.date(2019, 7, 22), 2, 72000],
    ['Fiona', 'Miller', datetime.date(2021, 1, 30), 3, 58000],
    ['George', 'Wilson', datetime.date(2016, 11, 18), 1, 90000]
]

# Employee Schema (Dtypes)
employees_columns = ["first_name", "last_name", "joining_date", "department_id", "salary"]
employees_dtypes = {
    "first_name": "string",          # String type
    "last_name": "string",           # String type
    "joining_date": "datetime64",    # Date type
    "department_id": "int32",        # Integer (32-bit)
    "salary": "int64"                # Integer (64-bit)
}

# Create Pandas DataFrame with Schema
employees_df = pd.DataFrame(employees_data, columns=employees_columns).astype(employees_dtypes)

# Display DataFrame Info and Data
print(employees_df.info())
print(employees_df.head())

# Department Data and Schema
departments_data = [
    [1, 'Engineering'],
    [2, 'Marketing'],
    [3, 'Finance'],
    [4, 'Human Resources'],
    [5, 'Operations']
]

departments_columns = ["department_id", "department_name"]
departments_dtypes = {
    "department_id": "int32",
    "department_name": "string"
}

# Create Departments DataFrame with Schema
departments_df = pd.DataFrame(departments_data, columns=departments_columns).astype(departments_dtypes)

# Display DataFrame Info and Data
print(departments_df.info())
print(departments_df.head())

  ```    
  </td>
  <td>
  
  ```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DateType
import datetime

# Initialize Spark session
spark = SparkSession.builder.appName("SparkSQLExample").getOrCreate()

# Employee Schema and Data
employees_schema = StructType([
    StructField("first_name", StringType(), True),
    StructField("last_name", StringType(), True),
    StructField("joining_date", DateType(), True),
    StructField("department_id", IntegerType(), True),
    StructField("salary", IntegerType(), True)
])

employees_data = [
    ['Alice', 'Smith', datetime.date(2015, 6, 21), 1, 75000],
    ['Bob', 'Johnson', datetime.date(2018, 9, 15), 2, 65000],
    ['Charlie', 'Williams', datetime.date(2020, 3, 10), 3, 60000],
    ['Diana', 'Brown', datetime.date(2017, 12, 5), 4, 80000],
    ['Ethan', 'Davis', datetime.date(2019, 7, 22), 2, 72000],
    ['Fiona', 'Miller', datetime.date(2021, 1, 30), 3, 58000],
    ['George', 'Wilson', datetime.date(2016, 11, 18), 1, 90000]
]

# Department Schema and Data
departments_schema = StructType([
    StructField("department_id", IntegerType(), True),
    StructField("department_name", StringType(), True)
])

departments_data = [
    [1, 'Engineering'],
    [2, 'Marketing'],
    [3, 'Finance'],
    [4, 'Human Resources'],
    [5, 'Operations']
]

# Create DataFrames
employees_df = spark.createDataFrame(data=employees_data, schema=employees_schema)
departments_df = spark.createDataFrame(data=departments_data, schema=departments_schema)

# Register DataFrames as Temporary Views
employees_df.createOrReplaceTempView("employees")
departments_df.createOrReplaceTempView("departments")

```
  </td>
  <td>
  
  ```sql

-- Create employees table
CREATE EXTERNAL TABLE IF NOT EXISTS default.employees (
    first_name STRING,
    last_name STRING,
    joining_date DATE,
    department_id INT,
    salary INT
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION 's3://your-bucket/employees/';

-- Create departments table
CREATE EXTERNAL TABLE IF NOT EXISTS default.departments (
    department_id INT,
    department_name STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION 's3://your-bucket/departments/';

employees.csv
Alice,Smith,2015-06-21,1,75000
Bob,Johnson,2018-09-15,2,65000
Charlie,Williams,2020-03-10,3,60000
Diana,Brown,2017-12-05,4,80000
Ethan,Davis,2019-07-22,2,72000
Fiona,Miller,2021-01-30,3,58000
George,Wilson,2016-11-18,1,90000

departments.csv
1,Engineering
2,Marketing
3,Finance
4,Human Resources
5,Operations


  ```
  </td>
  <td>
  
  ```sql
-- Create Employees Table
CREATE TABLE public.employees (
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    joining_date DATE,
    department_id INT,
    salary INT
);

-- Create Departments Table
CREATE TABLE public.departments (
    department_id INT,
    department_name VARCHAR(100)
);

-- Insert Data into Employees Table
INSERT INTO public.employees (first_name, last_name, joining_date, department_id, salary) 
VALUES 
    ('Alice', 'Smith', '2015-06-21', 1, 75000),
    ('Bob', 'Johnson', '2018-09-15', 2, 65000),
    ('Charlie', 'Williams', '2020-03-10', 3, 60000),
    ('Diana', 'Brown', '2017-12-05', 4, 80000),
    ('Ethan', 'Davis', '2019-07-22', 2, 72000),
    ('Fiona', 'Miller', '2021-01-30', 3, 58000),
    ('George', 'Wilson', '2016-11-18', 1, 90000);

-- Insert Data into Departments Table
INSERT INTO public.departments (department_id, department_name) 
VALUES 
    (1, 'Engineering'),
    (2, 'Marketing'),
    (3, 'Finance'),
    (4, 'Human Resources'),
    (5, 'Operations');

-- Commit the transaction
COMMIT;

  ```
 </td>
</tr>
</table>

### **2. Data Operations**

| **Operation**                      | **PySpark**                                                                 | **Pandas**                                                                | **Spark SQL**                             | **Athena SQL**                               | **Redshift SQL**                               |
|-----------------------------------|---------------------------------------------------------------------------|--------------------------------------------------------------------------|-------------------------------------------|-----------------------------------------------|-----------------------------------------------|
| **Count**                         | `employees_df.count()`                                                     | `len(employees_df)`                                                      | `SELECT COUNT(*) FROM employees`          | `SELECT COUNT(*) FROM employees`              | `SELECT COUNT(*) FROM employees`              |
| **Random Rows (LIMIT 10)**         | `employees_df.limit(10).show()`                                            | `employees_df.sample(n=10)`                                              | `SELECT * FROM employees LIMIT 10`        | `SELECT * FROM employees LIMIT 10`            | `SELECT * FROM employees LIMIT 10`            |
| **Sample Rows**                   | `employees_df.sample(False, 0.1)`                                          | `employees_df.sample(frac=0.1)`                                          | `SELECT * FROM employees TABLESAMPLE(10)` | `SELECT * FROM employees TABLESAMPLE BERNOULLI (10)` | `SELECT * FROM employees TABLESAMPLE BERNOULLI (10)` |
| **Print Schema**                  | `employees_df.printSchema()`                                                | `employees_df.dtypes`                                                    | `DESCRIBE employees`                       | `DESCRIBE employees`                           | `SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'employees'` |
| **Selecting Columns**              | `employees_df.select("first_name", "salary").show()`                      | `employees_df[["first_name", "salary"]]`                               | `SELECT first_name, salary FROM employees` | `SELECT first_name, salary FROM employees`      | `SELECT first_name, salary FROM employees`      |
| **Column Rename**                 | `employees_df.withColumnRenamed("salary", "monthly_salary").show()`      | `employees_df.rename(columns={"salary": "monthly_salary"})`             | `ALTER TABLE employees RENAME COLUMN salary TO monthly_salary` | `ALTER TABLE employees RENAME COLUMN salary TO monthly_salary` | `ALTER TABLE employees RENAME COLUMN salary TO monthly_salary` |
| **Column Data Type Change**       | `employees_df.withColumn("salary", col("salary").cast("float")).show()` | `employees_df["salary"] = employees_df["salary"].astype(float)`         | `ALTER TABLE employees ALTER COLUMN salary TYPE FLOAT` | `ALTER TABLE employees ALTER COLUMN salary TYPE FLOAT` | `ALTER TABLE employees ALTER COLUMN salary TYPE FLOAT` |
| **Column Update Value**           | `employees_df.withColumn("salary", col("salary") + 5000).show()`          | `employees_df["salary"] += 5000`                                        | `UPDATE employees SET salary = salary + 5000` | `UPDATE employees SET salary = salary + 5000`   | `UPDATE employees SET salary = salary + 5000`   |
| **Column Null Value Default**     | `employees_df.fillna({"salary": 0}).show()`                                | `employees_df.fillna({"salary": 0})`                                    | `SELECT COALESCE(salary, 0) FROM employees` | `SELECT COALESCE(salary, 0) FROM employees`     | `SELECT COALESCE(salary, 0) FROM employees`     |
| **Column Addition**               | `employees_df.withColumn("bonus", col("salary") * 0.1).show()`            | `employees_df["bonus"] = employees_df["salary"] * 0.1`                  | `ALTER TABLE employees ADD bonus FLOAT`     | `ALTER TABLE employees ADD bonus FLOAT`         | `ALTER TABLE employees ADD bonus FLOAT`         |
| **Column Addition (Literal)**     | `employees_df.withColumn("constant", lit(100)).show()`                     | `employees_df["constant"] = 100`                                        | `SELECT *, 100 AS constant FROM employees` | `SELECT *, 100 AS constant FROM employees`      | `SELECT *, 100 AS constant FROM employees`      |
| **Column Dropped**                | `employees_df.drop("bonus").show()`                                        | `employees_df.drop("bonus", axis=1)`                                    | `ALTER TABLE employees DROP COLUMN bonus`   | `ALTER TABLE employees DROP COLUMN bonus`       | `ALTER TABLE employees DROP COLUMN bonus`       |
| **Data Filter (NULL Condition)**  | `employees_df.filter(col("salary").isNull()).show()`                       | `employees_df[employees_df["salary"].isnull()]`                          | `SELECT * FROM employees WHERE salary IS NULL` | `SELECT * FROM employees WHERE salary IS NULL`  | `SELECT * FROM employees WHERE salary IS NULL`  |
| **Data Filter (NOT NULL)**        | `employees_df.filter(col("salary").isNotNull()).show()`                    | `employees_df[employees_df["salary"].notnull()]`                         | `SELECT * FROM employees WHERE salary IS NOT NULL` | `SELECT * FROM employees WHERE salary IS NOT NULL` | `SELECT * FROM employees WHERE salary IS NOT NULL` |
| **Data Filter (Equality)**        | `employees_df.filter(col("salary") == 80000).show()`                        | `employees_df[employees_df["salary"] == 80000]`                          | `SELECT * FROM employees WHERE salary = 80000` | `SELECT * FROM employees WHERE salary = 80000`  | `SELECT * FROM employees WHERE salary = 80000`  |
| **Data Filter (LIKE)**            | `employees_df.filter(col("first_name").like("A%"))`                       | `employees_df[employees_df["first_name"].str.startswith("A")]`         | `SELECT * FROM employees WHERE first_name LIKE 'A%'` | `SELECT * FROM employees WHERE first_name LIKE 'A%'` | `SELECT * FROM employees WHERE first_name LIKE 'A%'` |
| **Data Filter (IN Condition)**    | `employees_df.filter(col("department_id").isin([1, 2])).show()`             | `employees_df[employees_df["department_id"].isin([1, 2])]`               | `SELECT * FROM employees WHERE department_id IN (1, 2)` | `SELECT * FROM employees WHERE department_id IN (1, 2)` | `SELECT * FROM employees WHERE department_id IN (1, 2)` |
| **Data Filter (Multiple Cond.)**  | `employees_df.filter((col("salary") > 60000) & (col("department_id") == 1))`| `employees_df[(employees_df["salary"] > 60000) & (employees_df["department_id"] == 1)]` | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1` | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1` | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1` |
| **Data Filter (Date Cond.)**      | `employees_df.filter(col("joining_date") > lit("2018-01-01")).show()`      | `employees_df[pd.to_datetime(employees_df["joining_date"]) > '2018-01-01']` | `SELECT * FROM employees WHERE joining_date > '2018-01-01'` | `SELECT * FROM employees WHERE joining_date > '2018-01-01'` | `SELECT * FROM employees WHERE joining_date > '2018-01-01'` |
| **Data Filter (Multiple Conditions)**       | `employees_df.filter((col("salary") > 60000) & (col("department_id") == 1)).show()`            | `employees_df[(employees_df["salary"] > 60000) & (employees_df["department_id"] == 1)]`            | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1`    | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1`    | `SELECT * FROM employees WHERE salary > 60000 AND department_id = 1`    |
| **Data Sorting**                            | `employees_df.orderBy("salary", ascending=False).show()`                                        | `employees_df.sort_values("salary", ascending=False)`                                              | `SELECT * FROM employees ORDER BY salary DESC`                          | `SELECT * FROM employees ORDER BY salary DESC`                          | `SELECT * FROM employees ORDER BY salary DESC`                          |
| **Data Grouping**                           | `employees_df.groupBy("department_id").count().show()`                                          | `employees_df.groupby("department_id").size().reset_index(name='count')`                           | `SELECT department_id, COUNT(*) FROM employees GROUP BY department_id`   | `SELECT department_id, COUNT(*) FROM employees GROUP BY department_id`   | `SELECT department_id, COUNT(*) FROM employees GROUP BY department_id`   |
| **Data Grouping with Aggregations & Alias** | `employees_df.groupBy("department_id").agg(sum("salary").alias("sum_salary"), max("joining_date").alias("max_joining_date")).show()`        | `employees_df.groupby("department_id").agg({'salary':'sum', 'joining_date':'max'}).reset_index()`           | `SELECT department_id, SUM(salary) AS sum_salary, MAX(joining_date) AS max_joining_date FROM employees GROUP BY department_id` | `SELECT department_id, SUM(salary) AS sum_salary, MAX(joining_date) AS max_joining_date FROM employees GROUP BY department_id` | `SELECT department_id, SUM(salary) AS sum_salary, MAX(joining_date) AS max_joining_date FROM employees GROUP BY department_id` |
| **Filter on Data Grouping (HAVING Clause)** | `employees_df.groupBy("department_id").agg(avg("salary").alias("avg_salary")).filter(col("avg_salary") > 60000).show()` | `df_grouped = employees_df.groupby("department_id")["salary"].mean().reset_index(name='avg_salary'); df_grouped[df_grouped["avg_salary"] > 60000]` | `SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id HAVING avg_salary > 60000` | `SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id HAVING avg_salary > 60000` | `SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id HAVING avg_salary > 60000` |
| **Dataframes Merging (Left Outer Join)**    | `employees_df.join(departments_df, employees_df.department_id == departments_df.department_id, "left").show()`    | `pd.merge(employees_df, departments_df, on="department_id", how='left')`          | `SELECT * FROM employees e LEFT OUTER JOIN departments d ON e.department_id = d.department_id` | `SELECT * FROM employees e LEFT OUTER JOIN departments d ON e.department_id = d.department_id` | `SELECT * FROM employees e LEFT OUTER JOIN departments d ON e.department_id = d.department_id` |


### **Read and Write Operations in Spark with AWS S3 (CSV, JSON, Parquet, Iceberg)**

| **Scenario**                               | **CSV (s3://bucket/path)**                                                                                     | **JSON (s3://bucket/path)**                                                                                    | **Parquet (s3://bucket/path)**                                                                                 | **Iceberg (Glue Catalog)**                                                                                       | **Glue Table (AWS Glue Catalog)**                                                                                  |
|--------------------------------------------|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| **Read with Header**                       | `spark.read.csv("s3://your-bucket/path", header=True)`<br>`spark.read.format("csv").option("header", "true").load("s3://your-bucket/path")` | N/A (JSON does not have headers)                                                                              | N/A (Parquet is schema-based)                                                                                   | N/A (Iceberg is schema-based)                                                                                    |`spark.table("glue_database.glue_table")`<br>`spark.sql("SELECT * FROM your_database.your_table")`<br>`spark.read.table("your_database.your_table")` <br>`spark.read.format("jdbc").option("dbtable", "database.table_name").load()` (for Glue tables via Athena JDBC)      |
| **Read without Header**                    | `spark.read.csv("s3://your-bucket/path", header=False)`<br>`spark.read.format("csv").option("header", "false").load("s3://your-bucket/path")` | N/A                                                                                                           | N/A                                                                                                            | N/A                                                                                                             | N/A                                                                                                                |
| **Read with Schema Inference**             | `spark.read.csv("s3://your-bucket/path", inferSchema=True)`<br>`spark.read.format("csv").option("inferSchema", "true").load("s3://your-bucket/path")` | `spark.read.json("s3://your-bucket/path")`<br>`spark.read.format("json").load("s3://your-bucket/path")`        | `spark.read.parquet("s3://your-bucket/path")`<br>`spark.read.format("parquet").load("s3://your-bucket/path")`  | `spark.read.format("iceberg").load("glue_catalog.db.iceberg_table")`                                             | `spark.read.format("catalog").option("table", "database.table_name").load()`                                       |
| **Read with Explicit Schema**              | `spark.read.csv("s3://your-bucket/path", schema=schema)`<br>`spark.read.format("csv").schema(schema).load("s3://your-bucket/path")` | `spark.read.schema(schema).json("s3://your-bucket/path")`<br>`spark.read.format("json").schema(schema).load("s3://your-bucket/path")` | `spark.read.schema(schema).parquet("s3://your-bucket/path")`<br>`spark.read.format("parquet").schema(schema).load("s3://your-bucket/path")` | `spark.read.format("iceberg").schema(schema).load("glue_catalog.db.iceberg_table")`                               | `spark.read.format("catalog").option("table", "database.table_name").schema(schema).load()`                        |
| **Read with Delimiter/Separator & Quote**  | `spark.read.csv("s3://your-bucket/path", sep="|", quote='"')`<br>`spark.read.format("csv").option("delimiter", "|").option("quote", '"').load("s3://your-bucket/path")` | N/A                                                                                                           | N/A                                                                                                            | N/A                                                                                                             | N/A                                                                                                                |
| **Read Handling Missing Values**           | `spark.read.format("csv").option("nullValue", "NA").load("s3://your-bucket/path")`                              | `spark.read.format("json").option("nullValue", "NA").load("s3://your-bucket/path")`                             | `spark.read.format("parquet").option("nullValue", "NA").load("s3://your-bucket/path")`                         | N/A (Iceberg manages nulls internally via schema)                                                                | Handled automatically in Glue tables via schema                                                                    |
| **Read Multiline Records**                 | `spark.read.format("csv").option("multiLine", "true").load("s3://your-bucket/path")`                            | `spark.read.format("json").option("multiLine", "true").load("s3://your-bucket/path")`                           | N/A                                                                                                            | N/A                                                                                                             | N/A                                                                                                                |
| **Write**                                  | `df.write.csv("s3://your-bucket/path")`<br>`df.write.format("csv").save("s3://your-bucket/path")`                | `df.write.json("s3://your-bucket/path")`<br>`df.write.format("json").save("s3://your-bucket/path")`             | `df.write.parquet("s3://your-bucket/path")`<br>`df.write.format("parquet").save("s3://your-bucket/path")`       | `df.write.format("iceberg").save("glue_catalog.db.iceberg_table")`                                               | `df.write.format("hive").mode("append").saveAsTable("database.table_name")`                                       |
| **Write with Schema**                      | `df.write.option("header", "true").csv("s3://your-bucket/path")`<br>`df.write.format("csv").option("header", "true").save("s3://your-bucket/path")` | N/A                                                                                                           | N/A                                                                                                            | N/A                                                                                                             | Schema is derived from DataFrame columns when using `saveAsTable`                                                  |
| **Write with Overwrite**                   | `df.write.mode("overwrite").csv("s3://your-bucket/path")`<br>`df.write.format("csv").mode("overwrite").save("s3://your-bucket/path")` | `df.write.mode("overwrite").json("s3://your-bucket/path")`<br>`df.write.format("json").mode("overwrite").save("s3://your-bucket/path")` | `df.write.mode("overwrite").parquet("s3://your-bucket/path")`<br>`df.write.format("parquet").mode("overwrite").save("s3://your-bucket/path")` | `df.write.format("iceberg").mode("overwrite").save("glue_catalog.db.iceberg_table")`                              | `df.write.mode("overwrite").saveAsTable("database.table_name")`                                                     |
| **Write with Overwrite & Partition By**    | `df.write.mode("overwrite").partitionBy("year").csv("s3://your-bucket/path")`<br>`df.write.format("csv").mode("overwrite").partitionBy("year").save("s3://your-bucket/path")` | `df.write.mode("overwrite").partitionBy("year").json("s3://your-bucket/path")`<br>`df.write.format("json").mode("overwrite").partitionBy("year").save("s3://your-bucket/path")` | `df.write.mode("overwrite").partitionBy("year").parquet("s3://your-bucket/path")`<br>`df.write.format("parquet").mode("overwrite").partitionBy("year").save("s3://your-bucket/path")` | `df.write.format("iceberg").mode("overwrite").partitionBy("year").save("glue_catalog.db.iceberg_table")`           | `df.write.mode("overwrite").partitionBy("year").saveAsTable("database.table_name")`                                |
| **Write with Compression & Partition By**  | `df.write.mode("overwrite").partitionBy("year").option("compression", "gzip").csv("s3://your-bucket/path")`      | `df.write.mode("overwrite").partitionBy("year").option("compression", "gzip").json("s3://your-bucket/path")`    | `df.write.mode("overwrite").partitionBy("year").option("compression", "snappy").parquet("s3://your-bucket/path")` | `df.write.format("iceberg").mode("overwrite").option("compression", "gzip").partitionBy("year").save("glue_catalog.db.iceberg_table")` | `df.write.mode("overwrite").option("compression", "snappy").partitionBy("year").saveAsTable("database.table_name")` |


