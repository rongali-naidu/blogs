## **Demystifying PostgreSQL Replication: WAL, WAL Decoding, and the Journey Toward Zero-ETL**

PostgreSQL is known for its robustness and extensibility, and at the heart of its durability and replication mechanisms lies the **Write-Ahead Log (WAL)**. Whether you're ensuring high availability with cluster replication or building a modern Change Data Capture (CDC) pipeline, understanding how PostgreSQL uses WAL — and how it compares with tools like AWS DMS and Zero-ETL — is critical.



### What is WAL?

The [Write-Ahead Log (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) is PostgreSQL’s foundational mechanism for ensuring **durability** and **crash recovery**. Every time data is modified, PostgreSQL first writes a record of the change to the WAL — *before* applying it to the actual data files.

Note : if you are familiar with Oracle, this is related to Oracle REDMO management.

This guarantees:

* **Durability** (the "D" in ACID) — no committed transaction is lost.
* **Crash Recovery** — WAL is replayed to recover state after a crash.
* **Foundation for Replication** — WAL powers both physical and logical replication.

> WAL entries are stored in a **binary format** and are written sequentially for performance.



### WAL Decoding: Making Change Data Streamable

While WAL ensures resilience, it isn’t readable by humans or directly usable for downstream systems.

**WAL Decoding** is the process of converting WAL’s binary entries into **logical, row-level change events** like:

```json
{"action": "INSERT", "table": "orders", "columns": {"id": 1, "status": "shipped"}}
```

This decoding enables:

* **Logical Replication**
* **Change Data Capture (CDC)**
* **Integration with Kafka, Redshift, or S3**

#### Tools for WAL Decoding:

* [**pgoutput**](https://www.postgresql.org/docs/current/protocol-logical-replication.html): Built-in logical decoding plugin used by native logical replication.
* [**test\_decoding**](https://www.postgresql.org/docs/current/test-decoding.html): Simple text-based plugin, great for learning and debugging.
* **Third-party tools** like [Debezium for PostgreSQL](https://debezium.io/documentation/reference/connectors/postgresql.html), which integrates with Kafka Connect to build real-time data pipelines.


### Physical vs Logical Replication in PostgreSQL

PostgreSQL supports two core types of replication: **Physical (Streaming)** and **Logical**.

#### 1. **Physical (Streaming) Replication**

* Streams **raw WAL segments** to a replica server.
* Maintains a **byte-for-byte copy** of the primary.
* Used for:

  * **High availability**
  * **Read scaling** (hot standby)
* Requires **same PostgreSQL version** and similar configurations.
* Reference: [PostgreSQL Physical Replication Guide](https://www.postgresql.org/docs/current/warm-standby.html)

#### 2. **Logical Replication (Publication/Subscription)**

* Introduced in [PostgreSQL 10](https://www.postgresql.org/docs/current/logical-replication.html).
* Uses **WAL decoding** to send row-level changes.
* You define a:

  * **Publication** on the source
  * **Subscription** on the target
* Benefits:

  * Table-level granularity
  * Cross-version replication
  * Heterogeneous targets
* Limitation:

  * Only supports DML (`INSERT`, `UPDATE`, `DELETE`) — **not DDL** (schema changes).

Example : 

Scenario: While setting up provivisoned Postgresql cluster , we wanted to copy from Serveleess based Postgresql cluster, 
we wanted to sync the data so used the following replication 

```sql
on Source DB:

CREATE PUBLICATION replication_publication FOR TABLE 
    <schema1>.<table_name>,
	<schema2>.<table_name2>
	
on Target DB:

CREATE SUBSCRIPTION replication_subscription 
CONNECTION 'host={sourceEndpoint} port={port} dbname={dbname} user={db_user} password={password}'
PUBLICATION replication_publication
WITH (create_slot = true, enabled = false, copy_data = true);

ALTER SUBSCRIPTION replication_subscription ENABLE;






Cancelling the publication 

Target db :
ALTER SUBSCRIPTION replication_subscription DISABLE;
drop SUBSCRIPTION replication_subscription


on Source DB:

SELECT pg_drop_replication_slot('replication_subscription');
OR
drop PUBLICATION replication_publication

```
	


### WAL-Based Replication vs AWS DMS

[**AWS Database Migration Service (DMS)**](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.html) is a cloud-native service for migrating and replicating data across heterogeneous systems (e.g., PostgreSQL → Redshift, MySQL → Kafka).

#### How DMS Uses WAL:

* DMS can be configured with a **logical replication slot**.
* It uses plugins like `pgoutput` or `test_decoding` to read WAL and emit structured changes.
* Supports full-load + incremental CDC.

| Feature            | PostgreSQL Logical Replication | AWS DMS                     |
| ------------------ | ------------------------------ | --------------------------- |
| WAL Decoding       | ✅ Built-in (`pgoutput`)        | ✅ Reads via plugin          |
| Target Flexibility | PostgreSQL only                | ✅ Kafka, S3, Redshift, etc. |
| Transformation     | ❌ Minimal                      | ✅ Supports light mapping    |
| Built-in           | ✅ Native PostgreSQL feature    | ❌ External AWS service      |
| Setup Complexity   | Medium                         | Medium                      |
| Use in HA          | ✅ Yes                          | ❌ Not intended for HA       |

Further reading:

* [DMS PostgreSQL Source Docs](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html)
* [Logical Decoding with DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html#CHAP_Source.PostgreSQL.Requirements)

---

### The Rise of Zero-ETL and Its Relationship

**Zero-ETL** represents a cloud-native evolution of CDC, offering **seamless, near real-time integration between OLTP and analytical systems** — *without the manual effort of building pipelines*.

Example: [**Amazon Aurora Zero-ETL integration with Amazon Redshift**](https://docs.aws.amazon.com/redshift/latest/dg/zero-etl.html)

These systems:

* Internally use **WAL or binlog-based decoding** under the hood.
* Push data directly to analytical stores like **Redshift** in near real time.
* Offer out-of-the-box reliability, scaling, and schema evolution handling.
* [**More details on how Aurora Zero-ETL works**](https://aws.amazon.com/blogs/database/amazon-aurora-postgresql-zero-etl-integration-with-amazon-redshift-is-generally-available/)*  

| Feature                | Logical Replication | AWS DMS | Aurora Zero-ETL        |
| ---------------------- | ------------------- | ------- | ---------------------- |
| Uses WAL               | ✅ Yes               | ✅ Yes   | ✅ Yes (under the hood) |
| Setup Effort           | Medium              | Medium  | ✅ Low                  |
| Transformation Support | ❌ None              | ✅ Light | ✅ Partial              |
| Target Flexibility     | ❌ PostgreSQL only   | ✅ Many  | ❌ Redshift-only        |
| Managed Infrastructure | ❌ Self-managed      | ✅ AWS   | ✅ Fully managed        |



### 📚 References

* [PostgreSQL WAL Internals](https://www.postgresql.org/docs/current/wal-intro.html)
* [Logical Decoding and Plugins](https://www.postgresql.org/docs/current/logicaldecoding.html)
* [Logical Replication Guide](https://www.postgresql.org/docs/current/logical-replication.html)
* [PostgreSQL Physical Replication](https://www.postgresql.org/docs/current/warm-standby.html)
* [AWS DMS for PostgreSQL](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html)
* [Debezium PostgreSQL Connector](https://debezium.io/documentation/reference/connectors/postgresql.html)
* [Aurora Zero-ETL to Redshift](https://docs.aws.amazon.com/redshift/latest/dg/zero-etl.html)

