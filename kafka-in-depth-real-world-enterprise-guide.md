# Apache Kafka — In-Depth Real-World Guide for Java and Spring Boot Developers

A practical guide to Kafka fundamentals, internals, Java/Spring Boot implementation, enterprise integration patterns, reliability, operations, troubleshooting, interview questions, and hands-on projects.

> **Version note:** Kafka client and broker features vary by Kafka release. Spring Kafka APIs also vary by Spring Boot/Spring Kafka version. Check the compatibility matrix for the versions used by your project. Code below is illustrative and should be adapted and tested against your dependency versions.

## 1. What is Apache Kafka?

Apache Kafka is a distributed event streaming platform. It allows applications to publish, store, process, and consume streams of records.

Common enterprise uses:
- Publish product, order, payment, and inventory events.
- Integrate independently deployed services.
- Ingest application logs, audit events, and telemetry.
- Build search indexes and read models.
- Stream data into analytics platforms.
- Buffer workloads between producers and consumers.

Kafka is not simply a queue. Records are stored in partitioned, replicated logs and can be consumed by multiple independent consumer groups.

### Kafka vs traditional message queue

| Concept | Traditional queue (typical model) | Kafka |
|---|---|---|
| Consumption | A message is commonly removed/acknowledged from a queue | Records remain in the log according to retention policy |
| Multiple applications | Often requires separate queues/subscriptions | Independent consumer groups can read the same topic |
| Replay | Depends on broker/product | Consumers can reset offsets and replay retained records |
| Ordering | Often queue-level or product-specific | Ordering is guaranteed within a partition |
| Scaling | Product-specific competing consumers | Partition-based parallelism |
| Storage | Product-specific | Durable append-only partition logs |

Kafka can support queue-like and pub/sub-like patterns, but its storage and offset model is different.

## 2. Core Kafka concepts

### Event / record
A Kafka record typically contains:
- Key: optional; influences partition selection.
- Value: payload.
- Headers: metadata such as correlation ID or schema information.
- Timestamp: record timestamp.
- Offset: position within a partition.

Example event:

```json
{
  "eventId": "evt-9f31",
  "eventType": "ProductUpdated",
  "eventVersion": 1,
  "occurredAt": "2026-11-17T10:15:30Z",
  "productId": 501,
  "sku": "SKU-1001",
  "changedFields": ["name", "brandId"]
}
```

Use stable event IDs, explicit event types, schema versions, and timestamps. Avoid putting passwords, tokens, or unnecessary personal data in events.

### Topic
A topic is a named stream of records, such as:
- `product-events`
- `order-events`
- `payment-events`
- `inventory-events`

A topic is split into partitions.

### Partition
A partition is an ordered, append-only log. Each record has an offset within that partition.

<AsyncImage query="Apache Kafka topic partitions broker distributed log architecture diagram" aspectRatio="16:9" width="100%" />

Important:
- Ordering is guaranteed within one partition, not across an entire topic.
- More partitions allow more consumer parallelism, up to the number of active consumers in a group for that topic.
- Partition count affects key distribution, ordering, and operational planning.
- Increasing partitions can change key-to-partition mapping for future records with common partitioners; plan carefully.

### Offset
An offset identifies a record's position in a partition. Consumers track offsets to know where to resume.

An offset is not a global topic position. Each partition has its own offset sequence.

### Broker
A Kafka broker is a Kafka server. A Kafka cluster consists of multiple brokers.

### Replication
Partitions can have replicas on multiple brokers:
- Leader replica handles reads/writes for that partition (under normal configurations).
- Follower replicas replicate the leader.
- Replication factor defines how many replicas exist.
- ISR (in-sync replicas) are replicas sufficiently caught up according to Kafka's replication rules.

Replication improves availability and durability when configured correctly; it does not replace backups, disaster recovery, or operational discipline.

### Producer
A producer publishes records to a topic. It can choose a partition explicitly or use a partitioner, often based on the record key.

