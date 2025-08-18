
# Did u notice this while creating S3-Tables through Athena CTAS?

AWS recently released **Athena support for creating S3 Tables directly via CTAS statements**. When I read the official [AWS Big Data blog on transforming your data to Amazon S3 Tables with Athena](https://aws.amazon.com/blogs/big-data/transform-your-data-to-amazon-s3-tables-with-amazon-athena/), one detail caught my eye:

```sql
format = 'parquet'
```

in the CTAS statement.

This raised a natural question: how does this relate to the familiar S3 Table DDL statement where we explicitly say:

```sql
TBLPROPERTIES ('table_type' = 'iceberg')
```

This post clarifies the connection, showing how Athena CTAS now produces S3 Tables, and how the Parquet format fits into the picture.

---

## Example 1: Creating a Table in S3 Tables with Iceberg

S3 Tables is a newer abstraction that integrates Amazon S3 storage with the **Apache Iceberg table format**. Iceberg adds a metadata and management layer on top of your raw Parquet/ORC files, enabling database-like capabilities over a data lake.

```sql
-- Example: Creating an S3 Table with Iceberg
CREATE TABLE `s3_tables_prod`.daily_sales (
    sale_date date,
    product_category string,
    sales_amount double
)
PARTITIONED BY (month(sale_date))
TBLPROPERTIES ('table_type' = 'iceberg');
```

### Key Points

* **`table_type = 'iceberg'`** → Registers the dataset as an Iceberg-managed S3 Table.
* **Partitioning** → Defined logically (e.g., `month(sale_date)`), Iceberg manages the folder structure and metadata internally.
* **Use Case** → Ideal for long-term, production-grade data lakes with ACID transactions, schema evolution, time travel, and query-friendly partitioning.

---

## Example 2: Creating a Table with CTAS (Parquet + S3 Table)

Athena CTAS (Create Table As Select) allows you to transform and persist query results. With the new integration, **CTAS can create an S3 Table automatically**, while the **physical data files are stored in Parquet**.

```sql
-- Example: Creating a CTAS table in Athena as an S3 Table
CREATE TABLE "s3tablescatalog/athena-ctas-s3table-demo"."reviews_namespace"."customer_reviews_s3table"
WITH (
    format = 'parquet',
    partitioning = ARRAY['day(review_date)']
) AS
SELECT *
FROM "awsdatacatalog"."reviewsdb"."customer_reviews"
WHERE review_year >= 2016;
```

### Key Points

* **`format = 'parquet'`** → The query results are serialized as Parquet files on S3.
* **S3 Table Registration** → Athena automatically registers the table as an **Iceberg-backed S3 Table** in the S3 Tables catalog; you don’t need to specify `TBLPROPERTIES`.
* **Partitioning** → Data is logically partitioned (e.g., by day) and managed via Iceberg metadata.
* **Use Case** → Enables ETL-style transformations that **also create production-ready, managed tables** in one step.

