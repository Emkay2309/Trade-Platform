# SQL & Database

## Q: What is MVCC, and how does it support concurrent database operations?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round)
**Your answer:**

**Refined answer:**

MVCC (Multi-Version Concurrency Control) lets readers and writers operate on a table concurrently without blocking each other, by keeping **multiple versions** of a row instead of locking it for reads. When a transaction updates a row, the database doesn't overwrite it in place — it creates a new version, while older transactions that started before the update keep seeing the version that existed when *their* transaction began (a consistent snapshot). Postgres implements this directly (every row has hidden `xmin`/`xmax` transaction-ID columns marking its visibility window). The payoff: `SELECT` never has to block waiting for a concurrent `UPDATE`/`INSERT` to finish, which is why Postgres/MySQL(InnoDB) can sustain high read concurrency alongside writes.

**Follow-ups to expect:**
- What's the trade-off? (Old row versions accumulate until vacuumed/garbage-collected — Postgres's `VACUUM` process reclaims them; a long-running transaction can keep old versions alive longer than expected, bloating the table.)

---

## Q: What is a covering index, and when should you use one?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A covering index includes **every column a query needs** — both the filter columns and the selected columns — so the database can answer the query entirely from the index itself, without a second lookup into the actual table ("index-only scan"). E.g. an index on `(status, trade_date, amount)` covers `SELECT amount FROM trades WHERE status = ? AND trade_date = ?` completely — no need to fetch the full row. Use it for frequently-run, read-heavy queries where the selected columns are a small, stable subset — the cost is a wider index (more storage, slightly more expensive writes since the index has more to update).

---

## Q: How would you identify duplicate transactions using multiple columns?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Group by the combination of columns that together define "the same transaction" (e.g. `account_id, amount, timestamp` rounded to some tolerance, or better, an actual idempotency/request key if one exists) and filter for groups with `COUNT(*) > 1`:
```sql
SELECT account_id, amount, request_id, COUNT(*)
FROM transactions
GROUP BY account_id, amount, request_id
HAVING COUNT(*) > 1;
```
The real engineering point: this is a *detection* query for data that's already duplicated — the actual fix is preventing duplicates at write time via a unique constraint on the idempotency key (see `microservices-system-design.md`), not relying on a periodic cleanup query.

---

## Q: How would you archive millions of records without disrupting production?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Don't run one giant `DELETE`/`INSERT ... SELECT` — it holds long locks, bloats the transaction log, and can stall production traffic on the same table. Instead: batch it (delete/move a few thousand rows at a time in a loop, each a short transaction, with a brief pause between batches so you're not monopolizing I/O), run it during low-traffic windows, and prefer **partitioning by date** on large tables so "archiving old data" becomes dropping/detaching a whole old partition (near-instant) instead of a row-by-row delete. For true archival (not deletion), copy to cold storage (e.g. S3) before removing from the primary table, and make the whole process idempotent/resumable in case it's interrupted partway.

---

## Q: INNER JOIN vs LEFT JOIN?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 4)
**Your answer:**

**Refined answer:**

`INNER JOIN` returns only rows that have a match in both tables. `LEFT JOIN` returns every row from the left table, with `NULL`s filling in for columns from the right table when there's no match. Use `LEFT JOIN` when you need the parent row regardless of whether a related record exists (e.g. "all trades, with settlement info if it exists yet" — trades not yet settled should still appear, with null settlement fields).

---

## Q: How do indexes improve query performance?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 4)
**Your answer:**

**Refined answer:**

Without an index, the database does a full table scan — checking every row to find matches. An index (typically a B-tree) maintains a sorted structure over the indexed column(s), letting the database jump directly to matching rows (O(log n) lookup) instead of scanning everything (O(n)). The trade-off: every index speeds up reads on that column but slows down writes slightly (the index itself must be updated on every insert/update/delete) and takes extra storage — so indexes are added deliberately for the queries that are actually slow and frequent, not on every column by default.

---

## Q: Explain ACID properties.
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 4)
**Your answer:**

**Refined answer:**

- **Atomicity** — a transaction's operations all succeed or all fail together; no partial application.
- **Consistency** — a transaction can only move the database from one valid state to another (constraints, foreign keys, business rules are never violated mid-transaction-commit).
- **Isolation** — concurrent transactions don't see each other's uncommitted intermediate state (the exact degree depends on the isolation level — see next entry).
- **Durability** — once committed, a transaction's effects survive a crash (written to durable storage/WAL, not just memory).

---

## Q: What are transaction isolation levels?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 4)
**Your answer:**

**Refined answer:**

From weakest to strongest guarantee (and correspondingly, from best to worst concurrency/throughput):
- **Read Uncommitted** — can see another transaction's uncommitted changes ("dirty reads"). Rarely used in practice.
- **Read Committed** — only ever sees committed data, but a value can change between two reads *within the same transaction* ("non-repeatable read"). Postgres's default.
- **Repeatable Read** — the same row read twice within a transaction returns the same value, but a *new* row matching your query's filter can still appear on a re-run ("phantom read").
- **Serializable** — transactions behave as if they ran one at a time, in some serial order; the strongest guarantee, at the highest cost to concurrency (more conflicts/aborts under contention).

**How this maps to your project:** worth explicitly deciding the isolation level for `settlement-service`'s balance/position updates (Phase 4) rather than leaving it at the database default without a reason.

---

## Q: (MongoDB) How do you optimize query performance under high traffic, design a scalable schema, and manage indexes for read/write efficiency?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AI-interviewer round)
**Your answer:**

**Refined answer (brief — this project uses Postgres, not MongoDB; covering the concept, not project-backed experience):**

MongoDB schema design generally favors **embedding** related data that's read together (denormalize for read speed, since there's no cheap join) and **referencing** only when the related data is large, independently updated, or shared across many parents (embedding it everywhere would duplicate updates). Indexing follows the same query-pattern-driven discipline as relational databases — compound indexes matching actual filter+sort patterns, and being deliberate about **write-heavy** collections where too many indexes slow down every insert/update. Under high read traffic, **read replicas** (secondary reads) offload load from the primary; under high write traffic, **sharding** by a well-chosen shard key distributes writes, and a poorly chosen shard key (low cardinality, or one that concentrates writes on one shard) is the most common scaling mistake.

**How this maps to your project:** not applicable directly (Postgres is the chosen store for the ledger/trade data, deliberately — see the "fund transfer system" entry in `microservices-system-design.md` for why a relational store was the right call here). Keep this as conceptual knowledge unless a target role specifically uses MongoDB.

---

## Q: How would you optimize a slow SQL query?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 4)
**Your answer:**

**Refined answer:**

Start with `EXPLAIN ANALYZE` to see the actual execution plan — don't guess. Common findings and fixes: a sequential scan where an index should exist (add one on the filtered/joined columns); an index that exists but isn't being used because the query applies a function to the column (`WHERE LOWER(name) = ...` can't use a plain index on `name` — either a functional index or rewriting the query is needed); a join order that explodes row counts before filtering (restructure so filters apply earlier); or simply selecting far more columns/rows than actually needed (`SELECT *` pulling unused columns, missing a `LIMIT`). The discipline is: measure with the plan, fix the specific thing the plan says is expensive, re-measure — not guessing at indexes speculatively.