### Consumer
A consumer reads records and tracks progress using offsets.

### Consumer group
Consumers sharing the same `group.id` coordinate to divide partitions among themselves. In a group, a partition is assigned to at most one consumer at a time under the standard group model.

Different groups can independently read the same topic.

## 3. Kafka architecture and record flow

<AsyncImage query="Apache Kafka producer broker partition replication consumer group architecture diagram" aspectRatio="16:9" width="100%" />

Typical flow:
1. Producer serializes a record.
2. Producer selects a topic partition.
3. Broker leader appends the record to the partition log.
4. Followers replicate the record.
5. Producer receives an acknowledgement according to `acks`.
6. Consumer group fetches records.
7. Consumer processes records.
8. Consumer commits offsets according to its processing strategy.

Kafka stores bytes. Producers and consumers must agree on serialization and event schema.

## 4. Producer internals and configuration

### Key producer settings

| Setting | Purpose | Production consideration |
|---|---|---|
| `bootstrap.servers` | Initial broker addresses | Use multiple broker addresses |
| `key.serializer` | Key serialization | Must match key format |
| `value.serializer` | Value serialization | Use a governed schema |
| `acks` | Required acknowledgement level | `all` for stronger durability |
| `enable.idempotence` | Deduplicates producer retries within supported scope | Usually enable for reliable publishing |
| `retries` | Retry transient send failures | Coordinate with delivery timeout |
| `delivery.timeout.ms` | Upper bound for delivery attempt | Set against business latency budget |
| `request.timeout.ms` | Request timeout | Must be coherent with broker/network settings |
| `linger.ms` | Wait briefly to batch records | Trade latency for throughput |
| `batch.size` | Batch size per partition | Tune from measured workload |
| `compression.type` | Compression codec | Benchmark CPU vs bandwidth/storage |
| `max.in.flight.requests.per.connection` | Concurrent unacknowledged requests | Preserve ordering with idempotence-compatible settings |

### Acknowledgement modes
- `acks=0`: producer does not wait for broker acknowledgement.
- `acks=1`: leader acknowledges after local append.
- `acks=all`: leader waits for required in-sync replicas according to topic/broker settings.

For critical events, commonly combine:
- `acks=all`
- replication factor appropriate to failure model (often 3 in production clusters)
- `min.insync.replicas` set to a deliberate value (often 2 with RF=3)
- producer idempotence enabled

This configuration is not a universal guarantee. Understand unclean leader election, ISR behavior, rack placement, storage, and the actual cluster topology.

### Java producer example
```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
          StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
          StringSerializer.class.getName());
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
    ProducerRecord<String, String> record =
        new ProducerRecord<>("product-events", "SKU-1001", payload);

    RecordMetadata metadata = producer.send(record).get();
    System.out.printf("topic=%s partition=%d offset=%d%n",
        metadata.topic(), metadata.partition(), metadata.offset());
}
```

Calling `.get()` blocks the calling thread. In a high-throughput service, use asynchronous callbacks or a managed producer integration and handle failures deliberately.

### Partition key design
Choose a key based on the ordering boundary required by the business:
- `orderId`: preserve order of events for one order.
- `productId` or `sku`: preserve order of product updates.
- `customerId`: preserve per-customer sequence where required.

Avoid:
- Random keys when per-entity ordering is required.
- A single constant key, which sends all records to one partition and creates a hot partition.
- High-cardinality or sensitive keys without considering privacy and partitioning effects.

## 5. Consumer internals and offset management

### Consumer configuration
Important settings:
- `bootstrap.servers`
- `group.id`
- `key.deserializer`
- `value.deserializer`
- `enable.auto.commit`
- `auto.offset.reset`
- `max.poll.records`
- `max.poll.interval.ms`
- `session.timeout.ms`
- `heartbeat.interval.ms`
- `isolation.level`

