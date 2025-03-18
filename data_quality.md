## What Years of Working with Data Have Taught Me About Data Quality

**Data quality** is a term that gets thrown around a lot, but its meaning varies depending on who you ask.

For a **data engineer**, it’s about ensuring pipelines run accurately and on time, datasets have the complete data at the required grain, and all necessary attributes have expected values.

For **data consumers**—such as **data analysts, business analysts, data scientists, and business intelligence engineers**—it means working with reliable and timely data to generate meaningful insights.

For **business stakeholders**, it’s the ability to trust data-driven decisions.

After working with data for years across multiple platforms and industries, I’ve realized that data quality covers the following major categories:

- **Data Accuracy** – Simply put whatever data we loaded is correct in all aspects. I usually include **Data Validity** (agreeing with the schema), **Data Integrity** (maintaining relationships across fact and dimension tables), and **Data Uniqueness** (ensuring no duplicates) under Data Accuracy for simplicity.
- **Data Completeness** – Data is complete. This means we capture all data from source systems, load the entire dataset without omissions, and include all columns required by consumers.
- **Data Consistency with Other Systems** – Data must align with and reflect the information in source systems and other integrated platforms.
- **Data Availability as per SLA** – Data is timely and delivered according to the agreed Service Level Agreements (SLAs). Some use cases require real-time delivery, while others may allow for daily batch updates.

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
