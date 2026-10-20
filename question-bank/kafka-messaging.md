# Kafka & Messaging

## Q: You have 20 Kafka consumers but only 8 partitions — how many consumers actively process messages?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo/Mphasis round)
**Your answer:**

**Refined answer:**

**8.** Within one consumer group, Kafka assigns each partition to exactly one consumer at a time — a partition is never shared between two consumers in the same group. With 8 partitions and 20 consumers in the same group, only 8 consumers get a partition assigned and actually process messages; the other 12 sit idle as standby/do nothing, ready to pick up a partition only if one of the active 8 dies. This is *the* reason partition count is a hard ceiling on a consumer group's parallelism — scaling consumers past the partition count buys you nothing until you also increase partitions.

**Follow-ups to expect:**
- What happens if you increase the partition count later on a topic with existing data? (See next entry — it doesn't retroactively fix ordering for existing keys, and it's a one-way operation: you can't decrease partitions.)

---

## Q: What happens when a Kafka topic's partition count increases?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

New messages get distributed across the new, larger set of partitions (partition = `hash(key) % partitionCount`), so the moment you add partitions, the same key can start landing in a *different* partition than it used to. For keyed topics where ordering-per-key matters, this breaks the "all events for this key are in order" guarantee across the increase point — events for the same key before and after the resize may now be split across two partitions with no ordering relationship between them. Also, this is a one-way operation: Kafka lets you increase partitions on an existing topic but never decrease them. Because of both of these, partition count is usually planned up front based on expected peak parallelism, not treated as a dial to casually turn.

**How this maps to your project:** the trade-by-account-ID partitioning scheme in `settlement-service`'s Kafka topics should be sized for projected load *before* going live — not something we casually bump later.

---

## Q: How would you replay historical events without affecting live processing?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Don't reset the offset of the *live* consumer group — that would make your production consumers reprocess everything and double-handle events currently flowing through. Instead, spin up a **separate consumer group** (a new `group.id`) dedicated to the replay, pointed at the earliest offset (or a specific timestamp via `offsetsForTimes`) for the topics in question. It reads independently, at its own pace, writing to wherever the replay needs to go (a backfill job, a different environment, an analytics sink) — completely isolated from the live group's offsets and consumption. If the replay target is the *same* downstream system the live consumer also writes to, the replayed events must be idempotent (same key as before) so replaying doesn't double-apply them.

**Follow-ups to expect:**
- Why can't you just rewind the live consumer group's offset? (It would make the live consumer reprocess everything from that point forward in real time, re-triggering every downstream side effect — emails, settlements, whatever that consumer does — a second time.)

---

## Q: How would you handle poison messages with controlled redelivery?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A "poison message" is one that *always* fails processing (malformed payload, a bug triggered by this specific data) — naively retrying it forever blocks the partition, since Kafka consumers process a partition's messages in order and won't move past a message that keeps throwing. The standard fix: after a bounded number of retry attempts (tracked per-message, e.g. via a retry-count header), stop retrying inline and route the message to a **Dead Letter Topic (DLT)** instead — so the consumer can commit past it and keep processing the rest of the partition, while the poison message is preserved for investigation/manual reprocessing rather than silently dropped.

Spring Kafka supports this directly via `DefaultErrorHandler` with a configured `DeadLetterPublishingRecoverer` and a retry policy (`FixedBackOff`/`ExponentialBackOffWithMaxRetries`).

**How this maps to your project:** `notification-audit-service`'s Kafka consumer is where this matters most — a malformed or unexpected event shape shouldn't be able to permanently stall audit processing for every trade behind it.

---

## Q: How do you ensure zero message loss in Kafka?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo round)
**Your answer:**

**Refined answer:**

Loss can happen on the producer side, the broker side, or the consumer side — each needs its own guarantee:

- **Producer:** `acks=all` — the producer waits for acknowledgment from *all* in-sync replicas, not just the leader, before considering the write successful. Combine with `retries` set high (or `Integer.MAX_VALUE`) and `enable.idempotence=true` so producer-side retries can't create duplicates.
- **Broker:** `replication.factor >= 3` with `min.insync.replicas=2` — a write isn't acknowledged unless it's durably on multiple brokers, so losing one broker doesn't lose data.
- **Consumer:** don't commit an offset until the message has actually been fully processed (`enable.auto.commit=false`, manual `ack` after processing succeeds) — otherwise a crash between "offset committed" and "processing completed" silently loses that message's effect.

**How this maps to your project:** given this project is explicitly modeling a financial trade lifecycle, this configuration (`acks=all`, idempotent producer, manual offset commit) is the non-negotiable baseline for every Kafka producer/consumer we build, not an optional tuning knob.

---

## Q: What is exponential backoff, and how do you implement retries for Kafka consumers?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Exponential backoff retries a failed operation with a growing delay between attempts (1s, 2s, 4s, 8s...) instead of immediately or at a fixed interval — giving a struggling downstream dependency time to recover instead of hammering it right as it's failing. In Spring Kafka, configure it on the `DefaultErrorHandler` via `ExponentialBackOff` (initial interval, multiplier, max interval), typically combined with a max-attempts cap before routing to a Dead Letter Topic (see poison-message entry above) — backoff handles *transient* failures, DLT handles the *permanent* ones.

---

## Q: How would you coordinate a workflow involving three independent microservices?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

This is the Saga question again from the messaging angle (see `microservices-system-design.md`) — coordinate via either choreography (each service publishes an event the next one listens for) or orchestration (a central orchestrator calls each service in sequence and handles failures/compensations explicitly). For three services specifically, orchestration is usually easier to reason about and debug than choreography, since the full sequence lives in one place rather than being inferred from three services' independent event-handling logic.

---

## Q: How would you process 1 lakh (100,000) messages per second in Kafka?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Throughput at that scale comes from parallelism and batching, not from making any single consumer faster in isolation:

- **Enough partitions** to allow enough parallel consumers — partition count is the hard ceiling on consumer-side parallelism within a group (see the "20 consumers, 8 partitions" entry above).
- **Batch on both sides** — producers batching sends (`linger.ms`, `batch.size`) instead of one network round-trip per message; consumers pulling and processing in batches (`max.poll.records`) instead of one record at a time.
- **Keep per-message consumer processing cheap** — if processing does a slow synchronous call per message (a DB write, an external API call), that's the actual bottleneck regardless of partition count; push slow work to async/batched writes where correctness allows.
- **Scale brokers/horizontal partitioning** across enough broker nodes that no single broker's disk/network becomes the ceiling.

**Follow-ups to expect:**
- If processing can't be made faster, what's the fallback? (Add more partitions and consumers up to the point where something else — the database, a downstream API — becomes the bottleneck instead; at that point the bottleneck has simply moved, not disappeared.)