### `auto.offset.reset`
Used when there is no valid committed offset:
- `earliest`: start at the earliest retained record.
- `latest`: start at the end and receive new records.
- `none`: fail if no valid offset exists.

It does not reset an existing valid committed offset.

### Auto commit vs manual commit
Auto commit can commit offsets independently of completion of business processing. If the application crashes after a commit but before processing finishes, records may be skipped.

For critical processing, control offset commits so they occur only after successful processing and the required durable side effects.

### Manual commit example (plain Java, simplified)
```java
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("product-events"));

try {
    while (true) {
        ConsumerRecords<String, String> records =
            consumer.poll(Duration.ofMillis(500));

        for (ConsumerRecord<String, String> record : records) {
            processIdempotently(record);
        }

        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

This simplified example commits after the whole polled batch. A production design must handle partial failures, retries, rebalance callbacks, processing duration, and shutdown correctly.

### At-least-once processing
A common reliable approach:
1. Read records.
2. Apply the business effect in a database transaction.
3. Commit the consumed offset only after the database transaction succeeds.
4. If the process crashes between DB commit and offset commit, the record may be replayed.
5. Use idempotency/deduplication so replay does not duplicate the business effect.

### At-most-once processing
Commit before processing. A crash after commit but before the business effect can lose processing. Use only where occasional loss is acceptable.

### Exactly-once semantics
Kafka supports transactional producer workflows and exactly-once processing within supported Kafka-to-Kafka transactional boundaries. This does not automatically make an arbitrary database update, HTTP call, email, or payment exactly once.

For external databases/services, use idempotency, transactional outbox/inbox, or another explicit coordination design.

## 6. Consumer groups and rebalancing

A consumer group distributes topic partitions among group members.

Example:
- Topic has 6 partitions.
- Group has 3 consumers.
- Each consumer can receive 2 partitions.
- Add a fourth consumer: up to 4 consumers can receive work, but some partitions are reassigned.
- Add 8 consumers: 2 consumers will be idle because only 6 partitions exist.

### Rebalancing
A rebalance occurs when group membership or partition assignment changes, for example:
- Consumer joins or leaves.
- Consumer crashes.
- A consumer misses heartbeats or fails to poll within configured limits.
- Topic partition count changes.

Rebalances temporarily interrupt normal consumption and can affect latency.

### Rebalance strategies
Kafka clients support assignment strategies such as range, round-robin, and cooperative-sticky (availability depends on client version/configuration). Cooperative rebalancing can reduce the amount of partition movement.

### Common causes of repeated rebalances
- Long processing inside the poll loop.
- Blocking downstream HTTP/DB calls.
- Too-small `max.poll.interval.ms`.
- Slow GC pauses or CPU starvation.
- Unstable deployments or frequent restarts.
- Network instability.

Mitigations:
- Keep poll loop responsive.
- Tune `max.poll.records` to processing capacity.
- Use bounded worker pools and pause/resume partitions when needed.
- Keep processing time within the poll interval.
- Monitor rebalance rate and consumer lag.
- Use graceful shutdown and cooperative assignment where suitable.

## 7. Spring Boot and Spring Kafka

### Dependencies
Use Spring Boot dependency management and the matching Spring Kafka version.

Maven starter commonly used:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

For tests, include the corresponding Spring Kafka test dependency when needed.

### Configuration example
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: catalog-consumer
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: com.example.events
    listener:
      ack-mode: record
```

Serializer/deserializer configuration is version-sensitive. In production, use explicit trusted packages/type mappings or a schema-based serialization approach; do not broadly trust arbitrary deserialized Java types.

### Event DTO
```java
public record ProductUpdatedEvent(
    String eventId,
    int eventVersion,
    Long productId,
    String sku,
    Instant occurredAt
) {}
```

### Producer with KafkaTemplate
```java
@Service
class ProductEventPublisher {
    private final KafkaTemplate<String, ProductUpdatedEvent> kafkaTemplate;

    ProductEventPublisher(
            KafkaTemplate<String, ProductUpdatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    CompletableFuture<SendResult<String, ProductUpdatedEvent>> publish(
            ProductUpdatedEvent event) {
        return kafkaTemplate.send(
            "product-events", event.sku(), event);
    }
}
```

