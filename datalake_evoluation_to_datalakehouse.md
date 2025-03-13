# How Our Data Lake Evolved into a Data Lakehouse

## Understanding the Basics: Data Lake vs. Data Lakehouse

### What is a Data Lake?
A **Data Lake** is a centralized repository designed to store large volumes of structured, semi-structured, and unstructured data. It allows organizations to collect and retain raw data in its native format without the need for immediate transformation.

**Key Characteristics of a Data Lake:**
- **Storage**: Cost-effective, scalable storage (e.g., Amazon S3).
- **Data Types**: Supports diverse formats (CSV, JSON, Parquet, images, logs, etc.).
- **Schema**: Schema-on-read (applied when data is queried, not when stored).
- **Use Case**: Ideal for data exploration, machine learning, and big data analytics.

**Limitations of a Traditional Data Lake:**
- **No ACID Transactions**: Cannot perform updates and deletes. so, if you want to maintain merge tables or meet the privacy compliance like GDPR , we cannot do.


### What is a Data Lakehouse?
A **Data Lakehouse** combines the **scalability** of a Data Lake with the **transactional and governance** capabilities of a Data Warehouse. This hybrid architecture allows for advanced analytics, including updates and deletes, while maintaining open and flexible data storage.

**Key Characteristics of a Data Lakehouse:**
- **ACID Transactions**: Supports  updates, deletes, and inserts.This is supported through open table formats (e.g., Apache Iceberg, Hudi, Delta Lake)


## Our Journey: From Data Lake to Data Lakehouse

### Phase 1: Building the Data Lake
When we started, our architecture followed a classic Data Lake model:
- **Storage**: Amazon S3 for raw and processed data.
- **Catalog Management**: AWS Glue Data Catalog for table definitions.
- **Permissions & Governance**: AWS Lake Formation for access control.
- **File Formats**: Parquet, JSON, and CSV for data storage.

This setup worked well for our analytics use cases and allowed us to choose different query engines to process the data from the Datalake (AWS Athena, AWS EMR, AWS Glue ETL, AWS Redshift, AWS SPectrum, AWS Quicksight).

1. **Data Updates & Deletes**: We couldn't maintain merge-datasets i.e we dont have support to remove or modify records due to a lack of ACID support.
2. **Privacy Compliance**: Regulations like **GDPR** require us to delete or mask personal data on request.


### Phase 2: Introducing Open Table Formats
To address these challenges, we adopted **open table formats** that bridge the gap between lakes and warehouses:

1.  **Apache Iceberg** and **Apache Hudi**:  For scalable data handling with ACID transactions and schema evolution.
2.  **New AWS S3 Tables** : S3 with built-in Apache Iceberg suppor
3.  **Brought Processing data back into Datalake**

This transition enabled us to:
- Perform **upserts** (update or insert) and **deletes** directly on S3 data.
- Meet **privacy compliance** like GDPR

### Phase 3: Transforming into a Data Lakehouse
By integrating these advanced table formats and optimizing our architecture, our Data Lake evolved into a **Data Lakehouse**. Here’s how our setup looks today:

| Component                  | Data Lake (Original)         | Data Lakehouse (Current)           |
|----------------------------|------------------------------|------------------------------------|
| **Storage**                | Amazon S3                   | Amazon S3 (with Iceberg & Hudi)   |
| **File Formats**           | Parquet, JSON, CSV          | Iceberg, Hudi (ACID support) ,S3 Tables, Parquet, JSON, CSV     |
| **Transactions**           | None                        | Full ACID (Insert, Update, Delete)|
| **Query Engines**          | AWS Athena, AWS EMR, AWS Glue ETL, AWS Redshift, AWS SPectrum, AWS Quicksight | AWS Athena, AWS EMR, AWS Glue ETL, AWS Redshift, AWS SPectrum, AWS Quicksight)      |
| **Governance & Compliance**| Lake Formation              | Lake Formation+ GDPR privacy compliant|



