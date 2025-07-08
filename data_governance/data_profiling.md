# What is Data Profiling and Why Do We Need It?

Data profiling is the process of examining data from existing sources to collect statistics, summaries about the data’s structure, content. It’s a foundational step in understanding the data .
This data can be used to detect any deviations in the data (in some context data anomaly or data quality).

## Why is Data Profiling Important?

- **Data Quality Assurance:** Identify missing, inconsistent, or erroneous data.
- **Better Decision Making:** Understand data distributions, outliers, and patterns.
- **Schema Understanding:** Discover data types, cardinality, and relationships.
- **Compliance & Governance:** Monitor data completeness and integrity over time.
- **ETL Optimization:** Design efficient data pipelines by knowing data characteristics.

---

# Key Metrics to Collect During Data Profiling

Data profiling metrics span multiple categories: dataset-level, column-level, and value-level. Here’s a comprehensive list:

## Dataset-Level Metrics

| Metric       | Description               |
|--------------|---------------------------|
| RowCount     | Total number of rows       |
| Size (MB/GB) | Data size                  |
| ColumnCount  | Number of columns          |

## Column-Level Metrics

| Metric             | Description                                                                                   |
|--------------------|-----------------------------------------------------------------------------------------------|
| ColumnCount        | Number of rows for that column                                                                |
| NullCount / EmptyCount | Count of null or empty values                                                               |
| Completeness Ratio  | `(Non-null + Non-empty) / TotalCount`                                                        |
| Uniqueness Ratio    | `DistinctCount / ColumnCount`                                                                 |
| DistinctValueCount (aka Cardinality) | Number of distinct values                                                                     |
| Size (MB)           | Data size for the column                                                                      |

### Additional Metrics for String Columns

| Metric           | Description                  |
|------------------|------------------------------|
| MaxLength        | Maximum string length         |
| MinLength        | Minimum string length         |
| AverageLength    | Average string length         |
| Mode             | Most frequent value (derived from value counts) |

### Additional Metrics for Numeric Columns

| Metric      | Description                   |
|-------------|-------------------------------|
| Q1 (25th percentile) | First quartile                    |
| Median (50th percentile) | Second quartile                |
| Q3 (75th percentile) | Third quartile                   |
| Mean        | Average value                  |
| Standard Deviation | Variation in values            |
| Min / Max   | Minimum and maximum values      |
| Sum         | Total sum of values             |

## Column-Value Metrics

For string columns especially, profiling the frequency distribution of distinct values is critical:

| Metric          | Description                                   |
|-----------------|-----------------------------------------------|
| ColumnValue     | Distinct column value (null treated separately) |
| RowCount       | Number of rows containing that value           |
| SQLRuleName    | Optional: to associate profiling with specific validation or business rules |



# Need for a Platform-Agnostic Data Profiling System

Enterprises often maintain heterogeneous data platforms (on-prem, cloud, multiple vendors). A robust profiling system must:

- Run on any platform (Hive, Redshift, Snowflake, Athena, Databricks, etc.).
- Support incremental profiling for partitioned and large datasets.
- Store profiling results in a **generic, standardized schema** for easy querying and integration.
- Allow extensible metrics and rules for custom profiling needs.
- Integrate with data quality monitoring and lineage frameworks.



# Generic Schema to Store Profiling Results

Below is a proposed schema design for storing profiling data in tables, capturing dataset, column, and value-level metrics.

## Column Metrics Table

| Field               | Type      | Description                                  |
|---------------------|-----------|----------------------------------------------|
| dataset_name        | string    | Dataset identifier                            |
| granularity         | string    | “Entire Table” or “Partition”                 |
| granularity_value   | string    | Partition name or NULL for entire table       |
| profile_date        | datetime  | Timestamp when profiling was run              |
| column_name         | string    | Column name                                   |
| data_type           | string    | Inferred data type (e.g., string, int)       |
| row_count           | int       | Total rows                                    |
| non_null_count      | int       | Count of non-null values                       |
| null_count          | int       | Count of nulls                                |
| empty_string_count  | int       | For strings, count of empty strings ('' (not same as null)          |
| distinct_count      | int       | Number of distinct values (may be approximate)|
| uniqueness_ratio    | float     | distinct_count / row_count                     |
| completeness_ratio  | float     | (non_null + non_empty) / row_count             |
| min_value           | string    | Min value (supports all types)                 |
| max_value           | string    | Max value                                      |
| mean                | float     | Numeric mean                                   |
| std_dev             | float     | Numeric standard deviation                      |
| sum                 | float     | Numeric sum                                    |
| q1                  | float     | 25th percentile (numeric)                       |
| median              | float     | 50th percentile (numeric)                       |
| q3                  | float     | 75th percentile (numeric)                       |
| min_length          | int       | Min string length                               |
| max_length          | int       | Max string length                               |
| average_length      | float     | Average string length                           |
| data_size_mb        | float     | Size of column data in MB                        |


## Table Metrics Table

| Field            | Type      | Description                                  |
|------------------|-----------|----------------------------------------------|
| dataset_name     | string    | Dataset identifier                            |
| granularity      | string    | “Entire Table” or “Partition”                 |
| granularity_value| string    | Partition name or NULL                         |
| profile_date     | datetime  | Timestamp when profiling was run              |
| row_count        | int       | Total number of rows                          |
| column_count     | int       | Number of columns                             |
| data_size_gb     | float     | Data size in GB                              |



## Column-Value Metrics Table

| Field          | Type     | Description                                |
|----------------|----------|--------------------------------------------|
| dataset_name   | string   | Dataset identifier                          |
| granularity    | string   | “Entire Table” or “Partition”               |
| granularity_value | string | Partition name or NULL                       |
| profile_date   | datetime | Timestamp profiling was run                 |
| column_name    | string   | Column name                                 |
| column_value   | string   | Distinct column value (null treated separately) |
| row_count     | int      | Number of rows with this column value       |
| sql_rule_name  | string   | Optional: associated validation rule        |

