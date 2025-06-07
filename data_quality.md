## What Years of Working with Data Have Taught Me About Data Quality

**Data quality** is a term that gets thrown around a lot, but its meaning varies depending on who you ask.

For a **data engineer**, it’s about ensuring pipelines run accurately and on time, datasets have the complete data at the required grain, and all necessary attributes have expected values.

For **data consumers**—such as **data analysts, business analysts, data scientists, and business intelligence engineers**—it means working with reliable and timely data to generate meaningful insights.

For **business stakeholders**, it’s the ability to trust data-driven decisions.

After working with data for years across multiple platforms and industries, I’ve realized that data quality covers the following major categories:

- **Data Accuracy** – Accuracy refers to how closely the data reflects the real-world value or event it represents . Simply put whatever data we loaded is correct in all aspects.
   * At Source Systems: Application logic (e.g., input validation, dropdowns instead of free text) ensures correct values are captured.
     * Examples: Correct customer address, accurate order quantity, valid timestamps.
   * At Data Lake / Warehouse: Accuracy means matching what was received from the source, with no corruption or transformation error during ingestion or processing. accuracy is usually interpreted as "did we receive what the source emitted?"

     
- **Data Completeness** – Completeness ensures all expected data is present — all rows, fields, and values.
   * At Source Systems:
     * Required fields enforced (e.g., no NULLs in customer_id)
     * All records expected in a transaction or API call are submitted.
   * At Data Lake / Warehouse:
     * Checks for missing files, truncated rows, or empty partitions.
     * Column-level null checks in staging and curated zones.
     * Record count matching between source and raw zone.
     * Ensures Data is available for all columns required by the dowsntream consumers of the dataset (Analytics, Data Scince)
    
- **Data Consistency** – Consistency ensures no contradictions exist within or across datasets and systems.
   * At Source Systems:
     * Referential integrity between entities (e.g., orders link to valid customers). [Note: this is an overlap with the Data Integrity dimension of Data quality]
     * Consistent rules (e.g., status enums are standardized).
   * At Data Lake / Warehouse:
     * Cross-source comparisons (e.g., same product name wherever its is present).
     * Consistency of metrics across reports (e.g., revenue totals aligns base datasets, agregated datasets).
     * Timezone handling, Currency Handling, and data type standardization during ingestion.
    
- **Data Uniqueness** – No duplicate records or keys where uniqueness is required. This is required for master data management. Master data is usually a dimension in Data Lake/Datawarehouse.
   * At Source Systems: Unique constraints and validation on keys.
   * At Data Lake / Warehouse:Deduplication, record matching, identity resolution
 
- **Data Integrity** – Maintaining correct relationships across data entities (foreign keys, dimensional relationships).
   * At Source Systems: Referential integrity enforcement.
   * At Data Lake / Warehouse:  Validations ensuring integrity across facts and dimension tables.
       
 - **Data Validity** – Ensures that data values adhere to all rules that define whether the data is valid, including correct syntax (format), allowed values, logical consistency, and business constraints. Data validity overlaps with Data Accuracy and Data Consistency since invalid data cannot be accurate or consisten
   
   * Format validation is a key part of validity, focusing on syntax and structure. For example:
     * Email addresses follow a valid pattern
     * Dates are in YYYY-MM-DD format
     * Numeric fields contain only digits

   * Other validity checks include:
     * Values fall within allowed ranges or categories (e.g., status codes)
     * Dates are logical (e.g., no February 30)
     * Referential integrity between related fields

   * At Source Systems:
     * Front-end validations (e.g., email regex)
     * API schema contracts
     * Database constraints (e.g., date formats, non-numeric checks)
   * At Data Lake / Warehouse:
     * It’s ideal to validate format upstream. Validating emails or phone numbers downstream often signals poor data hygiene at the source      

- **Data Availability as per SLA  (Service Level Agreement)** – Data is timely and delivered according to the agreed Service Level Agreements (SLAs). Some use cases require real-time delivery, while others may allow for daily batch updates. Job failures, pipeline latency etc comes under operational issues but at broader level they could come under DQ issue since these operational issues affect the data fresheness.
   * At Source Systems:
     * Data is published as per SLA contracts
    * At Data Lake / Warehouse:
     * Data is ingested and processed and made available in the curated datasets used by the downstream consumers as per SLA

