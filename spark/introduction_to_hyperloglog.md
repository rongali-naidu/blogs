## Introduction: probabilistic algorithms for counting 

Let’s say you want to:

* Know how many unique users visited your platform last week
* Measure how many distinct IPs hit a service per region
* Track how many unique search queries came in during peak hours

Naturally, you'd reach for a `COUNT(DISTINCT col)` — and it works. But when your data grows to **millions or billions of rows**, this query becomes expensive, especially in **distributed systems** where counting distinct values requires **shuffling and aggregating across partitions**.

The other day, while exploring [Deequ](https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/) — a data quality tool from AWS — I came across its use of `ApproxCountDistinct`. That raised a question:

> Why use a probabilistic algorithm when `COUNT(DISTINCT)` gives 100% accuracy?

Digging deeper, I realized this function comes from Apache Spark’s [approx\_count\_distinct](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.approx_count_distinct.html) API — a scalable approximation of distinct count, powered by the **HyperLogLog** algorithm.

At first, I was skeptical. For data profiling tasks, we’re already scanning the full dataset to compute metrics like count, sum, max, and min — so why not compute exact distinct counts too?

But then I realized

* While most metrics (`sum`, `max`, `min`) are **parallel**,
* `COUNT(DISTINCT)` **requires a shuffle** to deduplicate globally.

In contrast, `approx_count_distinct` **avoids shuffle**, making it much faster and more scalable. And it does this while using **fixed, tiny memory**  — even for very large datasets.

That made me curious:

> What’s this mysterious algorithm that makes approximate counting so powerful?

Turns out, it's **HyperLogLog** — a brilliant probabilistic algorithm not only used in Spark, but also in PostgreSQL, Redshift, BigQuery, Flink, etc

This curiosity led me down the path of understanding:

* How it evolved (from Flajolet-Martin to LogLog to HyperLogLog)
* The intuition behind the math
* What makes it work at scale
* And why it’s a game-changer for approximate computing

Here’s everything I learned — explained step-by-step, with examples and insights — so you too can appreciate the beauty behind `approx_count_distinct`.

Let me know if you want me to append the next section (`Evolution of HLL`) right after this for seamless flow.


### The Evolution: From Flajolet-Martin to HyperLogLog

In 1985, **Philippe Flajolet and G. Nigel Martin** introduced an algorithm to estimate the number of distinct elements in a data stream using **hashing and probabilistic estimation** — now known as the **Flajolet-Martin (FM) algorithm**. It was one of the first streaming-friendly algorithms to tackle the cardinality problem using clever statistical reasoning.

In 2003, Flajolet and others refined this idea further and introduced the **LogLog algorithm**, which observed that:

> The probability of a uniformly random binary string starting with `n` zeroes is about 1 in 2ⁿ.

This gave rise to a **logarithmic relationship between the max observed number of leading zeroes and the cardinality**. Hence the name:

> **LogLog = log₂(log₂(n))**, where *n* is the count of distinct items.

This insight led to compact, memory-efficient cardinality estimation.

In 2007, Flajolet et al. pushed the idea further with **HyperLogLog**. The *“Hyper”* here refers to:

* Going beyond the original LogLog by **splitting the input into many small substreams** (using the first bits of the hash to partition into multiple registers).
* Applying a **harmonic mean** across the substream estimates.
* Introducing **empirical bias correction** using mathematical tuning.

Thus, **HyperLogLog** = An enhanced (hyper) version of **LogLog** with better **accuracy, scalability, and parallelism** — perfect for distributed systems and streaming applications.



### Where HyperLogLog Shows Up in Practice


* **Redshift and BigQuery** use internal variants of HLL for dashboard optimizations.
* **PostgreSQL** supports HLL via extensions (e.g., `postgreSQL-hll`) for approximate counts.
* **Apache Spark** provides the `approx_count_distinct()` API powered by HLL (or HLL++).
* **Streaming engines** like Apache Flink use HLL for real-time metrics and monitoring.


In essence, **HLL is the secret sauce** behind lightning-fast distinct metrics across most modern, large-scale data platforms.



# Demystifying HyperLogLog: Smarter Ways to Count in Big Data

In the world of big data, even something as simple as counting distinct items can get surprisingly expensive. Think trillions of rows, massive parallelism, distributed systems — and the humble `COUNT(DISTINCT x)` can become your query’s bottleneck.

This is where **HyperLogLog (HLL)** shines. It’s an ingenious probabilistic algorithm that offers **near-accurate cardinality estimates** with **minimal memory** — a perfect tradeoff for massive-scale analytics.

