# Microservices & System Design

## Q: How do microservices communicate with each other?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — asked 3+ times across different posts)
**Your answer:**

**Refined answer:**

Two broad styles: **synchronous** (REST/gRPC — caller waits for a response, simple to reason about, but couples the caller's availability to the callee's) and **asynchronous** (message broker like Kafka/RabbitMQ — caller publishes an event and moves on, decoupled in time, naturally resilient to a consumer being temporarily down, but harder to reason about ordering/consistency). Real systems use both: synchronous for request/response needs where the caller genuinely needs an answer now, asynchronous for anything that can be eventually consistent (notifications, audit, cross-service workflows).

**How this maps to your project:** `trade-capture-service` calling `risk-service` synchronously (needs an immediate accept/reject) vs `settlement-service` publishing a `TradeSettled` event that `notification-audit-service` consumes asynchronously.

**Follow-ups to expect:**
- See "When would you choose synchronous REST over asynchronous Kafka?" below — same question from the other direction.

---

## Q: When would you choose synchronous REST over asynchronous Kafka communication?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo/Mphasis round)
**Your answer:**

**Refined answer:**

Choose **REST** when the caller needs an immediate answer to proceed (e.g. "is this trade within risk limits?" — you can't settle a trade without knowing). Choose **Kafka** when the work can happen eventually, the caller doesn't need to block on it, or multiple independent consumers need to react to the same event (audit, notification, analytics all reacting to one `TradeSettled` event without the publisher knowing or caring who's listening). A good signal: if you find yourself wanting "fire and forget, but reliably," that's a message broker; if you need "I can't continue without this answer," that's a synchronous call (ideally wrapped in a circuit breaker).

---

## Q: API Gateway vs Load Balancer vs Service Discovery — what are the use cases?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

- **Load Balancer** — distributes incoming traffic across multiple instances of the *same* service, for scale and availability.
- **API Gateway** — a single entry point for *external* clients into the system; handles cross-cutting concerns (auth, rate limiting, request routing to the correct backend service, response aggregation) so individual services don't each reimplement them.
- **Service Discovery** — lets services find each other's current network location dynamically (instances scale up/down, IPs change) instead of hardcoding addresses — e.g. Eureka, or Kubernetes' built-in DNS-based discovery.

These are complementary, not alternatives: a request typically hits a Load Balancer, which routes to an API Gateway instance, which uses Service Discovery to find the right downstream service instance.

**How this maps to your project:** `api-gateway` (Spring Cloud Gateway) + Service Discovery (Eureka, or K8s-native once we containerize) are both explicit services in `PROJECT_PLAN.md`.

---

## Q: Explain the Saga pattern for managing distributed transactions across microservices.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — asked 3+ times; fund-transfer, e-commerce checkout, and generic framings)
**Your answer:**

**Refined answer:**

When one business transaction spans multiple services that each own their own database, a single ACID transaction can't cover the whole thing. A **Saga** breaks the transaction into a sequence of local transactions, each committed independently in its own service. If a later step fails, instead of a database rollback, the Saga runs **compensating actions** — explicit business operations that undo the effect of earlier steps.

Example (e-commerce checkout): Create Order → Process Payment → Reserve Inventory → Create Shipment. If inventory reservation fails after payment succeeded, the Saga doesn't "roll back" in the database sense — it runs compensations: refund payment, cancel order, notify the customer. Compensations are themselves business operations that can fail and need their own retry/recovery strategy.

**Two implementation styles:**
- **Choreography** — services react to each other's published events, no central coordinator. Fewer coupling dependencies, but the overall workflow becomes hard to trace as it grows (logic is smeared across services).
- **Orchestration** — a central Saga orchestrator explicitly drives each step and decides what compensation to run on failure. Workflow state and recovery logic are explicit in one place, at the cost of that orchestrator becoming a component that itself must be reliable.

**Senior-level practices that come up in follow-ups:**
- Persist Saga state so an in-flight workflow can resume after a crash.
- Make every step and compensation **idempotent** (a retried step shouldn't double-apply).
- Track a correlation/Saga ID across all services involved, for tracing and debugging.
- Define bounded retries and timeouts per step — don't retry forever.

**The one-line answer an interviewer wants to hear:** "A Saga doesn't make a distributed transaction atomic — it makes its failure and recovery paths explicit." Use it for long-running cross-service workflows; for anything contained in one database, a normal transaction is simpler and you shouldn't reach for a Saga.

**How this maps to your project:** this is literally `settlement-service` in `PROJECT_PLAN.md` Phase 4 — we'll build it as an **orchestrated** Saga (explicit state machine) driving Trade Capture → Risk Check → Settlement → Notification, with real compensating actions (e.g. a failed settlement must reverse a provisional risk-limit reservation).

**Follow-ups to expect:**
- Common mistake: assuming Saga gives global atomicity, or assuming a compensation (like a refund) always succeeds immediately — it can fail too, and needs its own handling.
- Why is choreography harder to debug at scale? (No single place shows the whole workflow — you have to reconstruct it from events scattered across services' logs.)

---

## Q: Your database transaction commits successfully, but publishing the event to Kafka fails. How do you handle it? (Transactional Outbox pattern)
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo/Mphasis round)
**Your answer:**

**Refined answer:**

This is the **dual-write problem**: you can't atomically write to your database *and* publish to Kafka in one transaction — they're two different systems. If you publish-then-commit and the commit fails, you've told the world about something that never happened; if you commit-then-publish and the publish fails (network blip, broker down), you've durably recorded something nobody else will ever hear about.

The standard fix is the **Transactional Outbox pattern**: instead of publishing directly to Kafka inside the business transaction, write the event into an `outbox` table in the *same* database transaction as the business change — so either both commit or neither does, using the database's own atomicity guarantee. A separate process (a polling publisher, or Debezium reading the DB's change log/CDC) then reads unpublished outbox rows and publishes them to Kafka, marking them published once Kafka acknowledges. If that publisher crashes mid-way, it just resumes — nothing is lost, because the source of truth is the outbox row, not an in-memory publish attempt.

**How this maps to your project:** `settlement-service` writing a `TradeSettled` row and needing to notify `notification-audit-service` via Kafka is exactly this problem — Phase 3/4 will implement the outbox table alongside the Saga state table, since they solve adjacent problems (reliable state + reliable event publishing).

**Follow-ups to expect:**
- Why not just retry the Kafka publish in a loop right there in the request thread? (Because the request already committed to the DB — if the service crashes between commit and the retry succeeding, the event is lost forever; the outbox survives a crash because it's already durably stored.)
- How does the consumer avoid processing the same outbox-sourced event twice? (Idempotent consumers — see the idempotency entry below.)

---

## Q: How would you prevent duplicate transactions if a user clicks submit multiple times (idempotency)?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — asked 3+ times, PayPal/Wells Fargo/general)
**Your answer:**

**Refined answer:**

The core tool is an **idempotency key**: the client generates a unique key per logical request attempt (e.g. a UUID created once when the user clicks "Pay," reused on any client-side retry of that same click) and sends it in a header or field. The server stores a record of `(idempotency key → result)`. On receiving a request:
1. If the key hasn't been seen, process it normally and store the result against the key.
2. If the key has been seen and already completed, return the **same stored result** without reprocessing — don't charge/create twice.
3. If the key is in-flight (a previous attempt is still processing), either reject with a 409 or make the caller wait, to avoid two concurrent executions racing.

This needs a unique constraint (not just an in-memory check) at the database level on the idempotency key column, so that even a race between two near-simultaneous duplicate requests can't both pass a "does it exist?" check before either inserts.

**How this maps to your project:** `trade-capture-service`'s `POST /trades` endpoint should require an idempotency key from the caller from the start (Phase 1) — a trade submitted twice because of a client retry must not become two trades.

**Follow-ups to expect:**
- Where should the idempotency key be enforced — application logic, or a database unique constraint? (Both — application logic for a fast, friendly response; the DB constraint as the actual correctness guarantee under concurrency.)
- How long do you retain idempotency records, and why can't you keep them forever? (Storage cost vs. the realistic window a client would retry in — usually hours to a few days.)

---

## Q: How would you implement a circuit breaker in Spring Boot?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Use **Resilience4j** (the modern replacement for Netflix Hystrix) with its Spring Boot starter. Annotate the method making the risky call:

```java
@CircuitBreaker(name = "riskService", fallbackMethod = "riskFallback")
public RiskResult checkRisk(Trade trade) {
    return riskServiceClient.check(trade);
}

public RiskResult riskFallback(Trade trade, Throwable t) {
    return RiskResult.deferred(); // e.g., queue for manual review instead of failing the whole request
}
```

The circuit breaker tracks failure rate over a sliding window. Three states: **Closed** (calls go through normally), **Open** (failure rate crossed a threshold — calls fail fast immediately, via the fallback, without even attempting the network call), **Half-Open** (after a wait duration, it lets a few test calls through to see if the downstream has recovered, then closes or re-opens based on the result).

The point isn't just "retry" — it's **failing fast** once a dependency is clearly unhealthy, so your own service doesn't pile up blocked threads/connections waiting on something that isn't going to answer, which is how one failing downstream service cascades into taking down its callers too.

**How this maps to your project:** every inter-service call in this project (`trade-capture-service` → `risk-service`, `settlement-service` → external clearing) gets wrapped in Resilience4j — this is the "cross-cutting" line item in `PROJECT_PLAN.md` Phase 6.

**Follow-ups to expect:**
- Circuit breaker vs retry — why do you usually want both, and in what order? (Retry-with-backoff for transient blips; circuit breaker to stop retrying altogether once it's clear the dependency is down, so retries don't make an overloaded service even worse.)
- What's a good fallback when there isn't an obvious "default value"? (Queue for later/manual review, return cached/stale data if acceptable, or a clear degraded-mode error rather than hanging.)

---

## Q: 6 retry & resilience patterns every backend engineer should know
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — resilience patterns roundup)
**Your answer:**

**Refined answer:**

1. **Timeout** — don't wait forever for a call; give up after a bounded time so your own thread/connection isn't held hostage by a slow dependency.
2. **Retry** — many failures are transient (a blip, not a real outage) — retrying once or twice often succeeds.
3. **Exponential backoff** — space retries out (1s, 2s, 4s, 8s...) instead of hammering a struggling service immediately and repeatedly.
4. **Jitter** — add random variance to the backoff so thousands of clients that failed at the same moment don't all retry in lockstep and re-overwhelm the service the instant it starts recovering.
5. **Circuit breaker** — stop calling a service that's clearly down for a while; fail fast, then test it again later (see above).
6. **Fallback** — when retries and the circuit breaker both say "not available," return something useful anyway — cached data, a sensible default, or a clearly degraded response — instead of propagating a hard failure to the end user.

**The key point to say out loud in an interview:** these work *together*, not individually — retry without backoff+jitter is how a brief outage turns into a thundering-herd outage; a circuit breaker without a fallback just converts one failure mode into another one with no recovery story.

**How this maps to your project:** this full stack (timeout + retry + backoff + jitter + circuit breaker + fallback) is what Resilience4j config gives you almost for free in Phase 6 — the interview value is being able to explain *why* each layer exists, not just that you added an annotation.

---

## Q: Design a rate limiter allowing 100 requests per user per minute.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — coding round)
**Your answer:**

**Refined answer:**

**Algorithm choice — Fixed Window vs Sliding Window:**
- **Fixed window** — count requests in discrete buckets (e.g. "this calendar minute"); simple, but allows a burst of 2x the limit right at a window boundary (100 requests at 0:59, another 100 at 1:00).
- **Sliding window** (log or counter-based) — considers the trailing 60 seconds from *now*, not aligned to the clock; smooths out the boundary-burst problem at the cost of a bit more bookkeeping (e.g. a sorted timestamp log per user, or a weighted combination of the current and previous fixed window's counts).

**Thread-safety:** a per-user counter being incremented by concurrent requests needs atomic increment-and-check — e.g. `AtomicInteger`/Redis `INCR` with an expiry, not a plain read-then-write (which races).

**Across multiple application instances:** an in-memory counter per instance doesn't work — a user's 100 requests could be spread across 5 instances, each thinking it's seen only 20. The counter must live in a **shared store** — Redis is the standard choice: `INCR user:<id>:<window>` is atomic, and `EXPIRE` handles window reset automatically.

**Handling millions of users efficiently:** don't create a key per user until they make a request (lazy creation), rely on Redis's own key expiry rather than a separate cleanup job, and consider a lightweight **token bucket** (store remaining tokens + last-refill timestamp per user, refill lazily on each request) to avoid needing a scheduled job to reset every user's window.

**How this maps to your project:** this is a direct fit for `api-gateway` — rate limiting per client/API key is exactly what a gateway is for, and Redis is already in the stack for `reference-data-service` caching, so the same Redis instance (different keyspace) can back this.

**Follow-ups to expect:**
- Token bucket vs sliding window — which is more memory-efficient at scale? (Token bucket — O(1) state per user vs. a sliding-window log's O(n) timestamps per user.)

---

## Q: How would you prevent cascading failures in microservices?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Cascading failure happens when one slow/failing service causes its callers to pile up blocked threads/connections waiting on it, which makes the *callers* slow or unavailable too, which then does the same thing to *their* callers — a failure propagating outward. Defenses, layered:
- **Timeouts** on every inter-service call — never wait indefinitely.
- **Circuit breakers** — stop calling a service that's clearly unhealthy, so you're not continuously tying up resources on calls that will fail anyway.
- **Bulkhead** — isolate resource pools (thread pools/connection pools) per downstream dependency, so one slow dependency can't exhaust the resources needed to call a *different*, healthy dependency.
- **Load shedding/backpressure** — when overloaded, deliberately reject some requests fast (e.g. 503) rather than accepting all of them and degrading for everyone.

**Follow-ups to expect:**
- What's the difference between a circuit breaker and a bulkhead — don't they overlap? (Circuit breaker decides *whether* to call a dependency at all; bulkhead limits *how much of your own capacity* any single dependency can consume, independent of whether it's currently failing.)

---

## Q: Design a fund transfer system capable of handling millions of transactions, keeping debit/credit consistent and processed in order.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Core tension: Account A's debit and Account B's credit are two separate local operations (possibly even two separate services/databases), and a system handling millions of transfers can't hold a long-lived distributed lock/transaction across both for every transfer.

- **Consistency across the two legs** — a Saga (or, if both accounts share one database, a single local ACID transaction) with a documented failure path: if the debit succeeds and the credit fails, don't leave money "gone" — run a compensation that reverses the debit, and alert if the compensation itself fails (money cannot just silently disappear; this is where you need the "common mistakes" list from the Saga answer — don't assume compensations always succeed).
- **Preventing duplicate transfers from double-clicks** — idempotency key per transfer request, enforced with a DB unique constraint (see idempotency entry above).
- **Ordering per account** — route all transactions affecting a given account through the **same Kafka partition** (partition key = account ID). Kafka guarantees ordering *within* a partition, not across partitions, so keying by account ID ensures every event touching Account A is processed in the order it was produced, even though events for Account B (different partition) are processed independently/in parallel.
- **Database choice** — a relational database (Postgres) for the ledger itself: transfers need strong consistency guarantees and ACID transactions per account, which is exactly what an RDBMS is built for; this is not a "pick NoSQL for scale" situation — scale here comes from sharding/partitioning the relational store by account range, not from relaxing consistency.

**How this maps to your project:** `settlement-service` is this exact problem for trades instead of account transfers — the Saga, idempotency, and Kafka-partition-by-entity-ID techniques are the same.

**Follow-ups to expect:**
- Why key Kafka partitions by account ID specifically, rather than by transaction ID? (Transaction ID is unique per event, so keying by it would spread a single account's events across all partitions randomly — destroying the ordering guarantee you actually need.)

---

## Q: System design pattern glossary — 25 recurring patterns and what problem each solves
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — reference list, not a single Q&A; use as a checklist to recognize which pattern a scenario is pointing at)
**Reference (not a per-item answer — recognize these by name and the one-line problem they solve):**

| Pattern | Problem it solves | Example use case |
|---|---|---|
| Load balancing | Uneven server traffic | High-traffic APIs |
| Caching | Repeated expensive reads | Product catalogs |
| CDN | Slow global delivery | Video/image hosting |
| Database sharding | One DB can't scale | Messaging, activity feeds |
| Replication | Availability + read scale | Catalogs, reporting |
| Consistent hashing | Excessive data movement on rebalance | Distributed caches |
| Batch processing | Large scheduled workloads | Billing, analytics |
| Stream processing | Real-time event handling | Fraud detection |
| Message queues | Sudden traffic spikes | Emails, video processing |
| Publish-subscribe | Many consumers need the same update | Alerts, live scores |
| Event-driven architecture | Tightly coupled services | Orders, ride tracking |
| CQRS | Different read/write demands | Ledgers, order histories |
| Event sourcing | Need complete change history | Transactions, audit logs |
| Saga | Cross-service transactions | Travel booking |
| Transactional outbox | DB updated but event lost | Payments, notifications |
| Idempotency | Duplicate actions on retry | Payments, webhooks |
| Rate limiting | Too many requests | Login attempts, public APIs |
| Circuit breaker | Failing downstream dependency | Payments |
| Retry with backoff | Temporary failures | External API calls |
| Backpressure | Consumer can't keep up | Log ingestion |
| Leader election | Multiple nodes competing | Schedulers |
| Distributed locking | Conflicting concurrent updates | Seat booking, inventory |
| Fan-out | Distributing updates at scale | Social feeds |
| Search indexing | Slow text discovery | Product search |
| Geospatial indexing | Finding nearby entities | Ride-hailing, delivery |

**How this maps to your project:** this project deliberately exercises Saga, transactional outbox, idempotency, rate limiting, circuit breaker, and retry-with-backoff directly. The rest (sharding, CDN, consistent hashing, leader election, geospatial indexing...) you won't build here, but should be able to recognize and discuss conceptually if asked — note any you get asked about in `progress-log/` so we can decide if one deserves a small standalone exercise.

---

## Q: SOLID-based design — redesign a payment system to support multiple payment methods / refactor a tightly coupled module / plugin architecture for new report formats.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AI-interviewer round, design-principles section)
**Your answer:**

**Refined answer:**

All three are the same underlying ask: **Open/Closed Principle via the Strategy (or Factory + Strategy) pattern** — add new behavior without modifying existing code.

Payment methods example: define a `PaymentStrategy` interface (`pay(amount, details)`), one implementation per method (`CreditCardPayment`, `UPIPayment`, `WalletPayment`). The checkout code depends only on the interface; adding a new payment method means adding a new class, never touching existing ones. A `PaymentStrategyFactory` (or Spring's own bean lookup by a discriminator) picks the right implementation at runtime based on the request.

The report-format / plugin-architecture version is the identical shape: a `ReportGenerator` interface, one implementation per format, a registry/factory resolving which one to use — new formats are new classes, not edits to a growing `if/else` chain.

**Why this scales/stays maintainable:** every new variant is an isolated, independently testable class; the core workflow code never needs to change (and re-test) just because a new variant was added — which is exactly what "closed for modification, open for extension" means in practice, not just as a slogan.

**How this maps to your project:** `risk-service`'s rule pipeline is this same shape — each risk rule (`CounterpartyLimitRule`, `PositionLimitRule`, etc.) is a `Strategy`, chained together (Chain of Responsibility, see `design-patterns-checklist.md`), so adding a new risk rule never touches existing rules' code.

**Follow-ups to expect:**
- How is this different from just using an `enum` + `switch`? (A switch statement must be edited every time a new case is added — that's the Open/Closed violation this pattern specifically avoids.)

---

## Q: One REST API used by mobile, web, and partner clients — how do you version and secure it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AI-interviewer round)
**Your answer:**

**Refined answer:**

**Versioning** — URI versioning (`/api/v1/trades`, `/api/v2/trades`) is the most explicit and easiest for clients to reason about; header-based versioning (`Accept: application/vnd.company.v2+json`) keeps URLs stable but is less discoverable. Either way, the real discipline is: never make a breaking change to an existing version — add a new version instead, and give clients a deprecation window.

**Securing multiple consumer types differently** — OAuth2 with distinct client credentials/scopes per consumer type: a partner integration gets a client-credentials grant with scopes limited to what partners should see; a mobile/web app (acting on behalf of a logged-in user) uses an authorization-code-style flow issuing a user-scoped JWT. The API gateway is the natural place to enforce "does this token's scope permit this endpoint," rather than each service re-implementing it.

**How this maps to your project:** `api-gateway` + `auth-service` (Phase 5) is exactly where this lives — scopes per client type map naturally onto what each consumer of the Trade Platform's API should be allowed to see.

---

## Q: Kubernetes — what happens when a pod crashes or a worker node goes down?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo round)
**Your answer:**

**Refined answer:**

**Pod crash:** if the pod is managed by a Deployment (which it should be, not a bare Pod), the ReplicaSet controller notices the actual replica count dropped below desired and schedules a replacement pod. A `livenessProbe` is what actually detects "this pod is unhealthy" and triggers a restart; a `readinessProbe` controls whether traffic is routed to it at all — a pod can be running but not yet ready (e.g. still warming up a cache) and correctly excluded from the load balancer until it reports ready.

**Node goes down:** the control plane detects the node is unreachable (missed heartbeats past `node-monitor-grace-period`), marks it `NotReady`, and after a further eviction timeout, reschedules that node's pods onto healthy nodes — assuming the Deployment/ReplicaSet has replicas defined and there's available capacity elsewhere in the cluster.

**How this maps to your project:** this is exactly what we'll observe hands-on in Phase 8 (K8s manifests) — deliberately killing a pod (`kubectl delete pod ...`) and watching the Deployment replace it is a much stronger interview answer than reciting the mechanism from memory.

**Follow-ups to expect:**
- Difference between liveness and readiness probes, concretely? (Liveness failing restarts the container; readiness failing just pulls it out of the traffic rotation without restarting it.)

---

## Q: Your service suddenly receives 10x traffic — how do you handle scaling?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Short-term (automatic): Kubernetes **Horizontal Pod Autoscaler** scales replica count based on CPU/memory (or custom metrics), and a cloud load balancer spreads traffic across the new replicas — assuming the service is stateless so any replica can serve any request. Medium-term: check whether a shared dependency (database, downstream service) becomes the new bottleneck once the app tier scales — autoscaling the stateless layer doesn't help if the database connection pool or a downstream rate limit is what actually caps throughput. Longer-term: this is exactly when rate limiting, caching, and circuit breakers earn their keep — protecting the system (and its dependencies) from being overwhelmed while capacity catches up, rather than letting 10x traffic become a cascading outage.