- **Why Data Quality ar both Layers Matter**
   * Source Systems:The first line of defense. They’re closest to the business process and user input. Errors caught here are cheapest to fix.
   * Data Lake / Warehouse:The final line of defense. They catch issues missed upstream and monitor quality at scale across systems.
   * Best practice: Don’t rely solely on downstream checks. Build in **layered, redundant validation at every handoff** — especially between systems.

- **Summary Table: Where to Validate Each Quality Dimension**

| DQ Dimension           | At Source System                                               | At Data Lake / Warehouse                                         |
|------------------------|----------------------------------------------------------------|------------------------------------------------------------------|
| **Accuracy**           | Ensure correctness of values entered                           | Verify values match source input (no corruption)                 |
| **Completeness**       | Enforce required fields; no skipped records                    | Row, column, and file-level completeness checks                  |
| **Consistency**        | Consistent business rules, no referential breaks               | Cross-system reconciliation and schema alignment                 |
| **Uniqueness**         | Unique constraints and key validation                          | Deduplication, record matching, identity resolution              |
| **Integrity**          | Enforce referential integrity (e.g., foreign key constraints)  | Validate relationships across facts and dimensions               |
| **Validity**           | Validate formats, ranges, allowed values, business constraints | Check data conforms to schema, enums, and domain logic           |
| **Availability (SLA)** | Ensure data is published or emitted on time                    | Ensure ingestion and delivery to downstream systems meets SLA    |

## Unit Testing vs DQ Monitoring: Different Purposes

While both unit testing and DQ monitoring focus on data reliability, they serve **complementary but distinct purposes** across the data pipeline lifecycle.

| Aspect       | Unit Testing of Datasets/Pipelines                                                                              | Data Quality (DQ) Monitoring                                                                                                              |
| ------------ | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **What**     | Schema checks, column types, business rules, edge cases, join behavior, partitioning logic                      | Monitors real-time/batch data for accuracy, completeness, uniqueness, freshness, etc.  
| **When**     | During development or before deployment; also during code changes                                               | Continuously in production                                                                                                                |
| **Why**      | Catch logic/schema issues early and validate transformation logic                                               | Detect unexpected issues in live data (e.g., pipeline failures, data drift, upstream changes)                                             |
| **Where**    | Manual, In CI/CD workflows                                                     | In production environments, often integrated with alerting and monitoring tools                                                           |
| **Examples** | Validate derived column logic <br> - Row Count validation between source and target of the pipeline | - Detect nulls in historically populated columns <br> - Warn on duplicate IDs appearing |



## DQ Approaches 

* RDBMS Based Datawarehouse supports some of these DQ Approaches
* ETL/Orchestration tools supports some of these DQ Approaches
* Additional DQ Speific frameworks supports of these DQ Approaches
* Statistical / ML Models based frameworks supports of these DQ Approaches


| **Category**                  | **Targets**                                   | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Schema (Structural) Validation**     | Completeness, Validity, Uniqueness, Integrity | Enforces structure and formatting rules using:<br>• **Data Type Conformance  (Validity)** — Match types (e.g., int, string, timestamp)<br>• **Null Checks (Completeness)** — Required fields must not be null<br>• **Uniqueness / PK Checks(Uniqueness/Duplicates)** — Prevent duplicate key entries<br>• **Referential Integrity(Integrity)** — Validate FK relationships (e.g., `customer_id` exists)<br>• **Format Validation(Validity)** — Check specific patterns:<br> – Dates (`YYYY-MM-DD`, ISO 8601)<br> – Currency (`$99.99`, `€1.000,00`)<br> – Time zones (`+05:30`, `UTC`) |
| **Business Rule Validation**  | Consistency                                   | Business/domain rules to catch cross-field inconsistencies:<br>• `order_amount > 0`<br>• `status IN ('SHIPPED', 'CANCELLED')`<br>• `start_date < end_date`<br>• Revenue drop shouldn't exceed 20%<br>Implemented via SQL, Python, dbt tests, or Great Expectations. <br>•metric values consistent across various granular datasets (daily, weekly, monthly)                                                                                                                                                                                                                                                                                                                                |
| **Data Profiling**            | Accuracy, Validity                            | Examines column-level stats:<br>• min, max, avg, std dev<br>• null counts, distinct values, cardinality<br>• frequency distributions<br>Used to detect type mismatches, misclassified fields, or outliers. Often the **first step** in understanding unknown data.                                                                                                                                                                                                                                                                                                                                   |
| **Anomaly Detection**         | Accuracy, Trend Stability                     | Detects deviations using statistical models or ML:<br>• Volume spikes/drops<br>• Distribution drift<br>• New/unseen value combinations<br>Complements rule-based checks with adaptive insights.                                                                                                                                                                                                                                                                                                                                                                                                      |
                                                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                                                                                                                                                                                 