Handle the returned future. Log topic/partition/offset on success and route failures to observable retry/error handling. Do not silently discard send failures.

### Consumer with `@KafkaListener`
```java
@Component
class ProductEventListener {
    @KafkaListener(
        topics = "product-events",
        groupId = "search-indexer")
    public void consume(ProductUpdatedEvent event) {
        // Validate event and update search projection idempotently.
    }
}
```

The listener method's successful return is not necessarily equivalent to a committed business transaction in every configuration. Understand the container's acknowledgement mode, error handler, transaction setup, and offset commit behavior.

### Topic configuration
```java
@Configuration
class KafkaTopicConfig {
    @Bean
    NewTopic productEventsTopic() {
        return TopicBuilder.name("product-events")
            .partitions(6)
            .replicas(3)
            .build();
    }
}
```

Creating topics from application startup may not be appropriate in every enterprise environment. Many organizations manage topics through platform/IaC processes and restrict application permissions.

## 8. Serialization and schema evolution

### Serialization choices
- JSON: human-readable and easy to inspect; larger payloads and looser type governance.
- Avro: compact, schema-driven, often used with a schema registry.
- Protobuf: compact typed schema format with broad language support.
- JSON Schema: schema validation while retaining JSON representation.

### Schema evolution rules
- Add optional fields with sensible defaults.
- Do not change the meaning or type of an existing field casually.
- Do not remove fields until all consumers have migrated.
- Define compatibility mode and enforce it in CI.
- Include event version and ownership.
- Test old producer/new consumer and new producer/old consumer combinations where required.

### Event envelope
A useful envelope can include:
- `eventId`
- `eventType`
- `eventVersion`
- `occurredAt`
- `producer`
- `correlationId`
- `aggregateId`
- `aggregateVersion`
- `payload`

Do not add metadata without a clear governance purpose. Ensure timestamps and identifiers have consistent semantics.

## 9. Error handling, retries, and dead-letter topics

### Failure categories
1. Transient: temporary network issue, broker unavailable, rate limit.
2. Permanent: invalid payload, unknown product, schema violation.
3. Business rejection: valid message but business rule prevents action.
4. Poison message: repeatedly fails and blocks progress.
5. Infrastructure failure: database outage, authentication issue, disk pressure.

Different categories need different handling.

### Retry approaches
- Short in-memory retry for brief transient failures.
- Retry topics with delay/backoff for longer delays.
- Dead-letter topic (DLT) for records requiring separate inspection or remediation.
- Manual replay after correction.

Avoid retrying forever in the main listener. It can block a partition and delay later records.

### Spring Kafka error handling (illustrative)
Spring Kafka provides listener error handlers and recoverers, including `DefaultErrorHandler` and dead-letter publishing integrations. Exact constructors and configuration vary by version.

Conceptual configuration:
```java
@Bean
DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer recoverer) {
    FixedBackOff backOff = new FixedBackOff(1000L, 3L);
    return new DefaultErrorHandler(recoverer, backOff);
}
```

Configure which exceptions are retryable, DLT naming, serialization of failed records, and operational alerting. DLT is not a garbage bin: include reason metadata, original topic/partition/offset, exception details (safe and bounded), and replay ownership.

## 10. Ordering, partitioning, and hot partitions

Kafka preserves order within one partition. It does not guarantee ordering across partitions.

If product events for the same SKU must be ordered, publish with SKU as key. Then all records for that key are routed to the same partition by the configured partitioner.

### Hot partition
If one key receives a disproportionate volume, its partition becomes a bottleneck.

Mitigations:
- Reconsider the required ordering boundary.
- Split a hot entity into subkeys only if ordering semantics permit.
- Increase partitions when useful for overall distribution.
- Use separate topics for distinct workloads.
- Monitor per-partition throughput and lag.

