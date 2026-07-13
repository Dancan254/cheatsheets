# Broker Comparison - Kafka vs RabbitMQ vs SQS

Reach for this to decide *which* broker a use case wants.

!!! note "Stub - scaffolded, not yet filled"
    Starter table below. Expand with decision notes over time.

## When to reach for which

| | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Model | Distributed log | Message broker | Managed queue |
| Retention | Time/size (replayable) | Until consumed | Until consumed (14d max) |
| Ordering | Per partition | Per queue | FIFO queues only |
| Throughput | Very high | Moderate-high | High (managed) |
| Delivery | At-least/exactly-once | At-least-once | At-least-once (std) / FIFO exactly-once-ish |
| Fan-out | Consumer groups | Exchanges | SNS→SQS |
| Ops burden | High (self-managed) | Medium | None (AWS-managed) |
| Best for | Event streaming, replay, high volume | Complex routing, RPC, per-message workflows | AWS-native, simple decoupling |

## Decision heuristics

- Need to **replay** history / event sourcing → **Kafka**.
- Complex **routing** rules, priorities, RPC → **RabbitMQ**.
- On AWS, want **zero ops**, simple decouple → **SQS** (+ SNS for fan-out).

## Gotchas / things I always forget

## Quick reference

- Ordering guarantee scope differs: Kafka=partition, Rabbit=queue, SQS=FIFO-group.
- All three are effectively at-least-once → design idempotent consumers.
