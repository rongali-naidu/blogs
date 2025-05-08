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
  * **Event (Data) Streams** are built from the ground up for **real-time analytical processing**, with built-in **durability, replayability, and high-throughput** capabilities.
  * **Data Streaming Platform** is designed to work with **Event Streams** and these are consumers of Message Borker/Message Queue/Pub-sub or Event bus.
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
* Topics and Exchanges
   * These are used for implenting Routing 
   * Topic is a named channel where producers send and subscribers listen. Multiple consumers can get the same message
   * Exchange is  A router (used in RabbitMQ) that delivers messages to queues based on rules (fan-out, direct, topic, headers)
* Fan-Out
   * Fan-out is Pub-Sub for all the practical purpises.
* Stateless Processing 
   * **State** refers to data a system keeps over time (like past events or session info).**stateless service** doesn't retain memory or history between requests. It can **read external state** (like from a database), but it does not store any state in **its own memory** between processing of the subsequent events.
* Event-Driven Architecture (EDA)
   * **EDA** is a design pattern where **events** trigger downstream actions. Messaging systems help implement Event-Driven Architectures by providing the infrastructure for event-based communication.


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



### Chosing **Kinesis Streams** VS **SNS* for Event streaming before its collected through Firehose for analytics?

When deciding between **Kinesis Streams** and **SNS** for event streaming before collecting the data through **Firehose** for analytics, it's important to consider the specific characteristics and use cases of each service:

#### **Kinesis Streams**:

* **Stream Processing**: Kinesis Streams is designed for real-time stream processing and is ideal for use cases where you need fine-grained control over event ordering, message retention, and parallel data consumption.
* **Message Retention**: It stores data for a configurable duration (24 hours to 7 days), allowing consumers to reprocess (or replay) the events from the stream.
* **Scaling and Parallel Processing**: Kinesis Streams supports high throughput and allows multiple consumers to process data in parallel using shards.
* **Pay load**: It can handle larger event sizes — up to **1 MB per record**. This makes it suitable for scenarios where you're processing large payloads of data, such as log entries, raw sensor data, or other high-volume, high-size events that need to be processed in real-time.
* **Use Case**: It's suited for scenarios that require advanced stream processing, like real-time data transformations, analytics, or high-volume event processing.

#### **SNS (Simple Notification Service)**:

* **Event Notification**: SNS is more suited for lightweight event notifications and pub/sub patterns, where messages are sent from a publisher to multiple subscribers. It is simple to set up and is designed to fan out messages to multiple destinations, including Lambda functions, SQS queues, and HTTP endpoints.
* **Event Delivery**: SNS is ideal for event-driven architectures where real-time notifications or triggers are required but doesn’t have the same level of stream processing capabilities as Kinesis.
* **Message Retention**: SNS does not retain messages by default — once a message is delivered to subscribers, it’s discarded.
* **Pay load**: SNS has a **message size limit of 256 KB per message**. If your event data exceeds this size, SNS won't be able to handle it. This makes SNS less suitable for large payloads unless you break the data into smaller chunks or use other methods to store large objects (e.g., S3 links).
* **Use Case**: SNS is better for scenarios where you need to notify multiple subscribers of an event or trigger an immediate action, but not for complex stream processing.




### ⚙️ SNS + Lambda vs. SQS + Lambda

| Feature             | SNS + Lambda                | SQS + Lambda                        |
| ------------------- | --------------------------- | ----------------------------------- |
| **Delivery Mode**   | Fire-and-forget             | Guaranteed delivery until processed |
| **Retries**         | Limited (no DLQ by default) | Built-in retry & DLQ support        |
| **Ordering**        | Not guaranteed              | FIFO support available              |
| **Fan-out Support** | Yes                         | No (1:1 queue consumption)          |

> Use **SNS** for **broadcasting** and **real-time reactions**.
> Use **SQS** for **guaranteed**, **durable**, and **scalable** task processing.