Do not increase partitions blindly: it can change key partition mapping and affect ordering assumptions.

## 11. Kafka transactions and exactly-once processing

Kafka producer transactions can atomically write to multiple Kafka partitions and commit consumed offsets as part of a Kafka-to-Kafka processing workflow.

Conceptual consume-transform-produce:
1. Consumer reads input records.
2. Producer writes transformed output records in a transaction.
3. Consumed offsets are added to the same transaction.
4. Transaction commits.
5. Downstream consumers using `read_committed` see committed output only.

Use `isolation.level=read_committed` for consumers that must not see aborted transactional records.

For DB + Kafka:
- Use transactional outbox for DB update plus event intent.
- Use idempotent consumers/inbox for DB side effects.
- Do not claim end-to-end exactly-once just because Kafka producer transactions are enabled.

## 12. Kafka Connect, Streams, and ecosystem

### Kafka Connect
A framework for moving data between Kafka and external systems using source and sink connectors.

Examples:
- Database change data capture into Kafka.
- Kafka topic to object storage.
- Kafka topic to search/database sink.

Operational concerns: connector configuration, offsets, schema compatibility, task failures, credentials, and replay.

### Change Data Capture (CDC)
CDC captures database changes, often from transaction logs, and publishes them as events. It can help integrate legacy databases or publish changes without adding event publication code to every write path.

CDC is not automatically a replacement for domain events: raw row changes may not express business meaning.

### Kafka Streams
A Java library for stream processing:
- Filter, map, group, aggregate, join.
- Stateful processing with local state stores.
- Windowing and event-time processing.
- Interactive queries (depending on design).

Use Kafka Streams for continuous event processing when its state and deployment model fit the use case. For complex analytics, evaluate Flink or other stream-processing platforms.

## 13. Real-world case study A — Product Information Management (PIM)

### Business context
A retail platform manages product master data, SKUs, attributes, categories, brands, supplier imports, and channel publishing. Product changes must reach search and downstream sales channels.

### Event flow
```text
React UI
   |
Product Catalog API
   |
Product DB + Outbox (one local transaction)
   |
Outbox Publisher
   |
Kafka: product-events
   |                 |
   v                 v
Search Indexer   Publishing Service
   |                 |
Search Store     Channel Adapters
```

### Event examples
- `ProductCreated`
- `ProductUpdated`
- `ProductApproved`
- `ProductPublished`
- `ProductDiscontinued`

### Key design choices
- Use SKU/product ID as key if per-product event order matters.
- Include event ID and aggregate version.
- Publish through outbox.
- Search consumer updates a projection idempotently.
- Publishing consumer checks channel eligibility.
- Keep supplier-specific payloads behind an anti-corruption adapter.
- Use retry/DLT for temporary or invalid downstream events.
- Monitor event age and consumer lag.

### Failure scenarios
| Failure | Expected behavior |
|---|---|
| Kafka unavailable | Product DB transaction still records outbox event; publisher retries |
| Duplicate event | Consumer deduplicates by event ID |
| Old event arrives late | Consumer checks aggregate version and avoids stale overwrite |
| Search service unavailable | Consumer pauses/retries or routes failures safely; lag rises visibly |
| Channel endpoint unavailable | Bounded retry, then DLT/operational workflow |
| Schema changed | Compatibility test prevents unsafe rollout |

## 14. Real-world case study B — Order, payment, and inventory

### Workflow
1. Order Service creates order with `PENDING` state and idempotency key.
2. Order Service publishes `OrderCreated`.
3. Inventory Service reserves stock and publishes `InventoryReserved` or `InventoryRejected`.
4. Payment Service authorizes payment and publishes result.
5. Order workflow confirms or fails the order.
6. On payment failure, inventory reservation is released.
7. Reconciliation handles uncertain external payment outcomes.