My approach to data quality revolves around three critical phases:

1. **During Data Ingestion** – Ensuring data integrity and schema compatibility when ingesting data. Approaches vary—some teams validate data before loading it into the **data lake/staging layer**, while others load raw data first and perform checks afterward.
2. **During Data Processing** – Monitoring data accuracy, consistency, and ensuring pipelines meet **SLAs**.
3. **After Data Processing – Validation Through the User's Lens** – Ensuring the processed data meets **business expectations** and addressing gaps uncovered by **data consumers**.

Each phase requires a unique set of checks and processes to ensure that the data remains **accurate**, **complete**, and **timely**. Let’s explore these phases in depth.


---

## 1. Data Quality Before Ingesting Data

Before data enters a data lake or warehouse, validating its structure and completeness prevents cascading errors downstream. Key checks at this stage include:

### **Schema Compatibility**
Ensuring schema alignment prevents ingestion failures and data corruption.

- **Data Types**: Validate that each column’s data type (e.g., `INT`, `STRING`, `TIMESTAMP`) matches the expected schema, including null constraints and allowed values.
- **Format Consistency**: Enforce standardized formats for timestamps (e.g., **ISO 8601**, UTC vs. local), currency, and other structured data—especially when merging data from multiple systems.

---

### **Data Completeness**

- **Row Count**: Verify that the entire dataset intended for processing is present.
- **Mandatory Columns**: Ensure all required columns are both present and populated.
- **Source Coverage**: Confirm all expected data sources are contributing to the dataset. For example, when integrating data from multiple systems, it’s easy to overlook missing records if pipeline scheduling is inconsistent.

---

### **How I Address This in My Work:**

- **Thorough Testing**: I rigorously test pipelines during development and validate data in production for a few days post-deployment. 
- **Alarms and Alerts**: Implement alarms for conditions like zero records or pipeline failures. For example, schema mismatches typically cause loading errors and alerts to the team.

---

### **Does This Mean I’ve Done Enough?**

- **Probably not…** Probably not… While these practices are a solid foundation, they don’t fully guarantee data quality as systems evolve and change. Alarms provide a safety net—but they are inherently reactive. Is there a better alternative? Tools like  [Deequ](https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/) offer a proactive approach by running comprehensive data quality checks before ingestion. But will this truly outperform reactive monitoring? If your ingestion tool and data lake already enforce schema validation, Deequ may add limited value for basic checks. However, it shines when tackling more complex validations—like detecting anomalies, data distribution drift, schema evolutuon (detecting new colums), ensuring data completeness, and validating business rules beyond standard schema constraints. It does come with trade-offs—increased compute costs and potential delays in processing. Therefore, the decision to implement Deequ (or similar tools) should weigh the benefits of deeper, proactive checks against the associated operational overhead.
---

## 2. Data Quality During Processing

Once data is ingested, maintaining its accuracy and ensuring timely delivery is paramount.

### **Data Accuracy**
Processed data should accurately reflect the source.

