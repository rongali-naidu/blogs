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

