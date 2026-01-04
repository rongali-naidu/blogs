# **The  Guide to Database Models : Logical, Physical, and Relational and NoSQL Models Explained**

Databases are the backbone of modern software systems. Choosing the right **data model**, understanding **logical vs physical modeling**, and considering **storage & access patterns** is critical for performance, scalability, and maintainability. This guide covers **all major database models**, including relational, NoSQL, dimensional, time-series, and hybrid systems, while integrating **normalization, denormalization, and one-wide-table (OWT) concepts** in the correct context.

---

## **What is a Data Model?**

A **data model** is a **conceptual framework** for organizing and representing data. It defines:

* **Entities** (things we store data about)
* **Attributes** (properties of entities)
* **Relationships** (how entities are connected)
* **Constraints & rules**

> Think of it as a **blueprint** for structuring your database before implementation.

---

## **Logical vs Physical Data Models**

### **Logical Data Model**

* Abstract representation of data structures and relationships.
* Focuses on **what data is stored** and **how it relates**, **independent of a specific DBMS**.
* Includes:

  * Entities and tables
  * Attributes and columns
  * Relationships (one-to-many, many-to-many)

**Key Points:**

* No database-specific types
* No storage considerations
* No indexes or partitions

### **Physical Data Model**

* Represents **implementation details** for a specific Database.
* Includes:

  * Data types
  * Constraints (PK, FK, unique, not null)
  * Partitions / shards / Sort Key / Distribution Key
  * Indexes
  * Storage layout (row vs columnar)
  * Column Level Encoding and Compression options

**Example (Relational Physical Model):**

```sql
CREATE TABLE Users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
PARTITION BY RANGE (created_at);
```

**Example (Cassandra Physical Model):**

```cql
CREATE TABLE user_orders (
    user_id UUID,
    order_id UUID,
    amount DECIMAL,
    order_date TIMESTAMP,
    PRIMARY KEY (user_id, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);
```



## ** Database Models Overview**

### **4.1 Key-Value Model**

* Stores data as **key → value pairs**
* Extremely fast, simple lookup by key

**Example:**

```text
"user:123" → {"name": "Alice", "age": 25}
"session:xyz" → "token_value_here"
```

**Use Cases:** Caching, session storage, application specific data etc
**Popular Databases:** Redis, MemoryDB, DynamoDB (KV), Memcached, Riak

---

### **4.2 Document Model**

* Stores **JSON/BSON documents**, flexible schema
* Queryable by document fields

**Example:**

```json
{
  "_id": "user123",
  "name": "Alice",
  "orders": [
    {"id":"order1","amount":50},
    {"id":"order2","amount":100}
  ]
}
```

**Use Cases:** CMS, e-commerce, application specific data
**Popular Databases:** MongoDB, DocumentDB, Couchbase, DynamoDB (document mode), Firebase Firestore

---

### **4.3 Wide-Column (Column-Family) Model**

* Stores **rows with variable columns**
* Optimized for **write-heavy workloads**

| Row Key | col:name | col:age | col:last_login |
| ------- | -------- | ------- | -------------- |
| user123 | Alice    | 25      | 2026-01-01     |
| user124 | Bob      | 30      | 2026-01-02     |

**Use Cases:** IoT, time-series, logs, analytics
**Popular Databases:** Cassandra, HBase, ScyllaDB, Bigtable

---

### **4.4 Graph Model**

* Stores **nodes and edges**
* Optimized for **relationships and traversals**

**Example:**

```text
Alice -[FRIEND]-> Bob
Bob -[FOLLOWS]-> Carol
```

**Use Cases:** Social networks, recommendation engines, fraud detection
**Popular Databases:** Neo4j, Amazon Neptune, ArangoDB

---

### **4.5 Relational / SQL Model (OLTP)**

**Definition:**

* Normalized **tables (aka entities) with rows and columns and relationships among the entities**
* Enforces **ACID transactions**

**Normalization Techniques:**

* Reduces redundancy and maintains consistency
* Standard normal forms: 1NF, 2NF, 3NF, BCNF

**Example:**

Normalized Orders database:

* **Orders Table:** order_id, customer_id, date_id
* **Customers Table:** customer_id, name, region
* **Products Table:** product_id, name, category

**Physical Considerations:**

* Data types, constraints (PK, FK, UNIQUE, NOT NULL)
* Indexes for fast lookups
* Row-oriented storage for transactional workloads

**Popular Databases:** AWS Aurora, PostgreSQL, Oracle, Microsfot SQL Server,MySQL etc

---

### **4.6 Dimensional / Star-Schema Model (OLAP / Analytics)**

**Definition:**
* **denormalization** based modelling to reduce the joins and optimize the SQL run time for repeated analytical queries
* **Star Schema** / **Snow Flake Schema** / **SCD - Slowly Changing Dimensions** : Fact tables , dimension tables 
* **one-wide-table (OWT)** : Fact table + all dimensional attributes.

**Use Cases:** OLAP queries, BI dashboards, reporting
**Popular Databases:** Snowflake, Redshift, BigQuery, Azure Synapse

---

### **4.7 Time-Series Model**

* Optimized for **timestamped events or measurements**

**Example:**

```
timestamp        sensor_id   temperature
2026-01-04 10:00  S1         22.5
2026-01-04 10:05  S1         22.8
```

**Use Cases:** Monitoring, IoT, metrics
**Popular Databases:** InfluxDB, TimescaleDB, Prometheus, OpenTSDB

---

### **4.8 Multi-Model / Polyglot Databases**

* Supports **multiple models** (KV, document, graph) in one system
* Physical storage may differ per model

**Use Cases:** Hybrid applications
**Popular Databases:** ArangoDB, OrientDB, Cosmos DB, PostgreSQL (JSON+relational)


### **4.9 OLAP Cubes / Multi-Dimensional**

* Pre-aggregated **multi-dimensional cubes** for analytics
* Often complements star-schema design

**Use Cases:** BI dashboards, reporting
**Popular Databases:** Cognos Cubes, Essbase Cubes, Microsoft SSAS, Mondrian, SAP BW