- **Record Count Accuracy**: Ensure all expected records are processed.
- **Column-Level Consistency**: Validate that derived values (e.g., calculated fields) are computed correctly.
- **Referential Integrity**: Enforce foreign key checks to maintain relationships between datasets.
- **Duplicate Detection** : Ensure uniqueness through defined primary keys. Some databases (e.g., **Redshift**) do not enforce uniqueness by default, requiring additional validation.For fact tables or event-based data lacking a clear primary key, enforce alternative unique identifiers..
---


### **Processing Timeliness (SLA Monitoring)**
Data is only valuable if it’s available when needed.

- **Latency Tracking**: Monitor and track how long data takes to move through the pipeline.
- **SLA Compliance**: Ensure data is delivered according to agreed-upon timelines.

---

### **How I Address This in My Work:**

- **Deduplication Strategies**: I design loading strategies to prevent duplicates and implement logic to identify and process the latest record when needed.
- **Pre-Deployment Testing**: Validate all scenarios—including edge cases—before deploying pipelines.
- **Alarms and Alerts**: Implement alarms for pipeline delays


### **Does This Mean I’ve Done Enough?**
- Most of this depends on the Orchestration tools , Query engine and other tech stack we work with and the features they support. 

## 3. After Data Processing – Validation Through the User's Lens

Once data is processed and made available, the final judgment of **data quality** often rests in the hands of **data consumers**—whether it’s a **data analyst**, **business analyst**, **data scientist**, or **business intelligence engineer**.
They expect all the aspects we validate during data ingestion and data processing i,e

- **Data Accuracry**
- **Data Completeness**
- **Data Consistency with other systems including source systems- **
- **Data Availability as per SLA defined**


While data engineers can implement technical checks, it’s the **users** interacting with reports and analyses who often identify **gaps, inconsistencies, or missing data points**. Their validation is shaped by how they interpret and apply the data to solve business problems.

Common areas where user-driven validation surfaces issues include:

- **Logic behind the KPIs (metrics) is outdated**
- **Unexpected KPI Values**: Users are the first to spot anomalies—such as a sudden drop in transaction volume for a particula product category.
- **Edge Case Detection**: Users working on **ad hoc** analyses are more likely to surface outliers or gaps that automated checks miss—like missing regions in a geographic report or discrepancies across time zones.

---

### **How I Address This in My Work:**

- **Running pre-defined rules on schedule**: The rules are based on our understanding of the data, learnings from previous data issues etc. Its an extension to running referential integrity check across the facts and dimensions post completing the batch load. Now that we have real-time data pipelines (aka Streaming pipeline), we need more tweaks to make it relevant.
- **Anomaly Reporting Pipelines**: Implementing anomaly reporting mechanisms helps surface hidden issues faster but needs more sophisticated anomaly detection logic to minimize false alarms. 


### **Does This Mean I’ve Done Enough?**

- There is always scope to improve… No matter how robust our checks are, users’ lenses will always reveal new data discrepancies that slip through. Proactive monitoring is crucial—but so is embracing user feedback as a core part of the data quality process. Establishing a feedback loop with data consumers and implementing cross-system reconciliation ensures a more holistic, user-centered approach to data quality.

---

## What’s Needed for Proactive Data Quality Management?

Modern data systems require continuous monitoring to catch errors in real time and ensure operational reliability. Accurate Metadata and Data Lineage plays critical role in reponding to any data quality issues 

### **Tracking Delayed Data Processing**
- **Monitor SLAs**: Implement real-time tracking for every stage of the pipeline.  
- **Alerting**: Trigger alerts when data falls behind schedule.  



### **Metadata and Data Lineage: Better quality documentation 
Every one values documentation but not committed to doing it regularly
Metdata helps to know more about what is stored in each dataset.
Data lineage maps how data moves and transforms through the system.

- **Source-to-Destination Mapping**: Understand how data travels across your infrastructure.  
- **Transformation Tracking**: Document how data is manipulated at each stage.  



---

| **Category** | **Tool/Framework** |
|--------------------------|-------------------------------------------|
| Data Pipeline Monitoring | ?? |
| Metadata (Catalog) | AWS Glue Data Catalog, OpenMetadata |
| Data Lineage Tracking | ?? |
| Data Validation | Deequ, ?? |
| Anomaly Detection | AWS CloudWatch, ?? |




---