### Important rules
- Order events keyed by order ID.
- Inventory reservation must be atomic within Inventory Service.
- Payment API calls need idempotency keys.
- Saga state must be persisted.
- Compensation is explicit (release reservation/refund).
- Never infer payment failure only from an HTTP timeout.
- Monitor time spent in each workflow state.

## 15. Real-world case study C — Supplier bulk import

### Workflow
1. Supplier file uploaded to object storage.
2. Import Service stores job metadata.
3. Import Service emits `ImportRequested`.
4. Worker consumes job and processes bounded chunks.
5. Row-level validation results are persisted.
6. Valid product updates go through Catalog APIs or a controlled internal command path.
7. Import job publishes progress and completion events.
8. Failed rows are downloadable for correction.

Kafka is useful for distributing jobs, buffering load, and communicating progress. It is not a substitute for durable import-job state, file storage, or a batch-processing design.

### Important safeguards
- Do not send a huge file as one Kafka record.
- Put file content in object storage; publish a reference and metadata.
- Limit concurrent jobs and consumer work.
- Make row processing idempotent.
- Track job ID, row number, retry count, and error code.
- Apply back-pressure and monitor lag.

## 16. Real-world case study D — Database CDC and analytics

A legacy product database may need to feed a data platform:
1. CDC connector reads database log changes.
2. Connector publishes change records to Kafka.
3. Stream processor validates/transforms/enriches.
4. Sink connector writes to object storage or analytics store.

Distinguish:
- **CDC event:** a row changed.
- **Domain event:** a business fact occurred.

If a downstream consumer needs business meaning, map raw CDC changes into governed domain events or a curated data contract.

## 17. Kafka performance and capacity planning

### Throughput depends on
- Record size and batch size.
- Partition count and distribution.
- Producer batching and compression.
- Broker CPU, disk, network, and page cache.
- Replication factor and acknowledgement settings.
- Consumer processing time and downstream capacity.
- Storage retention and segment configuration.

### Practical tuning sequence
1. Establish baseline throughput, latency, and record size.
2. Measure producer send latency and broker request metrics.
3. Check partition balance and hot keys.
4. Check consumer lag and processing time.
5. Check disk/network/CPU and replication health.
6. Tune batch/linger/compression based on measurements.
7. Scale partitions/consumers only after identifying the bottleneck.
8. Re-run load tests and verify tail latency and failure recovery.

### Consumer lag
Lag is the gap between the latest available offset and the consumer group's committed/current position, measured per partition.

Lag can indicate:
- Consumer processing too slowly.
- Insufficient consumer parallelism.
- Downstream DB/API bottleneck.
- Consumer is down or stuck.
- Traffic spike.

Lag alone does not explain the cause. Also monitor record age, processing latency, failure rate, and per-partition distribution.

### Retention and compaction
- Time/size retention removes older log segments according to policy.
- Log compaction retains the latest value for each key over time (subject to cleanup semantics), useful for changelog/state topics.
- Compaction is not an immediate delete and does not mean one record per key at every instant.
- Retention must cover expected outages, replay windows, and recovery requirements.

## 18. Security and governance

Production controls:
- TLS for broker connections.
- SASL authentication or platform identity integration.
- ACLs for topic/group/transactional ID access.
- Least privilege per producer/consumer identity.
- Secret rotation.
- Network segmentation.
- Encryption at rest where required.
- Schema governance and compatibility checks.
- PII classification, minimization, retention, and deletion obligations.
- Audit of administrative operations.

Do not give every service cluster-wide read/write access. Grant access only to required topics and consumer groups.

## 19. Monitoring and operational runbook

### Broker/cluster signals
- Under-replicated partitions.
- Offline partitions.
- ISR shrink/expand events.
- Request latency and error rate.
- Disk usage and disk I/O.
- Network throughput.
- Controller/quorum health according to Kafka deployment mode.
- Broker availability and log directory status.

