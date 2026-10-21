
# Kafka’s Internal Design: Architecture, Offsets, and Consumer Mechanics

Source : https://notes.stephenholiday.com/Kafka.pdf

Apache Kafka is designed for **high-throughput, fault-tolerant, and distributed messaging**. Beyond the surface-level API, Kafka’s internal decisions around storage, offsets, and consumer logic are key to its performance and reliability. Here’s a deep dive.



## 1. Storage Architecture: Segment Files and Indexes

* **Log Segments:** Each partition’s messages are stored in **append-only segment files**, typically 1 GB each.
* **Index Files:** Kafka maintains an **offset index** mapping offsets to **byte positions** in each segment, enabling efficient retrieval.
* **Time Index Files:** Map timestamps to offsets for time-based lookups.
* **Snapshots:** Track producer state for leader elections and consistency.

**Directory Example:**

```
├── my-topic-0
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
```

---

## 2. Continuous Global Offsets Across Segment Files

* **Global Offsets:** Offsets are **partition-wide**, not per file.
  Example:

| Segment File             | Offset Range      |
| ------------------------ | ----------------- |
| 00000000000000000000.log | 0 → 999999        |
| 00000000000010000000.log | 1000000 → 1999999 |

* **Offset Indexing:** Kafka uses the index file to map offsets to **byte positions** for efficient reads.

---

## 3. Consumer Mechanics

* **Pull-Based Model:** Consumers fetch messages using `poll()` and can control consumption rates.
* **Consumer Groups:** Multiple consumers can form a group for **load-balanced partition consumption**, or different groups can independently consume the same topic.
* **Offset Tracking:** Consumers track offsets either automatically (`enable.auto.commit=True`) or manually (`consumer.commit()`).
* **Metadata Discovery:** Consumers connect to **bootstrap brokers** to fetch cluster metadata, discover partition leaders, and handle leader failovers automatically.

**Python Example (Consumer with manual commits):**

```python
from confluent_kafka import Consumer

consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'my-group',
    'enable.auto.commit': False,
    'auto.offset.reset': 'earliest'
})

consumer.subscribe(['my-topic'])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        print(f"Consumer error: {msg.error()}")
        continue
    print(f"Consumed message: {msg.value().decode('utf-8')}")
    consumer.commit(msg)
```

---

## 4. Producer Mechanics

* **Batching & Flushing:** Producers batch multiple messages and flush them based on `linger.ms` or batch size, improving throughput.
* **Message Visibility:** Messages are only **visible to consumers after they are flushed**. This introduces a small, configurable delay.
* **Acknowledgments:** Configurable via `acks`, balancing latency vs durability.

**Python Example (Producer):**

```python
from confluent_kafka import Producer

def delivery_report(err, msg):
    if err is not None:
        print(f"Message delivery failed: {err}")
    else:
        print(f"Message delivered to {msg.topic()} [{msg.partition()}] at offset {msg.offset()}")

producer = Producer({'bootstrap.servers': 'localhost:9092', 'acks': 'all', 'linger.ms': 100})

for i in range(100):
    producer.produce('my-topic', key=str(i), value=f'message-{i}', callback=delivery_report)

producer.flush()
```

---

## 5. Common Questions & Clarifications

### **Q1: Are messages available to consumers immediately?**

* **Yes.** Kafka appends messages to the log in memory, and they are immediately available for consumption.
* Flushing to disk happens asynchronously for durability and performance, but **consumers do not need to wait for this flush to see messages**.
* When a producer sends a message, Kafka **writes it to the in-memory page cache** of the broker (the OS cache).
* **Consumers can read messages as soon as they are appended to the log segment in memory**, **before a flush to disk occurs**.
* Flushing to disk (`log.flush.interval.messages` or `log.flush.interval.ms`) is about **durability**, not visibility to consumers.

### **Q2: Does Kafka client read from disk?**

* Conceptually, yes — consumers fetch messages from **partition logs (segment files)**.
* In practice, messages are often served from **OS page cache**, so reads are usually fast.

### **Q3: If segment files are large, and offsets are per file, how are messages tracked continuously?**

* Offsets are **partition-wide**, monotonically increasing.
* Segment files are just storage chunks; the **first message in each segment continues from the previous offset**.
* Index files map any offset to the correct byte position in the appropriate segment.

### **Q4: Can multiple consumers subscribe to the same topic?**

* Yes.

  * **Different consumer groups:** Each group receives all messages independently.
  * **Same consumer group:** Consumers share partitions; each partition is read by **one consumer** in the group.

### **Q5: How does a consumer know which broker has a partition?**

* Consumers use **bootstrap brokers** to fetch **metadata**, which includes:

  * Partition IDs
  * Leader broker for each partition
  * Replica brokers
* The client library handles all connection logic.

### **Q6: How does Kafka track how many messages are processed?**

* Kafka uses **offsets per partition per consumer group**.
* Consumers **commit offsets** either automatically or manually.
* `__consumer_offsets` topic stores committed offsets for recovery and lag measurement.

### **Q7: Is all this logic coded in the client library?**

* Yes. Kafka client libraries (Java, Python, Go, etc.) implement:

  * Metadata discovery
  * Partition assignment
  * Offset tracking & committing
  * Leader failover handling

### **Q8: Can Kafka span multiple brokers for the same topic?**

* Yes. Topics have **partitions**, each partition has a **leader on one broker**.
* Multiple partitions can be distributed across brokers, and **replicas provide fault tolerance**.

### **Q9: Can we use SNS for a central logging/event system instead of Kafka?**

* Yes, but SNS is **push-based and not replayable**.
* Kafka offers **durability, replay, ordering, and high throughput**, making it better for centralized logs and event streams.

