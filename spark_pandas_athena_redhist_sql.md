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


