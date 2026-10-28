# Kafka Interview Answer for a QA

A complete answer framework: what Kafka is, its components, how to describe your testing, and likely follow-up questions.

## 1. What is Kafka? (open with this)

> "Kafka is a distributed event streaming platform. Instead of services calling each other directly, a service publishes events to Kafka, and other services subscribe and consume them independently. Events are stored durably in an ordered log, so services are decoupled, scale independently, and can replay events if needed. In our project we used it for communication between microservices, for example Order, Payment and Inventory."

## 2. Key components at a high level

| Component | One-line explanation | Why a QA cares |
|---|---|---|
| **Producer** | Application that publishes events | Verify the right event, key and payload are sent |
| **Consumer** | Application that reads events | Verify the event is processed correctly |
| **Topic** | Named stream of events (e.g. `orders.created`) | Verify events land on the correct topic |
| **Partition** | A slice of a topic. Ordering is guaranteed only within a partition | Ordering tests: same key goes to the same partition |
| **Offset** | Position of a message within a partition | Shows what has been consumed, and enables replay |
| **Broker / Cluster** | Kafka servers that store and serve data | Broker failure and restart scenarios |
| **Replication** | Each partition is copied to other brokers (one leader, rest followers) | No data loss on broker failure |
| **Consumer group** | Consumers sharing a topic's partitions. Each service usually has its own group | Each service receives every event independently |
| **Consumer lag** | How far a consumer is behind the latest message | Key signal in performance testing |
| **Dead letter topic (DLT)** | Holds messages that repeatedly fail processing | Error-handling tests |
| **Schema Registry** | Stores and enforces event schemas | Contract and backward-compatibility tests |
| **Controller (KRaft)** | Manages cluster metadata and leader election | Platform-level, good to know |
| **Kafka Connect** (optional) | Moves data between Kafka and other systems | Mention only if your project used it |

**The flow to remember:**

> Producer → Topic (partitions, replicated across brokers) → Consumer group → Service

**Say it out loud:**

> "A producer writes events to a topic. The topic is divided into partitions, which are replicated across brokers for fault tolerance. Consumers in a consumer group read from those partitions, and each consumer tracks its progress using offsets."

**Why Kafka is used:** decoupling, scalability (more partitions and consumers means more throughput), durability and replay, and resilience (events wait if a consumer is down).

**Delivery guarantee:**

> "By default Kafka gives at-least-once delivery, so duplicates are possible. That's why we test that consumers are idempotent."

## 3. 60-90 second answer: how we tested Kafka

I don't know your project's details, so keep only what you actually did. A smaller true answer beats a bigger invented one.

> "In our project, the Order service published an `OrderCreated` event to a topic, and the Payment and Inventory services consumed it in their own consumer groups.
>
> As a QA, I tested at three levels. First, the **producer side**: I triggered the business action through the API, then verified the right event reached the right topic with the correct key, payload and schema. Second, the **consumer side**: I published test events directly to the topic and verified the service processed them correctly, meaning the DB was updated or a downstream event was emitted. Third, **end-to-end flows** across services, such as order to payment to inventory, including failure paths.
>
> Beyond happy paths, I focused on Kafka-specific risks: duplicates and idempotency, ordering for the same key, retries and dead letter topics for bad messages, consumer lag under load, and consumer restart after downtime. I used [your tools] and polling with timeouts to handle the asynchronous behavior."

## 4. What to test, mapped to Kafka concepts

| Area | What you verify | Kafka concept |
|---|---|---|
| **Producer** | Correct topic, key, headers and payload | Topic, key, partition |
| **Consumer** | Event processed once, result correct (DB state, downstream event) | Consumer group, offset |
| **Schema / contract** | Payload matches schema, new versions don't break consumers | Schema Registry, backward compatibility |
| **Ordering** | Events for the same entity are processed in order | Same key, same partition |
| **Idempotency** | Same event sent twice doesn't create duplicate orders or payments | At-least-once delivery |
| **Error handling** | Bad message goes to the DLT after retries and doesn't block the partition | Retries, DLT |
| **Resilience** | Consumer down, then restarts and catches up with no loss or double-processing | Offset commit, retention |
| **Replay** | Resetting offsets reprocesses events safely | Offsets, retention |
| **Performance** | Lag stays acceptable and throughput meets target | Consumer lag, partitions |
| **Broker failure** (if done) | No data loss after broker restart or leader change | Replication, ISR, `acks=all` |