### Producer signals
- Record send rate.
- Error/retry rate.
- Request latency.
- Batch size and compression ratio.
- Buffer pool wait time.
- Failed deliveries.

### Consumer signals
- Lag and oldest record age.
- Records consumed per second.
- Processing duration.
- Commit failures.
- Rebalance frequency.
- DLT rate.
- Downstream DB/API latency.

### Incident checklist: consumer lag increasing
1. Is the consumer group healthy and members stable?
2. Are partitions evenly assigned?
3. Did traffic or record size increase?
4. Is processing slower due to DB/API dependency?
5. Are retries or poison messages blocking partitions?
6. Are there repeated rebalances or long GC pauses?
7. Is the number of partitions sufficient for desired parallelism?
8. Can the downstream dependency handle more consumers?
9. Is the lag old but stable, or actively growing?
10. After recovery, can the system drain backlog without harming live traffic?

### Incident checklist: producer errors
1. Check broker availability and network path.
2. Check authentication, ACLs, and TLS certificates.
3. Check topic existence and authorization.
4. Check `acks`, ISR, and `min.insync.replicas`.
5. Check request/delivery timeouts.
6. Check producer buffer exhaustion and broker throttling.
7. Check payload size and serialization failures.

## 20. Common Kafka anti-patterns

1. One partition for a high-volume topic.
2. Too many partitions without a capacity/operations plan.
3. Random keys when per-entity ordering matters.
4. One constant key that creates a hot partition.
5. Auto-commit without understanding processing guarantees.
6. Committing offsets before durable business processing.
7. Infinite retries in the main consumer path.
8. No DLT ownership or replay process.
9. Publishing DB changes directly without outbox/integration design.
10. Assuming Kafka transactions make external DB/API effects exactly once.
11. Using Kafka as a synchronous request-response mechanism for every operation.
12. Putting large files/blobs directly into records.
13. No schema compatibility policy.
14. Logging complete payloads containing sensitive data.
15. Scaling consumers beyond partition count and expecting more parallelism.
16. Ignoring consumer lag and record age.
17. Treating raw CDC row changes as business events without semantic mapping.

## 21. Testing Kafka applications

Test at several levels:
- Unit tests for mapping, validation, and business logic.
- Producer tests for key, topic, headers, and payload.
- Consumer tests for idempotency and error classification.
- Integration tests with a real Kafka broker using Testcontainers or an approved test environment.
- Contract/schema compatibility tests.
- Failure tests: broker unavailable, duplicate event, invalid payload, DB failure, rebalance, and retry/DLT.
- Load tests with realistic key distribution and message sizes.

Avoid testing only with mocks. Broker behavior, serialization, partitions, offsets, and rebalances need integration coverage.

## 22. Interview questions

### Fundamentals
1. What is Kafka and how is it different from a traditional queue?
2. What is a topic and partition?
3. What is an offset?
4. What is a consumer group?
5. How does Kafka preserve ordering?
6. What is a broker and controller?
7. What is replication factor?
8. What are leader, follower, and ISR?
9. What is retention vs compaction?
10. What happens when a consumer joins or leaves a group?

### Producer and consumer
11. Explain `acks=0`, `acks=1`, and `acks=all`.
12. What does producer idempotence do?
13. How do keys affect partition selection?
14. What is a hot partition?
15. Auto commit vs manual commit?
16. `earliest` vs `latest`?
17. What causes consumer lag?
18. What causes frequent rebalances?
19. How do you safely process records in parallel?
20. What happens when consumer processing takes longer than `max.poll.interval.ms`?

### Reliability
21. At-most-once vs at-least-once vs exactly-once?
22. What does Kafka exactly-once actually cover?
23. How do you make a DB-writing consumer idempotent?
24. Explain transactional outbox.
25. What is a DLT and how do you replay records safely?
26. How do you handle poison messages?
27. How do you preserve per-order event ordering?
28. How do you handle out-of-order events?

