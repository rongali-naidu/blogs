# Understanding Messaging Systems and Data Streaming in Modern Software Architecture and Modern Data Architecture

### Introduction

In modern software architectures, business processes often require integration between multiple software applications. These applications need to exchange and process messages. However, **tight coupling** between services (e.g., direct API calls where one service waits for a response) can lead to performance bottlenecks and system fragility.

The solution lies in **decoupling services through messaging systems**, allowing them to communicate **asynchronously** and operate independently.

At the same time, the growing demand for **real-time analytics** has driven the adoption of these messaging systems within modern data architectures — and has led to the emergence of analytics-focused **data streaming platforms**.

Whether you're a **Software Engineer** or a **Data Engineer**, understanding these messaging patterns equips you with the right knowledge to design scalable and resilient systems — both for software integration and data processing.

### Terminology

* **Event / Message**: A record representing a change in state or an action taken.
* **Producers**: Systems that emit those events.
* **Consumers**: Systems that receive and process those events.
* **Message Systems** 

  * **Message Systems** : Umbrella term for systems that allow services or components to communicate by sending (publisher) and receiving messages asynchronously (subscirber or consumer) .
  * **Message Broker** :A component or Software  that acts as a middleman between producers and consumers. These Brokers could use any of the following Messaging Patterns or Mix of the patterns. Ex : Apache ActiveMQ , RabbitMQ, AWS SQS, AWS SNS ,Apache Kafka, AWS Kinesis Stream etc
  * **Messaging Patterns**
    * **Message Queues**: Provide a **1:1 delivery model** — one producer, one consumer. This is typical for task processing or work distribution. Ex: SQS,RabbitMQ
    * **Publish-Subscribe (Pub/Sub)**: Provide a **1\:many delivery model** — one producer, many consumers. This is typical for broadcasting events or triggering multiple downstream actions.Ex: SNS, 
    * **Event (Data) Streams**: Represents a **continuous flow of data in real time** which is usually replayable log of events, typically used in **real-time data processing and analytics**. One of the key expectations of event streams is the ability to **store and replay events** over a short retention period (e.g., 1 to 7 days), enabling reprocessing, parallel consumption, and time-windowed analysis. Ex: Kafka, Kinesis Stream etc.
  * **Message Queues** and **Publish-Subscribe (Pub/Sub)** are primarily designed for **service integration** in software systems. 
  * **Event Streams** are built from the ground up for **real-time analytical processing**, with built-in **durability, replayability, and high-throughput** capabilities.
  * While both **Message Queues** and **Pub/Sub systems** can carry streams of events . with support from tools like **Firehose**, **Lambda**, or **streaming ETL**, they are increasingly being integrated into   **real-time analytics platforms**. You may wonder "why we need separate Event Stream or Streaming Data Platforms?" . I asnwered this under chosing **Kinesis Streams** VS **SNS* for Event streaming before its collected through Firehose for analytics
* **Messaging Protocols**
    * Protocols define how producers and consumers communicate with brokers.
    * AMQP (Advanced Message Queuing Protocol) → Used by RabbitMQ, ActiveMQ
    * MQTT (Message Queuing Telemetry Transport) → Used in IoT & real-time messaging
    * Kafka Protocol → Custom binary protocol used by Kafka clients
    * HTTP/HTTPS → Used by SNS, SQS, Kinesis Streams
* **Messaging Format**
    * Different formats are used.
    * SQS/SNS/Kinesis Streams usaully use binary format (base64) for exchanging the data.






### **Key Characteristics of Messaging Systems**

1. **Ordering**
   Ensures that messages or events are delivered in the same sequence in which they were produced — crucial for systems requiring strict consistency.

2. **Retention**
   The ability to store messages/events for a configurable duration, allowing consumers to retrieve or reprocess them later.

3. **Replay**
   Replay support allows consumers to **re-read past messages**, even after initial consumption.
   *Example*: Kafka consumers track offsets to reprocess from any point in the log.

