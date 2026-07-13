# System Design - Building Blocks

Reach for this in a design interview or before scaling something in prod. The components
and the trade-offs, not the buzzwords.

## The standard scaling story (memorize the arc)

1. **Single server** - app + DB on one box. Fine until it isn't.
2. **Split the DB out** - app tier and data tier scale independently.
3. **Add a load balancer + horizontal app replicas** - stateless app servers behind an LB.
4. **Add caching** - offload reads (CDN, Redis, app cache).
5. **Replicate the DB** - read replicas for read-heavy load.
6. **Shard the DB** - when writes/data outgrow one node.
7. **Add async** - queues for work that doesn't need to be synchronous.

Interview move: start at #1, ask about scale, add pieces only when a bottleneck forces it.

## Load balancing

- Spreads traffic across replicas; enables horizontal scale + failover.
- **L4** (TCP) = fast, dumb. **L7** (HTTP) = routes on path/host/header, does TLS termination.
- Algorithms: round-robin, least-connections, IP-hash (sticky), weighted.
- **Keep app servers stateless** so any replica can serve any request. Session state → Redis/JWT, not local memory.
- Health checks pull dead replicas out of rotation.

## Caching

| Layer | Example | Caches |
|---|---|---|
| Client/browser | Cache-Control | Static assets |
| CDN | CloudFront | Static + cacheable responses at the edge |
| Application | Caffeine (in-JVM) | Hot objects, per-instance |
| Distributed | Redis / ElastiCache | Shared across app replicas |
| Database | Buffer pool | Pages/query results |

Patterns:
- **Cache-aside** (lazy): app checks cache → miss → read DB → populate cache. Most common.
- **Write-through**: write cache + DB together (consistent, slower writes).
- **Write-behind**: write cache now, DB async (fast, risk of loss).

Invalidation is the hard part. Use **TTLs** + explicit eviction on write. "There are only two
hard things: cache invalidation and naming."

- **Eviction**: LRU (default mental model), LFU, FIFO.
- **Thundering herd / cache stampede**: many misses hit the DB at once when a hot key expires. Fix with request coalescing, jittered TTLs, or lock-on-recompute.

## Database scaling

**Replication** (read scaling):
- Primary handles writes; **read replicas** handle reads. Async replication → **replica lag** → reads can be stale.
- Failover promotes a replica to primary.

**Sharding / partitioning** (write + data scaling):
- Split data across nodes by a **shard key**.
- **Hash sharding**: even distribution, but range queries scatter.
- **Range sharding**: range queries are cheap, but risks hot shards.
- **Consistent hashing**: minimizes reshuffling when you add/remove nodes (only `K/N` keys move).
- Cost: cross-shard joins/transactions are painful. Pick a shard key that keeps related data together and spreads load.

**Vertical vs horizontal**: vertical = bigger box (simple, capped, single point of failure). Horizontal = more boxes (complex, scales far). Default to vertical until it hurts.

## SQL vs NoSQL

| | SQL (Postgres/MySQL) | NoSQL (Dynamo/Mongo/Cassandra) |
|---|---|---|
| Model | Relational, schema, joins | Key-value / document / wide-column |
| Consistency | Strong, ACID | Often eventual, tunable |
| Scale | Vertical + read replicas + sharding | Horizontal by design |
| Best for | Transactions, complex queries, integrity | Massive scale, simple access patterns, flexible schema |

Default to SQL. Reach for NoSQL when the access pattern is simple + scale is huge, or the schema is genuinely fluid.

## Async: queues & pub/sub

- **Queue** (SQS/RabbitMQ): decouple producer from consumer, absorb spikes, retry failures, smooth load. Work is consumed once.
- **Pub/sub** (SNS/Kafka): one event → many consumers (fan-out).
- **Back-pressure**: when consumers can't keep up, the queue grows - monitor depth/lag, scale consumers, or shed load.
- Enables **idempotency** requirement: at-least-once delivery means design consumers to tolerate duplicates.

## Idempotency & rate limiting

- **Idempotency key**: client sends a unique key; server dedupes retries → safe to retry POSTs.
- **Rate limiting** algorithms: token bucket (allows bursts), leaky bucket (smooths), fixed/sliding window (counts per interval). Enforce at the gateway/LB.

## CAP (one line, full sheet elsewhere)

Under a network **partition**, choose **consistency** (reject/stale-block) or **availability**
(serve possibly-stale). No partition = you get both. See [CAP & consistency](cap-consistency.md).

## Reliability patterns

- **Circuit breaker**: stop calling a failing dependency; fail fast, recover gracefully.
- **Retry with backoff + jitter**: don't hammer a struggling service in lockstep.
- **Timeout everything**: a call without a timeout is a resource leak waiting to happen.
- **Bulkhead**: isolate resource pools so one slow dependency can't drain all threads.
- **Graceful degradation**: serve a reduced experience instead of a hard failure.

## Estimation (napkin math - full table elsewhere)

- QPS = daily requests / 86,400 (~10^5 s/day). Peak ≈ 2-3× average.
- Storage = items × size × replication × retention.
- Memory: can the working set / hot data fit in RAM? That decides your cache tier.

See [Napkin math](napkin-math.md) for the latency numbers.

## Interview framework (say this order out loud)

1. **Requirements** - functional + non-functional (scale, latency, consistency, availability).
2. **Estimate** - QPS, storage, bandwidth (napkin math).
3. **API** - the key endpoints.
4. **Data model** - schema + access patterns → informs SQL/NoSQL + shard key.
5. **High-level design** - draw the boxes: client → LB → app → cache/DB → queue.
6. **Deep-dive** - pick the interesting bottleneck and go deep.
7. **Bottlenecks & trade-offs** - where it breaks, how you'd scale/harden it.

## Gotchas / talking points

- **Stateless app servers** are the prerequisite for horizontal scaling - say it early.
- Read replicas give you **eventual consistency** on reads; don't read-after-write from a replica.
- Sharding is a one-way door - hard to change the shard key later. Choose carefully.
- Caching adds a consistency problem; every cache needs an invalidation story.
- "Just add a cache/queue" is only correct if you can name what it *costs* (staleness, ordering, duplicates).
- The bottleneck is almost always the **database** first. Design reads/writes deliberately.

## Quick reference

| Problem | Reach for |
|---|---|
| Too many reads | Read replicas + cache |
| Too many writes | Sharding, async writes |
| Traffic spikes | Queue (absorb), autoscaling |
| Slow dependency | Circuit breaker + timeout + retry/backoff |
| Duplicate work | Idempotency keys |
| Hot key stampede | Jittered TTL + request coalescing |
| Global low latency | CDN + edge caching |
| Reshuffle on scale | Consistent hashing |
