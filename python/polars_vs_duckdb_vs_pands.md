
# Polars vs Pandas vs DuckDB — The Modern DataFrame Showdown (with Examples)

### Introduction

Data processing in Python has evolved significantly over the past few years.
While **Pandas** has been the de facto standard for tabular data for over a decade, newer technologies like **Polars** and **DuckDB** are redefining what’s possible in terms of **speed, scalability, and interoperability**.

This post compares **Pandas**, **Polars**, and **DuckDB** — three powerful tools for analytical data processing — across design, performance, and usability.
We’ll explain when to use each, provide examples, and share benchmark insights.

---

##  1. Pandas — The Veteran of Python Analytics

* **Website:** [https://pandas.pydata.org](https://pandas.pydata.org)
* **Created by:** Wes McKinney (2010)
* **Core idea:** Provide a powerful **DataFrame API** for tabular data, built on top of **NumPy**.

Pandas made Python the language of data science. It’s intuitive, feature-rich, and integrates with almost every data tool.
However, its architecture is **single-threaded** and **row-oriented**, meaning performance drops for datasets larger than a few million rows.

**Best for:** Small-to-medium datasets, data cleaning, exploration, and prototyping.

---

## 2. Polars — The Next-Gen Rust-Powered DataFrame

* **Website:** [https://pola.rs](https://pola.rs)
* **Created by:** Ritchie Vink (2021)
* **Core idea:** High-performance **columnar DataFrame engine** written in **Rust**, inspired by Apache Arrow.

Polars introduces **lazy computation**, **multi-threaded execution**, and **query optimization** — similar to Spark or DuckDB but fully embedded and blazing fast.
It’s **Arrow-native**, meaning zero-copy interchange with DuckDB, Pandas, and many other libraries.

**Best for:** Analytical workloads, aggregations, joins, and transformations on datasets up to tens of GBs — all from your laptop.

---

##  3. DuckDB — The "SQLite for Analytics"

* **Website:** [https://duckdb.org](https://duckdb.org)
* **Created by:** Hannes Mühleisen and Mark Raasveldt (2019)
* **Core idea:** An **embedded OLAP SQL database** that can query **CSV, Parquet, or Arrow** files directly — no server required.

DuckDB is built for analytical (OLAP) workloads. It’s **columnar**, **vectorized**, and **ACID-compliant**, yet lightweight enough to run inside your Python or R process.
It’s ideal for analysts who prefer SQL over Pythonic DataFrames.

**Best for:** Querying Parquet/CSV files, joining large datasets, or powering lightweight analytics pipelines.

---

## ⚙️ Architecture Overview

| Feature               | **Pandas**             | **Polars**                   | **DuckDB**                   |
| --------------------- | ---------------------- | ---------------------------- | ---------------------------- |
| **Core Language**     | Python (C extensions)  | Rust                         | C++                          |
| **Execution Model**   | Eager, single-threaded | Lazy + eager, multi-threaded | SQL query engine             |
| **Data Layout**       | Row-oriented           | Columnar (Arrow)             | Columnar (vectorized)        |
| **Memory Efficiency** | Medium                 | High                         | High                         |
| **Multi-threading**   | ❌ No                  | ✅ Yes                        | ✅ Yes                        |
| **Primary Interface** | Python DataFrame       | Python/Rust DataFrame        | SQL                          |
| **Best Use Case**     | Data cleaning          | Analytical pipelines         | Querying large files via SQL |

---

## Example 1 — Reading and Aggregating Data

Let’s use a simple dataset `sales.csv`:

```csv
order_id,region,sales
1,East,100
2,West,200
3,East,300
4,South,150
5,West,250
```

We’ll calculate **total sales per region** using all three.

---

### 🐍 Pandas

```python
import pandas as pd

df = pd.read_csv("sales.csv")
result = df.groupby("region")["sales"].sum().reset_index()
print(result)
```

**Output**

```
  region  sales
0   East    400
1  South    150
2   West    450
```

✅ Simple syntax, but not optimized for large datasets.

---

### Polars

```python
import polars as pl

df = pl.read_csv("sales.csv")
result = df.group_by("region").agg(pl.col("sales").sum())
print(result)
```

**Output**

```
shape: (3, 2)
┌────────┬───────┐
│ region │ sales │
│ ---    │ ---   │
│ str    │ i64   │
├────────┼───────┤
│ East   │ 400   │
│ West   │ 450   │
│ South  │ 150   │
└────────┴───────┘
```

✅ Fast, multi-threaded, and uses SIMD vectorization under the hood.
✅ Handles Arrow and Parquet natively.

---

### DuckDB

```python
import duckdb

result = duckdb.query("""
    SELECT region, SUM(sales) AS total_sales
    FROM 'sales.csv'
    GROUP BY region
""").to_df()

print(result)
```

**Output**

```
  region  total_sales
0   East          400
1   South         150
2   West          450
```

✅ SQL-friendly
✅ Extremely fast for file-based analytics
✅ Supports joins, filters, and even subqueries directly on CSV/Parquet files

---

## Example 2 — Lazy Computation (Polars Exclusive)

Polars introduces **lazy execution**, which lets it build an optimized query plan before running it.

```python
import polars as pl

lf = pl.scan_csv("sales.csv")  # LazyFrame (not loaded yet)

result = (
    lf.filter(pl.col("sales") > 150)
      .group_by("region")
      .agg(pl.col("sales").mean().alias("avg_sales"))
      .sort("avg_sales", descending=True)
      .collect()
)

print(result)
```

✅ Reads only necessary columns
✅ Combines filters + aggregations in one optimized plan
✅ Executes in parallel

Equivalent Pandas code:

```python
import pandas as pd

df = pd.read_csv("sales.csv")
result = (df[df.sales > 150]
          .groupby("region")["sales"]
          .mean()
          .reset_index()
          .sort_values("sales", ascending=False))
```

❌ Loads entire file in memory
❌ Executes step by step (no global optimization)

---

## Example 3 — Querying Parquet Data

All three tools can read **Parquet**, but Polars and DuckDB are significantly faster.

### Polars

```python
df = pl.read_parquet("sales.parquet")
```

### DuckDB

```python
result = duckdb.query("SELECT * FROM 'sales.parquet' WHERE sales > 100").to_df()
```

### Pandas

```python
df = pd.read_parquet("sales.parquet")
```

✅ Works fine, but Pandas relies on PyArrow/Fastparquet backends.
⚠️ Lacks advanced predicate pushdown and parallel read capabilities.

---

##  Example 4 — Interoperability (Arrow Format)

You can mix these tools seamlessly thanks to **Apache Arrow**.

```python
import polars as pl
import duckdb
import pandas as pd

# Load data in Polars
pl_df = pl.read_csv("sales.csv")

# Convert to Pandas
pd_df = pl_df.to_pandas()

# Query directly in DuckDB from Polars (Arrow backend)
duckdb.query("SELECT region, SUM(sales) FROM pl_df GROUP BY region").to_df()
```

✅ Zero-copy data sharing
✅ Combine Polars’ speed + DuckDB’s SQL expressiveness
✅ Pandas remains useful for visualization or integration



### Performance testing

```sh
pip install polars duckdb pandas

```
```python
import pandas as pd
import numpy as np

N = 1_000_000
df = pd.DataFrame({
    "order_id": np.arange(N),
    "region": np.random.choice(["East", "West", "South", "North"], size=N),
    "sales": np.random.randint(10, 1000, size=N)
})

df.to_csv("sales.csv", index=False)
df.to_parquet("sales.parquet")  # optional Parquet for DuckDB / Polars
```

```python

import pandas as pd
import polars as pl
import duckdb
import time

# --- Pandas ---
start = time.time()
pdf = pd.read_csv("sales.csv")
pandas_result = pdf.groupby("region")["sales"].sum()
print("Pandas Result:\n", pandas_result)
print("Pandas Time:", time.time() - start)

# --- Polars ---
start = time.time()
pl_df = pl.read_csv("sales.csv")
polars_result = pl_df.groupby("region").agg(pl.col("sales").sum())
print("Polars Result:\n", polars_result)
print("Polars Time:", time.time() - start)

# --- DuckDB ---
start = time.time()
duck_result = duckdb.query("""
    SELECT region, SUM(sales) AS total_sales
    FROM 'sales.csv'
    GROUP BY region
""").to_df()
print("DuckDB Result:\n", duck_result)
print("DuckDB Time:", time.time() - start)
```
```python
import sys, platform, os
import pandas as pd, polars as pl, duckdb
import psutil

print("Python:", sys.version)
print("Pandas:", pd.__version__)
print("Polars:", pl.__version__)
print("DuckDB:", duckdb.__version__)
print("Platform:", platform.platform())
print("Processor:", platform.processor())
print("CPU Cores:", os.cpu_count())
print("Physical cores:", psutil.cpu_count(logical=False))
print("Logical cores:", psutil.cpu_count(logical=True))
print("Total RAM (GB):", round(psutil.virtual_memory().total / 1e9, 2))
```

