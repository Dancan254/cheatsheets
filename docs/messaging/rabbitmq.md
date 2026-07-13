# RabbitMQ

Reach for this for exchange/queue routing, acknowledgements, and DLQ patterns.
Contrast with [Kafka](kafka.md) (log vs broker) - see the [comparison](broker-comparison.md).

## Mental model

- **Producer** publishes to an **exchange** (never directly to a queue).
- **Exchange** routes to **queues** via **bindings** (matched on a **routing key**).
- **Queue** holds messages until a **consumer** acks them.
- **Consumers** on the same queue **compete** - each message goes to one of them.

```
producer → exchange --(binding: routing key)--> queue → consumer
```

Unlike Kafka, messages are **removed** once acked - it's a broker, not a replayable log.

## Exchange types

| Type | Routes by | Use for |
|---|---|---|
| `direct` | exact routing-key match | point-to-point, routing by category |
| `topic` | routing-key pattern (`*` = 1 word, `#` = 0+ words) | flexible pub/sub (`order.*.created`) |
| `fanout` | ignores key - broadcast to all bound queues | pure broadcast |
| `headers` | header attributes instead of key | rare, complex matching |

```
# topic example
binding "order.*.created"  matches  "order.eu.created"  but not "order.eu.paid"
binding "order.#"          matches  "order.eu.created.retry"
```

## Acknowledgements

```java
// Manual ack - the correct default for at-least-once
channel.basicConsume(queue, false /* autoAck off */, (tag, msg) -> {
    try {
        process(msg.getBody());
        channel.basicAck(msg.getEnvelope().getDeliveryTag(), false);      // done
    } catch (Exception e) {
        channel.basicNack(msg.getEnvelope().getDeliveryTag(), false, false); // reject, don't requeue → DLQ
    }
}, tag -> {});
```

| Action | Effect |
|---|---|
| `basicAck` | remove message, done |
| `basicNack(..., requeue=true)` | put back on the queue (retry - beware infinite loop) |
| `basicNack(..., requeue=false)` | dead-letter it (if DLX configured) or drop |
| auto-ack | acked on delivery → **lost if the consumer crashes mid-process** |

## Prefetch (QoS) - don't let one consumer hog the queue

```java
channel.basicQos(10);   // at most 10 unacked messages in flight per consumer
```

Without prefetch, RabbitMQ pushes everything to the first consumer. Set prefetch to
balance work across competing consumers (start around 10-50, tune by processing time).

## Dead-letter exchange (DLQ) pattern

```java
// Main queue routes rejected/expired messages to a DLX
var args = Map.of(
    "x-dead-letter-exchange", "orders.dlx",
    "x-message-ttl", 60000,              // optional: expire after 60s → DLQ
    "x-max-length", 100000               // optional: cap queue depth
);
channel.queueDeclare("orders", true, false, false, args);
channel.exchangeDeclare("orders.dlx", "fanout", true);
channel.queueDeclare("orders.dlq", true, false, false, null);
channel.queueBind("orders.dlq", "orders.dlx", "");
```

Messages land in the DLQ when: nacked with `requeue=false`, TTL expired, or queue length exceeded. Inspect/replay from there instead of losing them.

## Durability (survive a broker restart)

```java
channel.exchangeDeclare("orders", "topic", true /* durable */);
channel.queueDeclare("orders.eu", true /* durable */, false, false, null);
channel.basicPublish("orders", "order.eu.created",
    MessageProperties.PERSISTENT_TEXT_PLAIN,   // persistent message
    body);
```

Need **all three**: durable exchange + durable queue + persistent message. Miss one and messages vanish on restart. (Durability ≠ zero loss - use publisher confirms for that.)

## Publisher confirms (guaranteed publish)

```java
channel.confirmSelect();
channel.basicPublish(exchange, key, props, body);
if (!channel.waitForConfirms(5000)) {
    // broker didn't confirm - retry / alert
}
```

## Spring AMQP (per your stack)

```java
@RabbitListener(queues = "orders.eu")
public void handle(OrderEvent event) {
    orderService.process(event);
    // ack is automatic on normal return; throw to nack (→ retry or DLQ per config)
}
```
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto        # ack on return, nack on exception
        prefetch: 20
        retry:
          enabled: true
          max-attempts: 3
        default-requeue-rejected: false   # send failures to DLQ, not back to the queue
```

```java
// Publishing
rabbitTemplate.convertAndSend("orders", "order.eu.created", event);
```

## Gotchas / things I always forget

- **Producers publish to an exchange, never to a queue directly.** The default (`""`) exchange routes by queue name, which hides this and confuses everyone later.
- Auto-ack loses messages on a consumer crash - use manual ack (or Spring's `auto` mode which acks on *successful return*, not on delivery).
- `basicNack(requeue=true)` on a poison message = infinite redelivery loop. Route failures to a DLQ instead (`requeue=false` + DLX).
- No prefetch limit → one consumer grabs the whole queue and the rest sit idle. Always set `basicQos`.
- Durability needs the trifecta (exchange + queue + message). One non-durable link and a restart drops everything.
- Ordering is **per queue**, and only holds with a single consumer + prefetch=1; competing consumers process out of order.
- A queue and exchange declared with different args than an existing one → declaration fails (`PRECONDITION_FAILED`). Delete/recreate or match exactly.
- Unlike Kafka, you can't replay acked messages - they're gone. Need replay/audit → keep a DLQ or use Kafka.

## Quick reference

| Concept | Value |
|---|---|
| Default port | 5672 (management UI 15672) |
| Publish target | an **exchange**, not a queue |
| At-least-once | manual/auto-on-return ack |
| Balance consumers | `basicQos(n)` prefetch |
| Handle failures | DLX + `requeue=false` |
| Survive restart | durable exchange + durable queue + persistent msg |
| Ordering scope | per queue (single consumer) |
| Spring listener | `@RabbitListener(queues = "...")` |
