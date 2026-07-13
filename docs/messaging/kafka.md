# Kafka

Reach for this when producing/consuming events, sizing partitions, reasoning about
delivery guarantees, or debugging consumer lag.

## Mental model

- **Topic** → named log, split into **partitions**. Ordering is guaranteed **only within a partition**.
- **Partition** → append-only, ordered sequence. Each message has an **offset**.
- **Producer** → picks a partition (by key hash, round-robin, or explicit).
- **Consumer group** → set of consumers sharing a `group.id`. Each partition is read by **exactly one consumer in the group**. Add consumers to scale - up to the partition count, then extra consumers sit idle.
- **Offset** → per group, per partition, committed to the internal `__consumer_offsets` topic.
- **Replication factor** → copies of a partition across brokers. RF=3 is the standard. One **leader** handles reads/writes; **followers** replicate. **ISR** = in-sync replicas.

Key = same key always lands on the same partition = ordering per key. No key = round-robin, no ordering.

## CLI (topics)

```bash
# Create a topic
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 6 --replication-factor 3

# Describe (partitions, leaders, ISR)
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders

# List / delete
kafka-topics.sh --bootstrap-server localhost:9092 --list
kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic orders

# Add partitions (you can only ever increase, never decrease)
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic orders --partitions 12
```

## CLI (produce / consume)

```bash
# Produce (type messages, Ctrl-D to end); key:value form
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders \
  --property parse.key=true --property key.separator=:

# Consume from the beginning, show keys
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders \
  --from-beginning --property print.key=true --property print.timestamp=true

# Consume as part of a group
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders \
  --group order-processor
```

## CLI (consumer groups & lag)

```bash
# THE command for debugging lag - LAG column = unread messages per partition
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processor

# List groups
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Reset offsets (must use --execute; --dry-run to preview). Group must be idle.
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group order-processor \
  --topic orders --reset-offsets --to-earliest --execute
# other targets: --to-latest  --to-offset N  --shift-by -100  --to-datetime <ISO>
```

## Delivery guarantees

| Guarantee | How | Cost |
|---|---|---|
| **At-most-once** | Commit offset *before* processing | Fast, can lose messages |
| **At-least-once** | Commit offset *after* processing | Default; duplicates on retry → make consumers idempotent |
| **Exactly-once** | Idempotent producer + transactions | `enable.idempotence=true`, `transactional.id`, read `isolation.level=read_committed` |

At-least-once + idempotent consumer is what you want 95% of the time.

## Producer knobs

```properties
acks=all                 # wait for all ISR - durability. (0=fire&forget, 1=leader only)
enable.idempotence=true  # no dupes on retry; sets acks=all, retries=MAX safely
compression.type=zstd    # zstd/lz4/snappy - big throughput/storage win
linger.ms=10             # wait a hair to batch → higher throughput
batch.size=32768         # bytes per partition batch
max.in.flight.requests.per.connection=5   # keep ≤5 with idempotence to preserve order
```

## Consumer knobs

```properties
group.id=order-processor
enable.auto.commit=false           # commit manually AFTER processing (at-least-once)
auto.offset.reset=earliest         # no committed offset → start from oldest (or 'latest')
max.poll.records=500               # batch size per poll()
max.poll.interval.ms=300000        # exceed this → kicked from group (rebalance)
isolation.level=read_committed     # only see committed txn messages
```

## Spring Boot (per your stack)

```java
@KafkaListener(topics = "orders", groupId = "order-processor")
public void handle(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
    process(record.value());
    ack.acknowledge();   // manual commit - requires AckMode.MANUAL
}
```
```properties
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.listener.ack-mode=manual
spring.kafka.producer.properties.enable.idempotence=true
```

## Gotchas / things I always forget

- **Ordering is per-partition, not per-topic.** Need global order? One partition (kills parallelism).
- You can **increase** partitions but never decrease. Adding partitions **changes key→partition mapping** → breaks per-key ordering for existing keys.
- More consumers than partitions = idle consumers. Partition count caps consumer parallelism.
- A slow consumer that blows past `max.poll.interval.ms` triggers a **rebalance** - everyone stops. Do slow work off the poll thread.
- Rebalances pause the whole group. Frequent rebalances = look at `session.timeout.ms` / heartbeat / long processing.
- Consumer **lag** is the health metric. Rising lag = consumers can't keep up → add consumers (up to partition count) or speed up processing.
- `acks=1` can lose data if the leader dies before replicas catch up. Use `acks=all` for anything important.
- Retention is time- **and** size-based (`retention.ms`, `retention.bytes`) - whichever hits first. Kafka is not a queue that empties on read; messages stay until retention expires.

## Quick reference

| Thing | Value / command |
|---|---|
| Default port | 9092 |
| Lag check | `kafka-consumer-groups.sh --describe --group G` |
| Ordering scope | single partition |
| Standard RF | 3 |
| Consumers per partition (per group) | exactly 1 |
| Idempotent producer | `enable.idempotence=true` |
| Manual commit | `enable.auto.commit=false` + commit after processing |
