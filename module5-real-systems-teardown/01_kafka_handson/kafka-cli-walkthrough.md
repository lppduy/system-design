# Kafka Hands-On: CLI Walkthrough

Docker-based, single broker, KRaft mode. All commands tested on apache/kafka:latest (4.2.0).

---

## Start Kafka (KRaft, Single Broker)

```bash
docker run -d --name kafka -p 9092:9092 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e CLUSTER_ID=MkU3OEVBNTcwNTJENDM2Qk \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  apache/kafka:latest
```

**Gotcha:** Without `OFFSETS_TOPIC_REPLICATION_FACTOR=1`, consumers fail with `TimeoutException` because `__consumer_offsets` topic defaults to replication-factor 3 but only 1 broker exists.

**Remote VPS:** Change `ADVERTISED_LISTENERS` to VPS public IP. Clients use this address to connect — if it says `localhost`, remote clients try to connect to themselves.

## Create Topic

```bash
docker exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order-events --partitions 3 --replication-factor 1
```

- Partition count = max parallel consumers in one group
- Replication factor = total copies per partition (1 = no failover)

## Produce with Keys

```bash
echo -e "user-123:order created\nuser-123:payment received\nuser-456:order created" | \
  docker exec -i kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 --topic order-events \
  --reader-property parse.key=true --reader-property key.separator=:
```

Same key = same partition = ordering guaranteed per key. `hash(key) % num_partitions`.

**Why it matters:** If order events for same user land in different partitions, payment service might process `payment received` before `order created` — race condition.

## Consume with Partition Info

```bash
docker exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic order-events --from-beginning \
  --formatter-property print.key=true \
  --formatter-property print.partition=true \
  --timeout-ms 10000
```

## Consumer Groups & Lag

```bash
# Consume in a named group
docker exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic order-events \
  --group payment-service --from-beginning --timeout-ms 10000

# Check lag (THE #1 production metric)
docker exec kafka /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --describe --group payment-service
```

Output: CURRENT-OFFSET, LOG-END-OFFSET, LAG per partition.
LAG > 0 and growing = consumer falling behind = alert.

Rules:
- Each partition -> exactly 1 consumer within a group
- Multiple groups -> each gets ALL messages (pub/sub between groups, queue within group)
- Consumers > partitions -> some idle

## Offset Reset (Replay)

```bash
# Dry-run first
docker exec kafka /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --group payment-service \
  --topic order-events --reset-offsets --to-earliest --dry-run

# Execute (consumers must be stopped)
docker exec kafka /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --group payment-service \
  --topic order-events --reset-offsets --to-earliest --execute
```

Use cases: bug in consumer -> fix -> replay. New service needs historical data. Rebuild search index.

## Inspect Segment Files

```bash
# List files
docker exec kafka ls /tmp/kafka-logs/order-events-0/

# Dump raw messages
docker exec kafka /opt/kafka/bin/kafka-dump-log.sh \
  --files /tmp/kafka-logs/order-events-0/00000000000000000000.log \
  --print-data-log
```

Each partition directory contains:
- `.log` — actual messages (append-only)
- `.index` — offset -> byte position (sparse, for fast seeking)
- `.timeindex` — timestamp -> offset

Messages are batched on disk. Each record has: offset, timestamp, key, payload, producerId + sequence (for idempotent dedup).

## Perf Test

```bash
# Producer: 100K messages, 256 bytes each
docker exec kafka /opt/kafka/bin/kafka-producer-perf-test.sh \
  --topic order-events --num-records 100000 --record-size 256 \
  --throughput -1 --command-property bootstrap.servers=localhost:9092

# Consumer
docker exec kafka /opt/kafka/bin/kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 --topic order-events --num-records 100000
```

Single Docker broker result: ~138K msg/sec produce, ~30K msg/sec consume (775K after rebalance).

## Quick Reference

| Command | What |
|---------|------|
| `kafka-topics.sh --create` | Create topic |
| `kafka-topics.sh --describe` | Partitions, leaders, ISR |
| `kafka-console-producer.sh` | Send from terminal |
| `kafka-console-consumer.sh` | Read from terminal |
| `kafka-consumer-groups.sh --describe` | Lag, offsets, assignments |
| `kafka-consumer-groups.sh --reset-offsets` | Replay messages |
| `kafka-dump-log.sh` | Inspect raw segment files |
| `kafka-producer-perf-test.sh` | Benchmark produce |
| `kafka-consumer-perf-test.sh` | Benchmark consume |
| `kafka-get-offsets.sh` | Check offsets per partition |

## Key Lessons Learned

1. **Single-broker gotcha:** Internal topics (`__consumer_offsets`) default to replication-factor 3 — must override for dev
2. **Key selection** is the most important design decision — wrong key = race conditions
3. **LAG** is the #1 metric — consumer falling behind = trouble
4. **Offset reset** = replay — Kafka's killer feature vs traditional queues
5. **Segment files** are just append-only logs — simple, fast, inspectable
6. **Consumer groups** give you both pub/sub (between groups) and queue (within group)
7. **Batching** happens automatically — fewer I/O ops = high throughput
