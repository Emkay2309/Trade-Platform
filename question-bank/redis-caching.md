# Redis & Caching

## Q: Why choose the cache-aside pattern? How is it different from write-through?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo round)
**Your answer:**

**Refined answer:**

**Cache-aside (lazy loading):** the application checks the cache first; on a miss, it reads from the database, then populates the cache for next time. The cache only ever contains data that's actually been requested — good when a large dataset is rarely fully accessed (why cache rows nobody reads?), and the application stays in control of exactly when/what gets cached. Downside: the first request for any given key always pays the full DB-read cost (cold-cache penalty), and the cache can briefly hold stale data between a DB update and the next read-through.

**Write-through:** every write goes to the cache *and* the database together (or the cache writes to the DB on the application's behalf), so the cache is always consistent with the DB immediately. Good when reads must never see stale data, at the cost of making every write slightly slower (it's now two writes) and caching data that might never be read.

**Why cache-aside is the more common default:** most read-heavy, write-light workloads (reference data, product catalogs) tolerate a *little* staleness far better than they tolerate slower writes — and cache-aside only pays caching cost for data that's actually read.

**How this maps to your project:** `reference-data-service` (instruments/counterparties — read far more often than written) is a textbook cache-aside candidate: check Redis, fall back to Postgres on miss, populate Redis with a TTL.

**Follow-ups to expect:**
- What invalidates a cache-aside entry when the underlying row changes? (Either a short TTL that accepts brief staleness, or an explicit cache-evict on write — see `@CacheEvict` in `spring-boot.md`.)

---

## Q: If 1,000 requests access the same Redis key immediately after it expires, how do you prevent a cache stampede?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo round)
**Your answer:**

**Refined answer:**

A **cache stampede** (a.k.a. thundering herd) happens when a hot key expires and every concurrent request that was relying on it misses the cache at the same instant, and all of them independently go hit the database to recompute/refetch the same value — multiplying what should have been one DB query into a thousand, right when the system is already under load.

**Mitigations:**
- **Locking/single-flight** — the first request to miss acquires a short-lived Redis lock (`SET key value NX EX <ttl>`) and is the *only* one allowed to recompute the value; other concurrent requests either wait briefly and then retry the cache, or are served a slightly stale/fallback value instead of all hitting the DB.
- **Probabilistic early refresh** — recompute the value slightly *before* it actually expires (e.g. a background refresh triggered randomly as TTL approaches zero), so there's rarely a hard "nothing in cache" moment at all.
- **Staggered/jittered TTLs** — don't give a large batch of keys the exact same TTL, so they don't all expire in the same instant in the first place.

**How this maps to your project:** worth deliberately testing in `reference-data-service` — set a short TTL, fire concurrent requests right at expiry, and observe the stampede before adding the lock-based fix, so you've actually seen the failure mode you're defending against.

---

## Q: How are you using Redis in your application? (open-ended — project-specific)
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — Wells Fargo round)
**Your answer:**

**Note:** this one is intentionally project-specific — there's no generic answer to memorize. Once `reference-data-service` has real Redis caching wired in (Phase 1), come back and write the actual answer here: what's cached, the TTL chosen and why, cache-aside vs write-through decision made and why, and any stampede/invalidation handling added. An interviewer will probe a canned answer here immediately ("what TTL did you use, and why that number?") — this one needs to be real.
