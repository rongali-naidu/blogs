# **Defining Data Quality: The Foundation of Modern Data Architecture**

## **Introduction**

This article is a follow-up to my previous post [Data Quality Frameworks: The Modern Data Architecture Replacement for Traditional Database Schema and Data Enforcement](https://medium.com/towards-data-engineering/data-quality-frameworks-the-modern-data-architecture-replacement-for-traditional-database-schema-1e4ef72f2c4b ).

In that piece, I explored how Data Quality (DQ) frameworks have evolved to fill the enforcement gap left by traditional database schemas in today’s distributed data environments.

Here, I’ll take a step back to answer a more fundamental question: What exactly is Data Quality, and can we measure it consistently enough to remove the subjectivity around the term?

In modern data-driven enterprises, **Data Quality (DQ)** has taken on a new level of significance. As organizations shift from monolithic databases to distributed **data lakes, warehouses, and data mesh architectures**, the old assumptions about schema enforcement and centralized validation no longer hold true.

This transformation has elevated **Data Quality Frameworks** into the backbone of **trust, governance, and reliability** in modern data ecosystems. Yet, despite its importance, the term *Data Quality* is often used loosely — its meaning and scope tend to vary depending on who you ask.

In this article, we’ll define what *Data Quality* truly means, explore its **core dimensions**, and discuss **how to enforce and measure it effectively** across modern data platforms.

---

## **What Is Data Quality?**

At its core, **Data Quality** reflects how well data meets the needs of its consumers — whether for analytics, machine learning, or operational decisions.
Good data is **accurate, complete, consistent, unique, valid, timely, and available**. Poor data quality leads to unreliable insights, broken pipelines, and loss of business trust.

---

## **The Dimensions of Data Quality**

### **1. Accuracy**

Accuracy measures how closely data reflects the real-world value or event it represents — simply put, *is the data correct?*

* **At Source Systems:**
  Input validation and business logic ensure correct values are captured (e.g., correct customer address, valid order quantity).
* **At Data Lake/Warehouse:**
  Data should match what the source emitted, with no corruption or transformation error during ingestion.
  Accuracy validation often involves cross-checking source and target via DB links or data-sharing mechanisms.

---

### **2. Completeness**

Completeness ensures that all expected data is present — no missing rows, fields, or files.

* **At Source Systems:**
  Required fields enforced, all expected records submitted.
* **At Data Lake/Warehouse:**
  Checks for missing partitions, truncated rows, or nulls in key columns.
  Record counts between source and raw zone should match.

---

### **3. Consistency**

Consistency ensures no contradictions exist within or across datasets.

* **At Source Systems:**
  Referential integrity and standardized rules (e.g., valid order-customer links).
* **At Data Lake/Warehouse:**
  Consistent values and metrics across reports and systems.
  Includes timezone, currency, and data type standardization during ingestion.

---

### **4. Uniqueness**

Uniqueness ensures no duplicate records or keys exist where they shouldn’t.

* **At Source Systems:**
  Unique key constraints and validation.
* **At Data Lake/Warehouse:**
  Deduplication, identity resolution, and master data management for dimensions.

---

### **5. Integrity**

Integrity maintains correct relationships between entities (facts and dimensions).

* **At Source Systems:**
  Referential integrity enforcement.
* **At Data Lake/Warehouse:**
  Validations to ensure fact-to-dimension relationships remain intact.

---

### **6. Validity**

Validity ensures data values conform to defined formats, domains, and business rules.

Examples:

* Email format: `user@example.com`

* Date format: `YYYY-MM-DD`

* Logical constraints: `start_date < end_date`

* **At Source Systems:**
  Front-end regex checks, API schema validation, database constraints.

* **At Data Lake/Warehouse:**
  Validations ideally done upstream; invalid patterns downstream indicate weak source hygiene.

---

### **7. Availability**

Availability ensures data is delivered **on time** and **per SLA**.
Even with correct values, stale data breaks trust.

* **At Source Systems:**
  Data published according to SLA.
* **At Data Lake/Warehouse:**
  Monitors ingestion timeliness and pipeline latency affecting data freshness.

---

### **8. Distribution (Data Drift)**

For data science workloads, data distribution stability is crucial.
Changes in distribution (data drift) can degrade model performance, even if data is accurate and complete.

---

## **Where to Apply Data Quality Checks**

### **Shift-Left Approach**

Borrowed from DevOps, the **shift-left principle** means moving DQ validation as close to the data source as possible:

* **At the Source:** Validate before ingestion.
* **During Ingestion:** Apply checks within pipelines to catch bad data early.
* **Post-Ingestion:** Reconciliation and monitoring for consistency and freshness.

### **Layered Validation**

* **Source Systems:** First line of defense — cheapest to fix errors.
* **Data Lake/Warehouse:** Final defense — monitor quality at scale.

Best practice: build **redundant validation layers** at every system handoff.

---

## **Testing, Monitoring, and Cleaning**

* **Unit Testing:** Ensures known data assumptions hold during development.
* **Data Cleaning/Transformation:** Fixes known quality issues.
* **Data Quality Monitoring:** Detects unexpected issues as data evolves — your “dynamic safety net.”

---

## **Who Owns Data Quality?**

* **Pipeline Engineers:** Must embed DQ checks within pipelines — this is mandatory.
* **Unit Tests:** DQ validations are part of integration and deployment testing.
* **Dedicated DQ Roles:** Optional; similar to QA in software. Needed only when business or regulatory contexts demand it.

---

## **Do We Need a Data Quality Platform or Data Stewards?**

Whether to invest in a centralized **DQ platform** or **Data Steward roles** depends on scale and governance maturity.

* **Engineering-Led Enforcement:**
  Most DQ dimensions can be enforced programmatically:

  * Timeliness via pipeline orchestration
  * Completeness via schema/row checks
  * Uniqueness via constraints
  * Validity & consistency via embedded business rules

* **Dedicated DQ Platform:**
  Justified only when enterprise-wide dashboards, compliance reporting, or cross-domain oversight are required.

In most modern organizations, **DQ platforms are optional — not mandatory.**

---

## **Data Stewardship in Context**

Like automated testing replaced manual QA in software engineering, data engineering can embed quality directly in pipelines.
Dedicated **Data Stewards** become necessary only for:

* Non-automatable business rule validation
* Regulatory reporting
* Enterprise-wide visibility

Otherwise, stewardship should be **distributed** among domain data product teams.

---

## **Data Quality and the Medallion Architecture**

Data quality overlaps with concepts like **Bronze/Silver/Gold layers** and **Data Usability Certifications**:

* **Data Quality Certification:** Measures how *accurate, complete, and reliable* data is.
  *Focus:* “How good is the data?”
* **Data Usability Certification:** Measures how *modeled, standardized, and analysis-ready* data is.
  *Focus:* “How ready is the data for use?”

These certifications help consumers understand dataset readiness across the data lifecycle.

---

## **Common Data Quality Approaches**

| **Category**                 | **Targets**                                             | **Description**                                                                                                                            |
| ---------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Schema & Data Validation** | Accuracy, Completeness, Validity, Uniqueness, Integrity | Checks for schema conformance, data type, nulls, duplicates, and referential integrity.                                                    |
| **Business Rule Validation** | Consistency, Accuracy, Validity                         | Applies domain logic and cross-field rules (e.g., `start_date < end_date`). Implemented via SQL, Python, dbt tests, or Great Expectations. |
| **Data Profiling**           | Accuracy, Validity                                      | Summarizes data to detect anomalies (min, max, avg, nulls, distinct values).                                                               |
| **Anomaly Detection**        | Accuracy, Trend Stability                               | Detects unexpected deviations (spikes, drops, or drifts) using statistical or ML models.                                                   |

---


## **Measuring Data Quality**

While discussions around **Data Quality (DQ)** often focus on frameworks and principles, the real challenge lies in **quantifying** it. How do we objectively measure whether a dataset is "good enough"?

A practical approach is to assign a **Data Quality Score** — sometimes also referred to as a **Data Quality Certification** — to each dataset. This score acts as a measurable indicator of the dataset’s overall health and reliability.

The concept is simple but powerful:

1. **Define Validation Rules:**
   For each dataset, establish a set of DQ rules or checks that cover key dimensions such as **accuracy**, **completeness**, **consistency**, and **validity**.

2. **Evaluate Rule Outcomes:**
   Each rule produces a binary result — **pass** or **fail** — based on whether the dataset meets the expected standard.

3. **Calculate the DQ Score:**
   The **Data Quality Score** is the percentage of rules passed:

   **DQ Score** = `(Number of Passed Rules / Total Number of Rules) × 100`

   This produces a clear, interpretable metric that can be tracked over time.

5. **Monitor Trends:**
   By monitoring DQ scores periodically, teams can identify degradation early, track improvements, and prioritize datasets that need remediation.

This rule-based scoring approach provides a **consistent, scalable, and actionable** way to measure Data Quality across diverse data domains. It transforms DQ from a subjective discussion into a quantifiable engineering metric — something that can be **automated, monitored, and improved continuously.**