4. **Routing**
   The mechanism to direct messages to appropriate subscribers based on rules, topics, headers, or content (e.g., topic-based, header-based, or content-based routing).

5. **Continuity**
   Refers to the **flow rate** or **frequency** of event generation — whether it's continuous (streaming) or intermittent.

6. **Real-time**
   Indicates whether the events are published and delivered **as they happen**, enabling low-latency, reactive systems.

7. **Synchronous vs Asynchronous**
   In system integration:

   * **Synchronous** = direct calls (e.g., API/HTTP) waiting for response; tight coupling.
   * **Asynchronous** = decoupling via message queues or brokers, enabling independent processing.

8. **Durability**
   Ensures that messages aren't lost even if the messaging system or consumers crash. Durable queues/logs guarantee message persistence.

9. **Scalability**
   The ability to handle increasing volumes of messages, producers, and consumers without performance degradation — often via partitioning, load balancing, or sharding or parallel reading.
    * Partitioning / Sharding: Distribute messages across partitions for parallel consumption.
    * Parallel Reading: A single consumer can read and process multiple messages concurrently using threads, async tasks, or worker pools.
    * Horizontal Scaling: Add more consumers or consumer instances for distributed processing.

11. **Acknowledgement & Delivery Semantics**
    Acknowledgements are used to track successful message processing. Messaging systems guarantee different levels of message delivery based on how they handle acknowledgements and failures.

    * **At-most-once**: no duplicates, but possible loss.
    * **At-least-once**: guaranteed delivery, but possible duplicates.
    * **Exactly-once**: no loss, no duplicates (harder to implement).
     

13. **Backpressure Handling (Flow Control)**
    The system's ability to handle slow consumers or spikes in producer throughput by buffering, throttling, or applying flow control.buffering has limits — large backlogs can exhaust memory/disk.Throttling (Rate Limiting) producers helps in avoiding the overwhelming the system. From consumer side, adding more consumer instances to process the backlog faster helps.




## Message Broker

A **Message Broker** is a software component that enables services to **communicate asynchronously** by **receiving**, **storing**, **routing**, and **delivering messages** between producers (senders) and consumers (receivers).

> It decouples producers and consumers in **time**, **space**, and **technology**.


## Why Routing is Relevant

Routing in message brokers ensures that messages are delivered to the **right consumer(s)** based on rules like:

* Message content
* Queue name
* Topic patterns
* Header or metadata filters

This is especially important in systems like **RabbitMQ** (using exchanges) or **EventBridge** (using routing rules).

---

## Are Message Queues and Message Brokers the Same?

Not exactly.

* **Message Queue**: A data structure used to store and forward messages to a single consumer (point-to-point).
* **Message Broker**: A full system that may include queues, topics, routing logic, transformations, retries, etc.

> A broker **may implement queues**, but it's more than just a queue.


## Topics and Exchanges

| Concept      | Description                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| **Topic**    | A named channel where producers send and subscribers listen. Multiple consumers can get the same message.      |
| **Exchange** | A router (used in RabbitMQ) that delivers messages to queues based on rules (fan-out, direct, topic, headers). |

> **Analogy**: A topic is like a **radio station** — anyone tuned in hears the broadcast.

---

## What is Fan-Out?

**Fan-out** is a **messaging pattern** where a single message is delivered to **multiple consumers**. It’s like broadcasting — one sender, many receivers.

Used in:

* **SNS**
* **RabbitMQ fanout exchange**
* **Kafka topics (with multiple consumer groups)**

---

## ⚙️ SNS + Lambda vs. SQS + Lambda

| Feature             | SNS + Lambda                | SQS + Lambda                        |
| ------------------- | --------------------------- | ----------------------------------- |
| **Delivery Mode**   | Fire-and-forget             | Guaranteed delivery until processed |
| **Retries**         | Limited (no DLQ by default) | Built-in retry & DLQ support        |
| **Ordering**        | Not guaranteed              | FIFO support available              |
| **Fan-out Support** | Yes                         | No (1:1 queue consumption)          |

