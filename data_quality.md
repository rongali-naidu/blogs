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
     * We usually verify this as part of Unit testing.For validating as part of DQ Monitoring, we need to query both Source data and Datalake/Datawarehouse data together. We could use DB Links, Data Sharing etc for querying them together.

     
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
| **Schema and Data Validation**     | Accurarcy, Completeness, Validity, Uniqueness, Integrity | Enforces structure and formatting rules using:<br>• **Data Type Conformance  (Validity)** — Match types (e.g., int, string, timestamp)<br>• **Null Checks (Completeness)** — Required fields must not be null<br>• **Uniqueness / PK Checks(Uniqueness/Duplicates)** — Prevent duplicate key entries<br>• **Referential Integrity(Integrity)** — Validate FK relationships (e.g., `customer_id` exists)<br>• **Format Validation(Validity)** — Check specific patterns:<br> – Dates (`YYYY-MM-DD`, ISO 8601)<br> – Currency (`$99.99`, `€1.000,00`)<br> – Time zones (`+05:30`, `UTC`)<br>• **Row Count**: Verify that the entire dataset intended for processing is present<br>• **Different Sources Coverage**: Confirm all expected data sources are contributing to the dataset. For example, when integrating data from multiple systems, it’s easy to overlook missing records from one of the system )<br>• **Schema evolutuon** : detecting new colums|
| **Business Rule Validation**  | Consistency    Accuracy, Validity                                 | Business/domain rules to catch cross-field inconsistencies:<br>• `order_amount > 0`<br>• `status IN ('SHIPPED', 'CANCELLED')`<br>• `start_date < end_date`<br>• Revenue drop shouldn't exceed 20%<br>Implemented via SQL, Python, dbt tests, or Great Expectations. <br>•metric values consistent across various granular datasets (daily, weekly, monthly)                                                                                                                                                                                                                                                                                                                                |
| **Data Profiling**            | Accuracy, Validity                            | Examines column-level stats:<br>• min, max, avg, std dev<br>• null counts, distinct values, cardinality<br>• New/unseen value combinations. Often the **first step** in understanding unknown data.                                                                                                                                                                                                                                                                                                                                   |
| **Anomaly Detection**         | Accuracy, Trend Stability                     | Detects deviations using statistical models or ML:<br>• Volume spikes/drops<br>• Distribution drift (relevant for ML Models monitoring and it doesnt come strictly Anomaly Detection <br><br>Complements rule-based checks with adaptive insights.                                                                                                                                                                                                                                                                                                                                                                                                      |
                                                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                                                     
## Data Quality Approaches at different Data Life Cycles:
- **Data Ingestion** – Schema and Data Validation.
    * Schema and data validation can be performed before loading into raw datasets in the data lake/staging layer or later during processing of raw data.
    * While pipeline monitoring may catch failures reactively, explicit DQ checks at ingestion help proactively prevent partial or corrupted loads that could mislead downstream consumers. They also support automated fallback actions like quarantine or alerting.
    * Catching issues late often requires costly reruns and cross-team coordination. Validating early acts as a protective buffer, minimizing downstream impact and improving overall data reliability.
    * Note : If your data ingestion tool (ETL, Orchestation tools) and data lake/data warehouse enforce schema validation, Explicity DQ Checks through tools like [Deequ](https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/)  before data ingestion might give limited value for basic checks. However, it shines when tackling more complex validations—like detecting anomalies, data distribution drift (required for ML models monitoring), schema evolutuon (detecting new colums), ensuring data completeness, and validating business rules beyond standard schema constraints. It does come with trade-offs—increased compute costs and potential delays in processing. Therefore, the decision to implement Deequ (or similar tools) should weigh the benefits of deeper, proactive checks against the associated computational and operational overhead.

- **During Data Processing** – Data availability.
- **Post Data Processing** – Ensuring the processed data meets **business expectations** (or **data consumers**). We use mix of Business Rule Validation, Data Profiling and Anomaly Detection DQ Approaches





## Does it mean we wont get any Data Quality issues reported?


While data engineers can implement technical checks, it’s the **users** interacting with reports and analyses who often identify **gaps, inconsistencies in business logic, or missing data points**. Their validation is shaped by how they interpret and apply the data to solve business problems.

Common areas where user-driven validation surfaces issues include:

- **Logic behind the KPIs (metrics) is outdated or inconsistent**
- **Unexpected KPI Values**: Users are the first to spot anomalies—such as a sudden drop in transaction volume for a particula product category. This could be addressed to some extent using Data Profiling, Statistical and Data Anomaly based techniques
- **Edge Case Detection**: Users working on **ad hoc** analyses are more likely to surface outliers or gaps that automated checks miss—like missing regions in a geographic report or discrepancies across time zones.



## Can we measure Data quality?

Even though we describe Data Quality among various dimensions, there is no accepted defintion for measuring the data quality. Here is an alternative to quantify the Data Quality metric

### Data Quality Definitions tables 

Table 1: `dq_rules`
| **Rule ID** | **Table Name**     | **DQ Dimension** | **Rule Description / Logic**                                                  | **Threshold (Optional)** | **Weight** |
| ----------- | ------------------ | ---------------- | ----------------------------------------------------------------------------- | ------------------------ | ---------- |
| `R001`      | `customer`         | Completeness     | `customer_id IS NOT NULL`                                                     | `< 1% nulls`             | 1         |
| `R002`      | `customer`           | Accuracy         | `order_total >= 0`                                                            | `= 0 violations`         | 1          |
| `R003`      | `customer`         | Uniqueness       | `customer_id` must be unique                                                  | `= 0 duplicates`         | 10         |
| `R004`      | `customer`           | Validity         | `email LIKE '%@%.%'`                                                          | `< 0.5% invalid`         | 1          |
| `R005`      | `customer`           | Consistency      | `status = 'SHIPPED'` must have `shipped_date IS NOT NULL`                     | `= 0 violations`         | 1          |
| `R006`      | `customer`           | Integrity        | `customer_id` must exist in `customer` table                                  | `= 0 FK failures`        | 1          |
| `R007`      | `customer`       | Timeliness / SLA | `MAX(updated_at)` should be within last 2 hours                               | `< 2 hrs delay`          | 2          |


Table 2: `dq_rule_runs`

This is your **runtime results** table, logging the outcome of each rule per run.
|**rule_id** | **dq_run_date** | **dq_status** | **violation_count** | **total_records\_checked** | **violation_percent** | 
|------------ | ------------ | ---------- | -------------------- | --------------------------- | ---------------------- | 
|`R001`       | `2024-06-01` | SUCCESS    | 12                   | 10000                       | 0.12%                  |
|`R002`       | `2024-06-01` | FAILED     | 36                   | 10000                       | 0.36%                  | 




DQ Score Logic (Example)

```sql
WITH dq_runs AS (
  SELECT rule_id, dq_status
  FROM dq_results
  WHERE 
	dq_rule_runs='2024-06-01'
)
SELECT
	q_rules.table_name,
    ROUND(SUM(CASE WHEN dq_status='SUCCESS' THEN weight ELSE 0 END) * 100.0 / SUM(weight), 2) AS dq_score
FROM dq_runs
	join dq_rules on dq_runs.rule_id= dq_rules.rule_id
GROUP BY 
	dq_rules.table_name
```


### **Metadata and Data Lineage: Better quality documentation 
--Note : This section should be moved to Data Governance

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
