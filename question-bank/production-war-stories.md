# Production Scenario Questions

**Important honesty note before using any of these:** the two scenarios below are written in the LinkedIn batch as someone else's **first-person** real incident ("I had an API taking 4 seconds..."). Do not present these as your own lived experience in an interview — that's a real incident that happened to a stranger, not to you, and a good interviewer will probe for details (what did the dashboard actually show you, what did you try first, what was the rollout like) that a borrowed story won't survive.

What to actually do with these: understand the root-cause reasoning deeply (that's genuinely valuable and testable knowledge), and — better — **deliberately reproduce the same failure mode in your own project**, then fix it. Once you've done that, replace the "Refined answer" below with your own real before/after, your own numbers, your own fix. That's a story that survives follow-up questions because it's true.

---

## Q: Investigate a payment service that occasionally processes the same transaction twice, even though the client sends only one request.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round)
**Your answer:**

**Refined answer (the reasoning — treat as a checklist to investigate with, not a memorized story):**

Work through the request's path end to end looking for anywhere a retry or a race could cause double-processing:
1. **Client-side retries** — does the client (or a load balancer/proxy in front of the service) retry on timeout? A request that succeeded but whose *response* was lost (network blip) looks identical to a client as "no response, must retry" — and without an idempotency key, the server has no way to tell "this is a retry of something I already did" from "this is a new request."
2. **Database-level race** — two near-simultaneous requests for the same transaction both read "not yet processed" before either has committed its write (a classic read-then-write race, not protected by a unique constraint).
3. **Kafka consumer retries** — if a step publishes an event, then a consumer processes it, then something fails *after* processing but *before* the consumer's offset commit, the consumer will re-receive and reprocess the same event on restart — unless that processing step is itself idempotent.
4. **The actual fix, in order of what to check first:** is there an idempotency key at all? If not, that's almost certainly the root cause and the fix (see `microservices-system-design.md`'s idempotency entry) — a DB-level unique constraint on the key, not just an application-level check, since only the DB constraint is race-proof under true concurrency.

**How this maps to your project:** build the idempotency key into `trade-capture-service`'s `POST /trades` from Phase 1 onward — then, as a deliberate exercise, try to break it (fire the same request twice concurrently without a key, see the duplicate; add the key + unique constraint, see it correctly rejected/deduplicated). That's your real story.

---

## Q: An API takes ~4 seconds to respond, but CPU is only ~40%. Why, and how would you investigate?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer (the reasoning):**

Low CPU + high latency is the signal that the service is **waiting**, not computing — so the investigation should stop looking at algorithms/CPU-bound code and start looking at what the request is blocked on. The specific failure pattern worth knowing cold:

A loop over N items where, per item, a lazy-loaded association triggers an extra DB query (N+1 — see `spring-boot.md`) *and* a synchronous external HTTP call is made per item. With N=15 items and each external call averaging 350ms, that's up to 15×350ms ≈ 5.25s of sequential network waiting alone. But the *bigger* problem is usually that this entire loop runs **inside an open database transaction** — which means the service is holding a DB connection from the pool for the entire 5+ seconds it's waiting on network I/O that has nothing to do with the database. With a small connection pool (say 10 connections) and enough concurrent requests hitting this same endpoint, the pool exhausts, new requests queue, and eventually time out — the service *looks* overloaded, but it's not CPU-bound at all; it's blocked, and concurrency is what turns one slow endpoint into a system-wide outage.

**Fixes, in the order that matters:**
1. Fetch the lazy relation explicitly (`JOIN FETCH`/`@EntityGraph`) instead of triggering it per-iteration.
2. Batch the external calls instead of making them one-by-one inside the loop (parallelize, or redesign the downstream API to accept a batch).
3. **Move any network I/O outside the transaction boundary** — never hold a DB connection while waiting on a network call that doesn't need the DB. This is the single highest-leverage fix.
4. Cache the read-heavy path if a little staleness is acceptable.

**The one-line lesson worth saying in an interview:** "high CPU + high latency means the app is doing too much work; low CPU + high latency means the app is waiting — and that distinction tells you where to even start looking." This is exactly why this bug is dangerous — the code reads fine in review, and passes tests with a handful of seeded records; it only shows up once production brings real data volume, real concurrency, and real network latency together.

**How this maps to your project:** this exact shape — a trade-listing endpoint doing a lazy-loaded reference-data fetch plus a synchronous call to `risk-service`, inside a transaction — is worth deliberately building (badly, on purpose) in Phase 1, measuring the latency under concurrent load, then fixing it in the same three steps above. Then this becomes your real answer, with your own numbers.