> Use **SNS** for **broadcasting** and **real-time reactions**.
>
> Use **SQS** for **guaranteed**, **durable**, and **scalable** task processing.


## What is Stateless Processing?

A **stateless service** doesn't retain memory or history between requests.

* It can **read external state** (like from a database), but it does not store any state in **its own memory**.
* Useful for **scalable**, **resilient**, and **ephemeral** processing (e.g., Lambda).

> **State** refers to data a system keeps over time (like past events or session info).

---

## Event-Driven Architecture (EDA)

**EDA** is a design pattern where **events** trigger downstream actions.

* **Event**: A Record that represents a change in state or an action taken.
* **Producers** The systems which emit those events.
* **Consumers** The Systems that consumes those events and process them.

### Why Use EDA?

* Loose coupling between systems
* Real-time responsiveness
* Scalability and extensibility

---

## Messaging Boker Vs Data Streaming Platform 
* A Message Broker is a software system that enables different components of a distributed application to communicate asynchronously by exchanging messages.
* A Data Streaming Platform is designed for capturing, processing, storing continuous streams of data in real time. Data stream couldb be consumer of Message Borker/Message Qyeye/Pub-sub or Event bus.

## Messaging Boker Vs Message Queue
*	A system that routes, stores, and manages messages between systems.Broader: includes support for multiple messaging patterns like queueing, pub/sub, routing
*	A specific messaging pattern .. focuses on point-to-point delivery via a queue

## Messaging Queue Vs Data Stream
* Data Stream : Publish and Collect time-ordered events for analysis or processing
* Decouple sender/receiver, process tasks asynchronously. Point-to-point (1 consumer gets the message). Data streams could be implemented using either Queues or Pub/Sub .
* 
## Messaging Patterns: Comparison

| Feature           | Message Queue  | Pub/Sub System              | Event Bus                      |
| ----------------- | -------------- | --------------------------- | ------------------------------ |
| Pattern           | Point-to-point | One-to-many (broadcast)     | One-to-many with routing logic |
| Consumers per msg | One            | All subscribers             | Filtered subscribers           |
| Routing           | Queue name     | Topic pattern               | Rules and filters              |
| Persistence       | Yes            | Depends (e.g., Kafka = yes) | Yes / Optional                 |
| Use Case          | Task queue     | Notifications               | Complex service orchestration  |


## Messaging & Streaming Tech Compared

| Technology          | Type            | Persistence | Fan-out        | Ordering  | Built-in Routing | Notes                                 |
| ------------------- | --------------- | ----------- | -------------- | --------- | ---------------- | ------------------------------------- |
| **RabbitMQ**        | Message Broker  | Yes         | Yes            | Limited   | Yes (Exchanges)  | Versatile, supports multiple patterns |
| **MQTT**            | Pub/Sub (IoT)   | Optional    | Yes            | Limited   | Topic-based      | Lightweight, ideal for IoT            |
| **Amazon SQS**      | Message Queue   | Yes         | No             | FIFO opt. | No               | Scalable, simple queueing             |
| **Amazon SNS**      | Pub/Sub         | No          | Yes            | No        | Topic-based      | Fan-out to multiple endpoints         |
| **Kinesis Streams** | Stream          | Yes         | Consumer-group | Yes       | Shard-based      | Real-time data ingestion              |
| **Apache Kafka**    | Distributed Log | Yes         | Consumer-group | Yes       | Topic/partition  | High-throughput, replayable           |

