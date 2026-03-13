# Kafka Internals

## What It Is
Distributed commit log — append-only, ordered, persistent. Not a traditional queue: messages aren't deleted after consumption, consumers can replay.

## Core Architecture
- **Topic** = logical channel (e.g., "order-events")
- **Partition** = ordered append-only log within topic. Unit of parallelism.
- **Broker** = server storing partitions
- Each partition has 1 leader (reads/writes) + N-1 follower replicas (failover)

## Why It's Fast: 3 Tricks

### 1. Sequential Writes
Append to end of file → sequential I/O → ~600MB/s HDD, ~2GB/s SSD. Faster than random memory access.

### 2. Zero-Copy (sendfile syscall)
Normal: Disk → kernel buf → app buf → socket buf → NIC (4 copies)
Zero-copy: Disk → kernel buf → NIC (2 copies, no CPU involvement)

### 3. Batching + Compression
1000 messages = 1 network round trip. Compressed (gzip/snappy/lz4) in batch.

## Partitions
- With key: `hash(key) % num_partitions` → same key = same partition = ordered per key
- Without key: round-robin → even distribution, no ordering
- Max useful consumers in a group = number of partitions

## Segment Files (On-Disk Storage)
```
Partition 0/
  ├── 00000000000000000000.log       (actual messages)
  ├── 00000000000000000000.index     (offset → byte position, sparse)
  └── 00000000000000000000.timeindex (timestamp → offset)
```
- Only last segment actively written (append-only)
- Old segments immutable → safe reads without locking
- Retention deletes whole segments (efficient)
- Lookup by offset: binary search .index → seek .log → O(log N)

## Consumer Groups
- Each partition → exactly 1 consumer within a group
- Multiple groups → each gets all messages (pub/sub between groups, queue within group)
- Consumer dies → partitions rebalanced to survivors
- Consumers > partitions → some idle

## Offsets
- Consumer tracks position per partition in `__consumer_offsets` topic
- Commit after processing → at-least-once (crash before commit = redelivery)
- Reset offset to 0 → replay all messages (impossible with traditional queues)

## Retention
- Time-based: `retention.ms = 604800000` (7 days default)
- Log compaction: keep only latest value per key forever (for CDC, config storage)

## Exactly-Once Semantics
- **Idempotent producer:** PID + sequence number → broker deduplicates retries
- **Transactions:** atomic read-process-write across topics + consumer offsets

## Replication & Durability
- ISR (In-Sync Replicas): followers caught up with leader
- `acks=0` → don't wait (fastest, may lose)
- `acks=1` → leader ACK (fast, lose if leader dies pre-replication)
- `acks=all` → all ISR ACK (slowest, no data loss)

## ZooKeeper vs KRaft
- Old: ZooKeeper (separate cluster) for metadata, leader election
- New: KRaft (Kafka Raft) — metadata in internal topic, no external dependency
- KRaft production-ready since 3.3, ZooKeeper removed in 4.0

## Consumer Rebalancing
- Eager (old): stop-the-world, revoke all, reassign all
- Cooperative (new): only affected partitions revoke, others keep processing
- Sticky assignor: minimizes partition movement

## CDC (Change Data Capture)
React to DB changes without code changes. Debezium reads PostgreSQL WAL → Kafka topic.
Use cases: search sync, cache invalidation, data warehouse, cross-service sync, audit logs.

## Stream Processing (Kafka Streams)
- Java library (no separate cluster), runs in your app
- KStream: event stream (all events)
- KTable: changelog stream (latest value per key, like materialized view)
- KStream + KTable join: enrich events with lookup data

## Real-World Usage
| Pattern | Example |
|---------|---------|
| Event sourcing | Order lifecycle events → rebuild state by replay |
| CDC | PostgreSQL → Debezium → Kafka → Elasticsearch/Redis/DW |
| Microservice decoupling | Order Service emits → Payment/Email/Analytics consume |
| Metrics pipeline | All servers → Kafka → ELK/S3/Alerting |

## Code Integration

### Producer (Spring Boot)
```java
@Autowired KafkaTemplate<String, OrderEvent> kafkaTemplate;
kafkaTemplate.send("order-events", order.getUserId(), event);
```

### Consumer (Spring Boot)
```java
@KafkaListener(topics = "order-events", groupId = "payment-service")
public void handle(OrderEvent event, Acknowledgment ack) {
    paymentService.charge(event.getOrder());
    ack.acknowledge(); // manual commit
}
```

### Key pattern
Producer declares: broker address, serializer. Sends to topic.
Consumer declares: broker address, group-id, which topics to subscribe.
Producer doesn't know consumers. Consumer subscribes independently. That's the decoupling.

## When to Use / Not Use
| Use Kafka | Don't use Kafka |
|-----------|----------------|
| High throughput (100K+ msg/sec) | Low volume task queue |
| Need replay / event sourcing | Complex routing (use RabbitMQ) |
| Multiple consumer groups | Simple request-reply |
| Stream processing | Priority ordering |
