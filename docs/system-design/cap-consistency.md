# CAP & Consistency

Reach for this to reason about consistency trade-offs precisely.

!!! note "Stub - scaffolded, not yet filled"
    Section skeleton below. Fill over time.

## CAP theorem

Under a network **partition (P)**, you pick **Consistency** or **Availability**.
No partition → you get both. It's a choice made *during* a partition, not always-on.

## CP vs AP systems (examples)

## Consistency models

- Strong / linearizable
- Sequential
- Causal
- Eventual
- Read-your-writes, monotonic reads

## PACELC (the extension - latency vs consistency when there's no partition)

## ACID vs BASE

## Quorums (R + W > N)

## Idempotency & conflict resolution (LWW, vector clocks, CRDTs)

## Gotchas / things I always forget

## Quick reference

| Model | Guarantee |
|---|---|
| Strong | Every read sees the latest write |
| Eventual | Reads converge, may be stale now |
| Read-your-writes | You see your own writes |
| Quorum consistent | `R + W > N` |