| Feature / Tech   | RabbitMQ                       | MQTT              | Amazon SQS                       | Amazon SNS             | Kinesis Streams                 | Apache Kafka                |
| ---------------- | ------------------------------ | ----------------- | -------------------------------- | ---------------------- | ------------------------------- | --------------------------- |
| **Type**         | Message Broker                 | Pub/Sub Protocol  | Message Queue                    | Pub/Sub Notifier       | Data Stream (Managed)           | Distributed Log System      |
| **Architecture** | Broker-based                   | Broker-based      | Fully Managed Queue              | Fully Managed Pub/Sub  | Brokered Stream                 | Brokered Stream             |
| **Use Case**     | Enterprise Messaging           | IoT & Lightweight | Decoupling apps                  | Fanout messaging       | Real-time analytics             | Event streaming & pipelines |
| **Ordering**     | Optional                       | No                | Not guaranteed                   | No                     | Guaranteed per shard            | Strong (per partition)      |
| **Retention**    | Ack-based + TTL                | Minimal           | Up to 14 days                    | None (fire-and-forget) | 24h to 7 days                   | Configurable (forever+)     |
| **Protocol**     | AMQP/STOMP/MQTT                | MQTT              | HTTP API                         | HTTP/SMS/Email         | HTTP/Kinesis SDK                | Kafka Protocol              |
| **Scaling**      | Manual (clustered)             | Depends on broker | Auto (serverless)                | Auto                   | Shards (manual tuning)          | Partitions (manual or auto) |
| **Latency**      | Low to Medium                  | Very Low          | Low                              | Very Low               | Very Low                        | Very Low                    |
| **Durability**   | High (with config)             | Low to Medium     | High                             | Low (no retries)       | High                            | Very High                   |
| **Exactly Once** | Hard to guarantee              | No                | No                               | No                     | No                              | Yes (with config)           |
| **Best For**     | Complex routing, back-end jobs | IoT devices       | Decoupling producers & consumers | Broadcasting alerts    | Real-time streaming & analytics | Event-driven architectures  |

## Chosing **Kinesis Streams** VS **SNS* for Event streaming before its collected through Firehose for analytics?

When deciding between **Kinesis Streams** and **SNS** for event streaming before collecting the data through **Firehose** for analytics, it's important to consider the specific characteristics and use cases of each service:

### **Kinesis Streams**:

* **Stream Processing**: Kinesis Streams is designed for real-time stream processing and is ideal for use cases where you need fine-grained control over event ordering, message retention, and parallel data consumption.
* **Message Retention**: It stores data for a configurable duration (24 hours to 7 days), allowing consumers to reprocess (or replay) the events from the stream.
* **Scaling and Parallel Processing**: Kinesis Streams supports high throughput and allows multiple consumers to process data in parallel using shards.
* **Pay load**: It can handle larger event sizes — up to **1 MB per record**. This makes it suitable for scenarios where you're processing large payloads of data, such as log entries, raw sensor data, or other high-volume, high-size events that need to be processed in real-time.
* **Use Case**: It's suited for scenarios that require advanced stream processing, like real-time data transformations, analytics, or high-volume event processing.

### **SNS (Simple Notification Service)**:

* **Event Notification**: SNS is more suited for lightweight event notifications and pub/sub patterns, where messages are sent from a publisher to multiple subscribers. It is simple to set up and is designed to fan out messages to multiple destinations, including Lambda functions, SQS queues, and HTTP endpoints.
* **Event Delivery**: SNS is ideal for event-driven architectures where real-time notifications or triggers are required but doesn’t have the same level of stream processing capabilities as Kinesis.
* **Message Retention**: SNS does not retain messages by default — once a message is delivered to subscribers, it’s discarded.
* **Pay load**: SNS has a **message size limit of 256 KB per message**. If your event data exceeds this size, SNS won't be able to handle it. This makes SNS less suitable for large payloads unless you break the data into smaller chunks or use other methods to store large objects (e.g., S3 links).
* **Use Case**: SNS is better for scenarios where you need to notify multiple subscribers of an event or trigger an immediate action, but not for complex stream processing.


