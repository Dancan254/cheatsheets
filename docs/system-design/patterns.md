# System Design - Patterns

Reach for this for the named distributed-systems patterns and when to apply each.

!!! note "Stub - scaffolded, not yet filled"
    Section skeleton below. Fill with diagrams/notes over time.

## CQRS (command/query responsibility segregation)

## Event sourcing

## Saga (distributed transactions - choreography vs orchestration)

## Outbox pattern (reliable event publishing)

## Circuit breaker

## Retry with backoff + jitter

## Bulkhead

## Idempotent consumer

## API gateway / BFF

## Strangler fig (incremental migration)

## Gotchas / things I always forget

## Quick reference

| Problem | Pattern |
|---|---|
| Reads ≠ writes shape | CQRS |
| Reliable event publish | Outbox |
| Cross-service transaction | Saga |
| Failing dependency | Circuit breaker |
| Duplicate messages | Idempotent consumer |
| Legacy migration | Strangler fig |