But before jumping to HyperLogLog, let’s step back to understand where it came from — and what makes it statistically brilliant.


## From LogLog to HyperLogLog

Counting distinct elements in a stream isn’t trivial when you can’t store them all. Traditional methods either:

* Require maintaining large hash sets (expensive in memory), or
* Provide poor approximations.

In 2003, the **LogLog algorithm** introduced a breakthrough: use the **position of the leftmost 1-bit** in hashed values to infer cardinality. It relied on the observation that:

> The probability that a randomly hashed number begins with *n* zeros is 1 in 2ⁿ.

This gave us a way to **estimate** the number of distinct elements by tracking the **maximum number of leading zeroes** seen across the stream.

HyperLogLog, introduced later in 2007, improved upon this idea with two main innovations:

1. **Partitioning the input space** using multiple registers (buckets).
2. **Bias correction** using harmonic mean and empirical correction tables.

This allowed HLL to scale with high precision (low relative standard error) and fixed, small memory.



## The Statistical Insight: Why It Works

The power of HLL lies in probabilistic thinking:

* Instead of **tracking all values**, we track **how extreme a random hash value can be**.
* The more unique values in the input, the more likely we are to see **hashes with more leading zeroes**.
* Think of it like: seeing a number like `000000000001101...` in a uniformly distributed hash implies many unique items must have been seen before it.

Mathematically:

* HLL divides the stream into **m registers**, each tracking the max rank (number of leading zeroes).
* The final cardinality estimate is derived as:

```
α * m² / ∑(2^-M[j])
```

Where:

* `M[j]` is the value in register `j`
* `α` is a bias correction constant

This estimation is extremely efficient — logarithmic in memory and robust for millions to billions of distinct values.



## A Simple Example

Let’s walk through this example:

```python
["apple", "banana", "banana", "orange", "kiwi"]
```

Assume a simple 8-bit hash output (for illustration only):

| Value  | Hash (binary) | Bucket (first 2 bits) | Remaining | Leading Zeros |
| ------ | ------------- | --------------------- | --------- | ------------- |
| apple  | `00101010`    | `00` (bucket 0)       | `101010`  | 0             |
| banana | `00011100`    | `00` (bucket 0)       | `011100`  | 1             |
| orange | `01000010`    | `01` (bucket 1)       | `000010`  | 5             |
| kiwi   | `00001011`    | `00` (bucket 0)       | `001011`  | 2             |

### Bucket Register Table:

| Bucket | Max Leading Zeros |
| ------ | ----------------- |
| 0      | 2                 |
| 1      | 5                 |
| 2      | 0 (empty)         |
| 3      | 0 (empty)         |

### Estimation

Let’s compute estimated cardinality:

* `M[0] = 2`, so `2^-2 = 0.25`
* `M[1] = 5`, so `2^-5 = 0.03125`
* Buckets 2 and 3 have `0` → `2^0 = 1` → `2^-0 = 1`

Total sum = `0.25 + 0.03125 + 1 + 1 = 2.28125`

Using the formula:

$$
E = 0.673 \cdot 4^2 / 2.28125 = 0.673 \cdot 16 / 2.28125 \approx 4.72
$$

True distinct count = 4
Estimate = \~4.72 → close, acceptable within error bounds


## HyperLogLog++ and Enhancements

Google later proposed ]**HyperLogLog++**](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/40671.pdf) with the following improvements:

* **Sparse representation** for small cardinalities.
* **Bias correction** using empirical tuning curves.
* **64-bit hashes** for better precision.
* Smarter thresholding between sparse and dense mode.

These tweaks made HLL++ the **go-to** algorithm for modern analytics engines.



## HyperLogLog in Spark: Approximate Distinct Count

Apache Spark, being designed for scale, incorporates HyperLogLog for its `approx_count_distinct` function via the **`approx_count_distinct(col, rsd)`** API.

* `rsd` is the relative standard deviation, default is 0.05 (≈ 5% error).
* Internally, Spark uses **HLL++** implemented in `DataSketches` or its own estimation logic (depending on the version).

This is especially useful when:

* You’re dealing with large joins or group-bys.
* You want a fast cardinality estimate in dashboards or sampling.
* You can trade a tiny margin of error for **huge performance wins**.

Example in Spark SQL:

```sql
SELECT approx_count_distinct(user_id) FROM events;
```

Versus:

```sql
SELECT COUNT(DISTINCT user_id) FROM events;
```

The former can be **orders of magnitude faster** — especially over partitioned datasets in cloud storage.