## 5. Practical approach and tools

### Testing an asynchronous system

- Don't assert immediately. Poll with a timeout (e.g. Awaitility in Java), not fixed sleeps.
- Use unique test data, such as a unique `orderId` per test, so you can find your event and avoid interference.
- Verify the **outcome** (DB record, API response, downstream event), not just that a message was sent.

### Tools (mention only those you used)

- **Inspecting data:** Kafka UI, Offset Explorer, AKHQ, `kcat`
- **CLI:** `kafka-console-producer`, `kafka-console-consumer`, and `kafka-consumer-groups.sh --describe` for offsets and lag
- **Automation:** a Kafka client in your framework (Java, Python or Node) with REST Assured, Postman or Karate for the API side
- **Integration tests:** Testcontainers or embedded Kafka
- **Contract testing:** Pact or Schema Registry compatibility checks
- **Performance and monitoring:** JMeter or k6 for load, Grafana/Prometheus for lag and throughput

## 6. Example scenarios to quote

- **Happy path:** place an order, verify `OrderCreated` on the topic with the right fields, then verify the payment record is created.
- **Duplicate event:** publish the same event twice and verify only one payment is created.
- **Poison message:** publish malformed JSON, verify it lands in the DLT after retries and the next valid message still gets processed.
- **Consumer downtime:** stop the consumer, publish 100 events, restart, and verify all 100 are processed and lag returns to zero.
- **Ordering:** publish `Created`, `Paid`, `Shipped` for one order and verify the final state is correct.
- **Backward compatibility:** add an optional field to the schema and verify old consumers still work.

## 7. Likely follow-up questions

**How do you test something asynchronous?**
Poll for the expected outcome with a timeout rather than fixed sleeps, and use unique IDs to isolate each test.

**How do you check a message was consumed?**
Check the consumer group's committed offset and lag with `kafka-consumer-groups`, and verify the side effect such as the DB update or downstream event.

**What is consumer lag and why does it matter?**
It's how far a consumer is behind the latest offset in a partition. Rising lag means the service can't keep up, so I watch it during load tests.

**What happens if a message fails processing?**
It's retried a limited number of times, then routed to a dead letter topic so it doesn't block the partition. I test that routing works and that failed messages can be inspected and reprocessed.

**How do you test for duplicates?**
Kafka is at-least-once by default, so duplicates can happen on retries or rebalances. I replay the same event and verify the consumer is idempotent.

**What's the difference between partitions and replicas?** (common trap)
Different partitions hold **different** messages, which gives scalability. Replicas of the same partition hold **identical** messages, which gives fault tolerance.

**What happens if a broker goes down?**
Each partition has a leader and followers on other brokers. If a leader's broker fails, an in-sync follower is promoted and clients retry automatically. With `acks=all` and `min.insync.replicas=2`, acknowledged writes are not lost.

**How do you handle test data in a shared topic?**
Unique keys or IDs per test, filtering by them, and isolated test topics or consumer groups where possible.

**How is it different from testing a REST API?**
There's no immediate response, so I verify through outcomes and offsets. I also have to consider ordering, duplicates, retries, schemas and lag, which don't exist in request/response testing.

## 8. Interview tips

- Follow this order: what Kafka is, then components, then one concrete flow from your project, then how you tested, then tools.
- Use the terms **topic, partition, consumer group, offset, lag, DLT, idempotency** naturally, since they show real understanding.
- If you didn't do something, such as broker failover testing, say: "I understand how it would be tested, though in our project the platform/DevOps team handled that." That is honest and still shows knowledge.
