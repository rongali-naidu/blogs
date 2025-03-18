# System Design Patterns Data Engineers Use Most

System design patterns provide proven solutions to common problems encountered in building scalable and reliable data systems. Data engineers frequently adapt these patterns to handle large-scale data movement, transformation, and analysis. Here are some key patterns and how they apply to data engineering:

## 1. Batching Pattern (ETL - Extract, Transform, Load)

The **Batching Pattern** involves processing data in chunks or batches rather than in real time. This pattern is widely used in ETL workflows where large datasets are extracted from source systems, transformed for analytical use, and loaded into data warehouses or data lakes.
**Example:**
- **ETL Pipelines:** Periodically extract data from relational databases, apply transformations (e.g., cleaning, joining), and load it into Amazon Redshift or a data lake.

*Note 1: Change Data Capture (CDC) techniques identify the incremental data we need to extract from the source systems.*
*Note 2: we hear ELT Pattern but i am not convinced thats its fundamentally different from ETL so not adding it as separate pattern*

## 2. Event Sourcing Pattern

The **Event Sourcing Pattern** records the state of a system as a sequence of immutable events. Instead of storing only the latest state, all changes (events) are captured and can be replayed to reconstruct the system's state.
**Example:**
- **Database Change Capture:** Use DynamoDB Kinesis Stream to capture item-level modifications and push them to an Amazon Kinesis stream for further processing.
- **RDBMS Audits:** Track changes via database triggers and log them to an audit table, which can be ingested into a data lake.

## 3. Publish-Subscribe (Pub-Sub) Pattern
The **Pub-Sub Pattern** decouples producers and consumers by allowing multiple subscribers to receive messages from a publisher through a messaging system. This enables real-time data dissemination across multiple systems.
**Example:**
- **Event Broadcasting:** Use Amazon SNS to publish events (e.g., user sign-ups) and deliver them to multiple consumers like Amazon Kinesis Firehose for storage or AWS Lambda for real-time processing of the data into Datalake or Datwarehouse.

## 4. Stream Processing Pattern
The **Stream Processing Pattern** handles continuous data flows in real time. This pattern is essential for low-latency data analysis and decision-making.
**Example:**
- **Real-Time Analytics:** Process live clickstream data from websites using Amazon Kinesis or Apache Flink to track user activity in real-time.
- **Overlapping with Event Sourcing and Pub-Sub:** For instance, event streams from DynamoDB can be published to Kinesis for real-time processing and downstream analytics.

*Note: Change Data Capture (CDC) techniques identify the incremental data we need to extract from the source systems.*

## 5. Lambda Architecture Pattern
The **Lambda Architecture Pattern** combines batch and real-time processing, allowing systems to deliver both historical and low-latency insights. It consists of three layers: batch, speed (real-time), and serving layers.
**Example:**
- **Hybrid Analytics:** Use AWS Glue for batch ETL and Amazon Kinesis for real-time stream processing, both feeding into an Amazon S3 data lake for unified querying via Athena.

## 6. Data Mesh Pattern
The **Data Mesh Pattern** promotes a decentralized approach where different teams own and manage their data as a product. It ensures better scalability, ownership, and interoperability across domains.
**Example:**
- **Domain Data Products:** Each team maintains its datasets in separate Amazon S3 buckets. Other team could link these S3 buckets to their data lakes using AWS GLue.


## 7. LakeHub Pattern
This is another pattern i wanted to give a name to for future references. LakeHub pattern emphasizes ingesting data first into a data lake, which serves as a central hub before further transformations and loading into other systems like Datwarehose.
**Example:**
- **Staging Layer:** Store raw and semi-processed data in Amazon S3 before transforming and loading it into Redshift or Snowflake for analytical queries.

## 8. API-Based Data Ingestion (ADI) Pattern
This is another pattern i wanted to give a name to for future references. The **API-Based Data Ingestion (ADI) Pattern** involves querying APIs (both internal and external) to extract data and load it into a data lake or warehouse for further analysis. Internal systems often expose REST APIs to provide controlled access to data instead of direct database queries.
**Example:**
- **API Data Collection:** Regularly call REST APIs from internal systems or third-party sources to fetch data (e.g., operational logs, weather data) and store it in Amazon S3 for analysis.

## 9. DataLink Pattern
This is another pattern i wanted to give a name to for future references. This is very similar to DB Links concept. The **DataLink Pattern Pattern** facilitates controlled sharing of datasets ( without the need of copying the data) across teams, organizations, or external partners while maintaining data privacy and security. This pattern is crucial for collaboration without data duplication.
**Example:**
- **Cross-Account Sharing:** Use AWS Lake Formation to securely share Amazon S3 data between business units without copying the datasets.
- **Redshift Data Sharing**
- **Data Mesh Pattern** overlaps with DataLink Pattern