### Enterprise design
29. Design product update propagation to search and channel systems.
30. Design an order/inventory/payment workflow using Kafka.
31. How would you process a 5 GB supplier file?
32. How do you evolve event schemas without breaking consumers?
33. Kafka vs REST for service integration?
34. Kafka Connect vs Kafka Streams?
35. CDC vs domain events?
36. How would you size partitions and consumer instances?
37. How do you secure topics?
38. What metrics and alerts are essential in production?

## 23. Hands-on exercises

### Core
1. Start a local Kafka cluster using an approved local environment.
2. Create a topic with multiple partitions and replication appropriate to the environment.
3. Produce records with and without keys.
4. Inspect partition assignment and offsets.
5. Run two consumers in one group and observe partition distribution.
6. Run a second group and verify independent consumption.
7. Stop one consumer and observe rebalancing.
8. Reset offsets in a test group and replay retained records.

### Java / Spring Kafka
9. Build a Spring Boot producer for `ProductUpdated`.
10. Build a consumer that updates a search projection.
11. Add headers for correlation ID and event version.
12. Configure manual/record acknowledgement and verify commit behavior.
13. Add retry and DLT handling.
14. Add an idempotency table and test duplicate delivery.
15. Implement outbox publishing.
16. Test schema compatibility.

### Enterprise scenarios
17. Build a product publication event pipeline.
18. Build an order saga with inventory reservation and payment authorization simulation.
19. Build an import job workflow using object-storage references.
20. Simulate a downstream DB outage and measure lag/recovery.
21. Simulate a poison message and replay after correction.
22. Load-test hot-key and evenly distributed-key scenarios.
23. Add metrics and alerts for lag, DLT rate, and producer errors.
24. Write an operational runbook for a consumer group.

## 24. Suggested 30-day learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | Topics, partitions, offsets, brokers, replication | Local cluster experiments |
| 4–6 | Producer internals, keys, batching, acknowledgements | Java producer |
| 7–9 | Consumers, groups, offsets, rebalancing | Consumer group experiments |
| 10–12 | Spring Kafka, serializers, listeners | Spring Boot producer/consumer |
| 13–15 | Retry, DLT, idempotency | Reliable consumer |
| 16–18 | Outbox, transactions, event consistency | DB-to-Kafka publication |
| 19–21 | Ordering, schema evolution, CDC | Governed event pipeline |
| 22–24 | Kafka Connect, Streams, security | Integration/stream exercise |
| 25–27 | Monitoring, capacity, incident response | Dashboard and runbook |
| 28–30 | PIM/order capstone and interview review | End-to-end project |

## 25. Capstone — Enterprise Product Event Platform

### Build
- Product Catalog API using Spring Boot and JPA.
- Product database and outbox table.
- Outbox publisher to Kafka.
- Search Indexer consumer.
- Publishing consumer.
- Retry and DLT topics.
- Idempotency/inbox table.
- Schema compatibility checks.
- Actuator/Micrometer metrics and distributed tracing.
- Docker-based local environment and CI tests.

### Acceptance criteria
- Product DB update and outbox insert are atomic.
- Publisher retries without losing event intent.
- Consumers tolerate duplicate events.
- Per-product ordering is preserved using a stable key and aggregate version.
- Old events cannot overwrite newer search state.
- Invalid messages are routed to a monitored DLT.
- Consumer lag and oldest event age are visible.
- Secrets and topic permissions follow least privilege.
- Integration tests cover broker and database failure paths.

---

## Official documentation
- Apache Kafka documentation: https://kafka.apache.org/documentation/
- Kafka producer configuration: https://kafka.apache.org/documentation/#producerconfigs
- Kafka consumer configuration: https://kafka.apache.org/documentation/#consumerconfigs
- Spring for Apache Kafka: https://docs.spring.io/spring-kafka/reference/
- Spring Boot reference: https://docs.spring.io/spring-boot/reference/
- Testcontainers Kafka module: https://java.testcontainers.org/modules/kafka/
