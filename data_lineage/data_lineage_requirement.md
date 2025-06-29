## Introduction

This is a follow up to [Data Lineage in the AI Era: Understanding Its Essence Through Multiple Lenses](https://medium.com/@rongalinaidu/data-lineage-in-the-ai-era-understanding-its-essence-through-multiple-perspectives-5d6570649b35), which gives overview of end to end data flow (data lineage) and why end-to-end Lineage is relevant.
 I often encountered fragmented approaches to data lineage—each tool or layer (ETL, OLTP, BI, Feature Store) maintaining its own understanding of the data flow. But none of them captured the full picture from source to dashboard. 
 Here, wanted to explore any implementations which covers end to end data lineage. In the process came across Meta blogs on this subject. Based on Meta's engineering blogs, this post summarizes their key implementation strategies and extends that into a generalized end-to-end data lineage and metadata tracking system—with privacy awareness built-in




## Summary of the Data Lineage Implementation at Meta

1. **Static Analysis**: Examining stored queries, configurations, and codebases (e.g., SQL, config files) to infer how data is structured and flows across systems before runtime.

2. **Runtime Signals**: Injecting lightweight instrumentation ("probes") into applications and data tools to emit lineage events during actual execution, capturing dynamic data flows.

3. **Unified Lineage Graph**: Combining static and runtime insights to create a full graph showing how data moves, transforms, and is accessed across systems.

4. **Privacy Awareness**: Using metadata tags to track and reason about sensitive data (like personal information) and ensure it’s handled appropriately.

5. **Automation**: Minimizing manual metadata management by integrating directly into developer workflows and infrastructure.

More details here:
🔗 [Meta's January 2025 blog:how-meta-discovers-data-flows-via-lineage-at-scale](https://engineering.fb.com/2025/01/22/security/how-meta-discovers-data-flows-via-lineage-at-scale/)
🔗 [Meta's April 2025 blog: how-meta-understands-data-at-scale](https://engineering.fb.com/2025/04/28/security/how-meta-understands-data-at-scale/)







## Title: **End-to-End Data Lineage and Privacy-Aware Metadata System**


### 1. Objective

Build a system that provides **complete data lineage visibility** — from the moment data is **created or transformed in application code or OLTP databases**, through **pipelines, features, reports, and dashboards**. This includes:

* Lineage capture across Software Application code, OLTP DBs, NoSQL DB, ETL Tools, Data Lakes ,  Datawarehouse DB, BI Reporting tools, Feature Stores
* Privacy, PII tagging, and schema annotation
* Support for multi-language (Java, Python, Node.js) and multi-database environments

---

### 2. Key Components (Lineage Signal Sources)

| Component                              | Description                                                |
| -------------------------------------- | ---------------------------------------------------------- |
| **Application Code Analysis**          | Analyze Java/Python/Node code that generates/modifies data |
| **SQLs and Stored Procedures in OLTP**          | Parse and trace lineage inside RDBMS SQLs and stored procedures     |
| **Static SQL Parsing (ETL)**           | Redshift SQL, etc.                              |
| **ETL Code Analysis**           | Python, Pyspark, Spark code, PL/SQL etc                             |
| **BI Tool Metadata**                   | QuickSight, Power BI, , Tableau                            |
| **Feature Store Definitions**          | Python code  , Jupyter Notebooks                           |
| **Runtime Instrumentation (Optional)** | Probes in APIs/services to emit lineage at runtime         |
| **Manual Tagging + Schema Decorators** | Add metadata for privacy, critical fields                  |

---

### 3. Architecture Overview

```text
[App Code + OLTP SPs]
         ↓
[Static Analyzers, Parsers, Instrumentation]
         ↓
[Lineage Events Stream / API]
         ↓
[Metadata Catalog (DataHub / OpenMetadata)]
         ↓
[UI for Lineage, Impact Analysis, PII Sensitivity]
```



### 4. Data Sources & Analysis Targets

| Layer                        | Targets & Examples                                                    |
| ---------------------------- | --------------------------------------------------------------------- |
| **Application Code**         | Java, Python, Node services (e.g., orderService writes to PostgreSQL) |
| **Stored Procedures (OLTP)** | PostgreSQL, SQL Server, Oracle PL/SQL                                 |
| **NoSQL Datastores**         | MongoDB, DynamoDB — read/write operations                             |
| **Event Systems**            | Kafka, SNS/SQS — track messages that carry data                       |
| **ETL Pipelines**            | Glue, Airflow, Spark                                                  |
| **Feature Store Code**       | Python                                                    |
| **BI/Reporting**             | QuickSight, Power BI  ,Tableau                                       |
| **Data Catalogs**            | Hive, Glue Catalog                                                    |



### 5. Functional Requirements

#### 5.1 Application + OLTP Lineage

* [ ] Parse code from application repositories (Java, Python, etc.) to identify:

  * Specific Code points (Functions/Package names)  has data generation steps
  * SQL queries (via JDBC/ORM/Raw SQL)
  * NoSQL insert/update/delete operations
  * API endpoints writing to DBs
* [ ] Parse stored procedures for control-flow + SQL dependencies
* [ ] Link functions/methods to table/column-level writes
* [ ] Capture inferred lineage even when queries are built dynamically (partial support)

#### 5.2 NoSQL & Event Lineage

* [ ] Capture topic/message schema (e.g., Kafka schema registry, SQS payloads)
* [ ] Map producers → consumers via message IDs or metadata
* [ ] Parse NoSQL mutations (write logs, audit tables)

#### 5.3 Lineage Graph Construction

* [ ] Stitch table/column-level lineage across application → OLTP → pipeline → warehouse → dashboard
* [ ] Link lineage to PII metadata, classification, and sensitivity
* [ ] Visualize all paths (e.g., `customer_email` from `signup-service.py` → Redshift → Tableau dashboard)

#### 5.4 Privacy & Data Sensitivity

* [ ] Support tagging of sensitive fields at schema or ingestion level
* [ ] Propagate tags across the lineage graph (data inheritance)
* [ ] Allow querying: "Where does PII flow downstream?" or "Who modifies sensitive data?"

#### 5.5 Developer Experience

* [ ] Git hooks / CLI to annotate fields or export lineage
* [ ] IDE plugins to show data flow impact
* [ ] APIs for injecting lineage via CI/CD or app instrumentation

---

### 6. Tooling Options for Extended Coverage

| Need                          | Tool/Approach                                                                    |
| ----------------------------- | -------------------------------------------------------------------------------- |
| Static SQL parsing (OLTP/ETL) | SQLGlot, dbt manifest, Apache Calcite                                            |
| Java/Python code analysis     | **Babeltrace**, **AST parsers**, **OpenRewrite**, **jSparrow**, custom AST tools |
| Runtime instrumentation       | OpenLineage SDK, custom HTTP middleware                                          |
| Stored procedure parsing      | ANTLR grammars, SQLFluff, Oracle tools                                           |
| NoSQL audit tracking          | Change Streams (MongoDB), DynamoDB Streams                                       |
| Lineage catalog & UI          | DataHub, OpenMetadata, Atlan                                                     |
| Privacy Tagging               | dbt `meta`, custom YAML, tag propagation                                         |



### 7. Phased Implementation Plan

| Phase                            | Deliverable                             | Duration |
| -------------------------------- | --------------------------------------- | -------- |
| Phase 1: SQL + ETL Lineage       | dbt, Glue, Redshift lineage             | 1–2 mo   |
| Phase 2: BI Tool Integration     | QuickSight/Tableau lineage              | 1 mo     |
| Phase 3: Application Code + OLTP | Code parsers + stored procedure lineage | 2–3 mo   |
| Phase 4: NoSQL + Event Streams   | Audit log processing, Kafka lineage     | 1–2 mo   |
| Phase 5: Privacy-Aware Features  | Sensitivity tagging, alerting           | 1 mo     |



### Example Use Case Tracked

> `user_email` written by `userService.java` (insert into PostgreSQL) →
> ETL job → S3 parquet file → Redshift `user_dim` → dashboard filter in QuickSight


