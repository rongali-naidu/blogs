# Messaging Patterns vs Messaging Protocols

## Introduction

In modern **microservices architectures**, business processes often span multiple software applications. These applications need to exchange and process messages reliably.

The naïve approach is to use **direct API calls**, where one service calls another and waits for a response. While simple, this creates **tight coupling**, leading to bottlenecks, fragility, and cascading failures.

The solution lies in **decoupling services through messaging systems**. Messaging allows applications to communicate **asynchronously**, absorb spikes in load, and operate independently.

At the same time, the **demand for real-time analytics** has pushed these same messaging technologies into the **data engineering space**, giving rise to **event streams** and **streaming data platforms**.

Whether you’re a **Software Engineer** designing microservices or a **Data Engineer** building real-time analytics pipelines, understanding messaging systems is essential for designing **scalable, resilient systems**.

---

## Key Terminology

* **Event / Message** → A record representing a change in state or action.
* **Producer** → A system that emits events.
* **Consumer** → A system that receives and processes events.
* **Message Broker** → Middleware that routes messages between producers and consumers. Examples: **RabbitMQ, ActiveMQ, Amazon MQ, Kafka brokers, Kinesis, MQTT brokers (Mosquitto, EMQX, HiveMQ, AWS IoT Core)**.
* **Messaging Protocols** → Define *how* producers/consumers talk to brokers. (AMQP, MQTT, Kafka Protocol, HTTP/HTTPS).
* **Messaging Formats** → Binary (base64), JSON, Avro, Protobuf, etc.

---

## Messaging Patterns

### 1. **Message Queues (Point-to-Point)**

* **1:1 delivery** → One producer, one consumer.
* Great for **task distribution** and **work queues**.
* Examples: **SQS, RabbitMQ**.

### 2. **Publish-Subscribe (Pub/Sub)**

* **1\:many delivery** → One producer, many consumers.
* Great for **broadcasting events** and **triggering multiple downstream actions**.
* Examples: **SNS, RabbitMQ fan-out, MQTT topics**.

### 3. **Event Streams**

* Continuous flow of replayable events.
* Designed for **real-time analytics** and **parallel consumption**.
* Examples: **Kafka, Kinesis Streams**.
* Key features: **retention, replay, ordering, partitioning**.

👉 **Queues & Pub/Sub (SQS, SNS, RabbitMQ, MQTT)** → Best for **application and IoT integration**.

👉 **Streams (Kafka, Kinesis)** → Best for **real-time analytics and data processing**.

---

## Messaging Protocols

Different brokers use different wire protocols to move messages:

* **AMQP (Advanced Message Queuing Protocol)** → RabbitMQ, ActiveMQ.
* **MQTT (Message Queuing Telemetry Transport)** → Lightweight pub/sub protocol for IoT, mobile, and unreliable networks. Used by **Mosquitto, EMQX, HiveMQ, AWS IoT Core**. Optimized for **small payloads, low bandwidth, and constrained devices**. Supports QoS levels (at-most-once, at-least-once, exactly-once).
* **Kafka Protocol** → Custom binary protocol used by Kafka clients.
* **HTTP/HTTPS** → Used by SNS, SQS, Kinesis.

👉 Notice: **Kafka is a full platform**, while **MQTT is just a protocol (with many broker implementations)**.

---

## Characteristics of Messaging Systems

* **Ordering** → Preserve sequence of events.
* **Retention & Replay** → Store & re-read past events (Kafka offsets, Kinesis retention).
* **Routing** → Direct messages via topics, exchanges, or headers.
* **Durability** → Persist events despite crashes.
* **Scalability** → Partitioning, sharding, parallel readers.
* **Delivery Semantics** → At-most-once, at-least-once, exactly-once.
* **Backpressure Handling** → Buffering, throttling, consumer scaling.

---

## Event-Driven Architecture (EDA)

Messaging systems are the backbone of **EDA**. In EDA, events **trigger downstream actions**, enabling reactive, loosely coupled systems.

Example:

* A **payment service** emits an event.
* **Inventory service** updates stock.
* **Notification service** sends confirmation.
* **Analytics pipeline** processes metrics in real time.
* **IoT device** (via MQTT) streams sensor data into AWS IoT Core and triggers alarms.

---

## Kafka — Why Is It So Fast?

Two design choices stand out:

1. **Sequential I/O** → Appends are fast compared to random writes.
2. **Zero-Copy Principle** → Data is transferred directly from disk to network (via `sendfile()`), skipping application-level copies.

This makes Kafka a **high-throughput, low-latency backbone** for both microservices and analytics.

---

## Choosing Between **Kinesis Streams vs. SNS**

When building on AWS, a common question arises: *Should I use Kinesis Streams or SNS before routing data into Firehose for analytics?*

**Kinesis Streams**

* Retention (24h–7 days) & replay.
* High throughput with shards.
* Larger payloads (up to 1 MB).
* Designed for **real-time stream processing**.
* Cost: Higher than SNS.

**SNS (Simple Notification Service)**

* Simple pub/sub for event notification.
* No retention → fire-and-forget.
* Smaller payloads (256 KB).
* Cheaper than Kinesis.
* Ideal for **fan-out notifications** and **triggers**.

👉 Use **SNS** for lightweight notifications.

👉 Use **Kinesis Streams** for replayable, high-volume event pipelines.

---

## SNS + Lambda vs. SQS + Lambda

| Feature       | SNS + Lambda                | SQS + Lambda               |
| ------------- | --------------------------- | -------------------------- |
| Delivery Mode | Fire-and-forget             | Guaranteed until processed |
| Retry / DLQ   | Limited (no DLQ by default) | Built-in retries & DLQ     |
| Ordering      | Not guaranteed              | FIFO queues available      |
| Fan-out       | Yes                         | No (1:1)                   |
| Batching      | No                          | Yes (up to 10 messages)    |

👉 Use **SNS + Lambda** for **real-time triggers**.
👉 Use **SQS + Lambda** for **durable task processing**.

-
