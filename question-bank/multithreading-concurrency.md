# Multithreading & Concurrency

## Q: synchronized vs ReentrantLock — when would you specifically pick ReentrantLock in a real service, and why (tryLock, fairness, interruptibility)?
**Asked/relevant on:** 2026-10-16 (warm-up)
**Your answer:**


**Refined answer:**

`synchronized` is Java's built-in intrinsic lock (monitor), managed entirely by the JVM. It's simple: wrap a block or mark a method `synchronized` and the lock is acquired/released automatically, even if an exception is thrown. Its limitations: a thread blocked waiting to enter a `synchronized` block cannot be interrupted, there's no way to attempt the lock with a timeout, and there's no fairness guarantee — a thread that's been waiting longest isn't guaranteed to go next.

`ReentrantLock` (`java.util.concurrent.locks`) is an explicit lock that adds control `synchronized` doesn't have:
- **`tryLock()` / `tryLock(timeout, unit)`** — attempt to acquire without blocking indefinitely; fail fast or give up after a timeout instead of hanging.
- **`lockInterruptibly()`** — a waiting thread can respond to `Thread.interrupt()` instead of blocking forever.
- **Fairness** — `new ReentrantLock(true)` grants the lock in FIFO order to waiting threads, avoiding starvation, at some throughput cost (it disallows a thread that's already running from "barging" ahead of one that's been waiting).

Because it's explicit, it must always be released in a `finally` block:

```java
private final ReentrantLock lock = new ReentrantLock();

public void process() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

**Real-world justification:** in a service handling concurrent requests, I'd reach for `ReentrantLock` when a request can't be allowed to block indefinitely for a shared resource — e.g. `tryLock(2, TimeUnit.SECONDS)` around a critical section, returning a 503/retry response if the lock isn't acquired in time, instead of letting the thread (and the request) hang. I'd also use it when a background task needs to be cleanly interruptible during shutdown, which `synchronized` doesn't support.

**Follow-ups to expect:**
- What happens to a thread blocked on `synchronized` if it's interrupted? How is that different with `lock.lockInterruptibly()`?
- What's a `fair` lock and what does it cost you?

---

## Q: synchronized vs volatile — what's the actual difference?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 2)
**Your answer:**

**Refined answer:**

`volatile` guarantees **visibility** (a write by one thread is immediately visible to all other threads — no thread caches a stale value in a CPU register/local cache) and prevents certain compiler/CPU instruction reorderings around it, but it does **not** provide atomicity for compound operations. `synchronized` provides both visibility *and* **mutual exclusion** (only one thread executes the block at a time) — which is why `volatile` alone can't make `i++` thread-safe (read-modify-write is three separate steps; visibility of each individual step doesn't stop two threads from interleaving between them), but `synchronized` can. Use `volatile` for a simple flag/status read by multiple threads and written by one (e.g. a `shutdown` flag); use `synchronized` (or an atomic type) when multiple threads need to *modify* shared state.

---

## Q: Why doesn't volatile make `i++` thread-safe?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

`i++` is actually three steps — read `i`, add 1, write back — and `volatile` only guarantees each individual read/write is visible across threads; it says nothing about the three steps happening atomically as one unit. Two threads can each read the same value, both compute "+1," and both write back the same result — one increment is lost. This is exactly why `AtomicInteger.incrementAndGet()` exists: it performs the read-modify-write as a single atomic hardware-backed operation (CAS — compare-and-swap), not just a visible one.

---

## Q: When would you choose AtomicInteger over synchronized?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

When the shared state is a single primitive/reference and the operation is a simple read-modify-write (increment a counter, compare-and-set a flag), `AtomicInteger`/`AtomicReference` (backed by CAS, a lock-free hardware instruction) avoids the overhead of acquiring a lock entirely, and scales better under contention since there's no thread ever blocked waiting for a lock — a failed CAS just retries immediately. Reach for `synchronized`/`ReentrantLock` instead once more than one piece of state must change together atomically (a single atomic variable can't express "update these two fields consistently") — that's a genuine critical section, not just a single-variable update.

---

## Q: What is a deadlock, and how can you prevent it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 2)
**Your answer:**

**Refined answer:**

A deadlock happens when two (or more) threads each hold a lock the other needs, and neither will release what it holds until it acquires what it's waiting for — so both wait forever. Classic shape: Thread A locks `resource1` then tries to lock `resource2`; Thread B locks `resource2` then tries to lock `resource1`.

**Prevention:**
- **Consistent lock ordering** — always acquire locks in the same global order across all code paths (e.g. always lock the lower account ID before the higher one in a funds-transfer scenario) — if nobody ever acquires B-then-A, the cycle can't form.
- **Lock timeout** — `tryLock(timeout)` instead of an unbounded `lock()`, so a thread gives up and backs off instead of waiting forever.
- **Minimize lock scope** — hold locks for the shortest time possible, and avoid acquiring a second lock while already holding one, where the design allows it.

**Follow-ups to expect:**
- See "how would you detect a deadlock in production" below — the natural next question.

---

## Q: How would you detect a deadlock in a production Java application?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A **thread dump** (`jstack <pid>`, or triggered via a profiler/APM tool) shows every thread's state and stack; the JVM itself explicitly detects and reports cycles among `synchronized` monitor locks in the dump ("Found one Java-level deadlock"). For `ReentrantLock`-based deadlocks (not JVM-monitor-based), the dump still shows each thread `WAITING` on a lock held by another thread — you follow the chain manually (Thread A waiting on a lock owned by Thread B, Thread B waiting on a lock owned by Thread A). In practice, the symptom that prompts looking for this in the first place is usually a service that's stopped making progress but isn't crashing or showing high CPU — threads are alive, just permanently blocked.

---

## Q: Your Java application suddenly has thousands of threads — what could be causing it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Most common cause: an unbounded thread-creation path under load — e.g. `@Async` without a configured bounded executor (defaults to `SimpleAsyncTaskExecutor`, which creates a new OS thread per call — see `spring-boot.md`), or manually calling `new Thread(...).start()` per request/task instead of using a pool. Also check for threads that are being created but never terminating (a thread stuck blocked indefinitely, so the pool/count only ever grows), and whether a thread pool's queue is unbounded, masking the real problem until it's this severe. The fix is always the same shape: a *bounded* executor with a sane core/max size and an explicit rejection policy, not an unbounded thread-per-task approach.

---

## Q: ExecutorService vs CompletableFuture — what's the difference? Does CompletableFuture create its own thread pool?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round, PwC round 2)
**Your answer:**

**Refined answer:**

`ExecutorService` is the lower-level abstraction — submit a `Runnable`/`Callable`, get back a `Future` you can `.get()` (which **blocks** the calling thread until the result is ready — no good way to chain further async work off of it without blocking). `CompletableFuture` builds on top of that idea but adds **composability**: `.thenApply()`, `.thenCompose()`, `.thenCombine()`, `.exceptionally()` let you chain, combine, and handle errors across async steps *without* blocking in between, and it implements `Future` so it's still compatible with code expecting one.

**Does it create its own thread pool?** By default, `CompletableFuture.supplyAsync(...)` (and its no-executor-argument variants) run on the **common `ForkJoinPool`** (`ForkJoinPool.commonPool()`) shared JVM-wide — not a pool dedicated to that one `CompletableFuture`. This matters: if you're also using the common pool for CPU-bound parallel streams elsewhere, a slow blocking `CompletableFuture` task can starve that shared pool. The fix is passing an explicit executor (`supplyAsync(() -> ..., myExecutor)`) when the work is I/O-bound/blocking or when you want isolation from other common-pool consumers.

**Follow-ups to expect:**
- Why would you deliberately pass a custom executor instead of relying on the common pool? (Isolation from other common-pool users, and control over sizing for I/O-bound vs CPU-bound workloads, which want very different pool shapes.)

---

## Q: What is a Race Condition? What is Synchronization, and how does it work? Callable vs Runnable? wait() vs sleep() vs join()?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round; grouped fundamentals)
**Your answer:**

**Refined answer:**

**Race condition** — the program's correctness depends on the unpredictable timing/interleaving of multiple threads accessing shared state; the classic lost-update `i++` example above is one instance of this general problem.

**Synchronization** — Java's mechanism (the `synchronized` keyword, backed by each object's intrinsic monitor lock) to ensure only one thread executes a critical section at a time, and that changes made inside it are visible to the next thread that enters (synchronization provides both mutual exclusion and a visibility guarantee, unlike `volatile` alone).

**`Callable` vs `Runnable`** — `Runnable.run()` returns nothing and can't throw a checked exception; `Callable<V>.call()` returns a value `V` and *can* throw a checked exception — use `Callable` whenever the submitted task needs to produce a result (and you'll get it back via the `Future` an `ExecutorService.submit(Callable)` returns, vs. `execute(Runnable)` which gives nothing back).

**`wait()` vs `sleep()` vs `join()`** — `Thread.sleep(ms)` pauses the current thread without releasing any lock it holds, and resumes on its own after the time elapses. `Object.wait()` (called on a monitor object, inside a `synchronized` block) releases that object's lock and pauses until another thread calls `notify()`/`notifyAll()` on the same object — the classic building block for producer-consumer coordination. `Thread.join()` blocks the *calling* thread until the thread it's called on finishes entirely — used to wait for another thread's completion, not for intra-thread coordination.

---

## Q: Why is HashMap not thread-safe, and how does ConcurrentHashMap work internally?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

`HashMap` performs no internal synchronization at all — two threads calling `put()` concurrently can corrupt its internal bucket/tree structure (lost updates, or in older Java versions, even infinite loops during a concurrent resize); it's fast precisely *because* it assumes single-threaded (or externally-synchronized) access.

`ConcurrentHashMap` achieves thread-safety without locking the entire map for every operation: historically (pre-Java 8) via **segment-level locking** (the map was divided into segments, each with its own lock, so unrelated keys in different segments didn't contend); from Java 8 onward, it instead locks at the **individual bucket (node) level** using synchronized blocks on the first node of a bin, combined with CAS-based operations for simple cases — meaning two threads writing to *different* buckets don't block each other at all, and reads generally don't block writes. This gives far better concurrent throughput than wrapping a `HashMap` in one coarse `synchronized` block (`Collections.synchronizedMap`), which serializes *every* operation through a single lock regardless of which keys are involved.

**Follow-ups to expect:**
- Why not just always use `ConcurrentHashMap` even in single-threaded code? (Its lock-striping/CAS machinery has overhead a plain `HashMap` doesn't pay — use the concurrent version only when actually sharing the map across threads.)

---

## Q: What is ThreadLocal, and how can it create a memory leak?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

`ThreadLocal<T>` gives each thread its own independent copy of a variable — no sharing, no synchronization needed, commonly used for per-request context (e.g. a correlation ID for request tracing — directly relevant given this project's Saga/idempotency correlation-ID needs). The memory leak risk: a `ThreadLocal`'s value is stored in a map *owned by the Thread object itself* — in a **thread pool** (threads are reused, never actually terminated between tasks), if a value is set via `threadLocal.set(...)` but never cleared with `threadLocal.remove()`, that value stays referenced by the pooled thread indefinitely, across every future task that thread ever runs — accumulating memory that's never released, and potentially leaking the *previous* request's context into a later, unrelated request.

**The fix:** always pair `set()` with a `remove()` in a `finally` block (or use a request-scoped framework abstraction that does this for you), especially in anything running on a pooled thread (which, in a Spring Boot app handling HTTP requests, is essentially everything).

---

## Q: What are Virtual Threads in Java 21, and what actually changed inside the JVM?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — multiple detailed posts; PwC round 2; very high interview relevance given the project runs on Java 21)
**Your answer:**

**Refined answer:**

Traditional ("platform") threads map **1:1 to OS threads** — each one reserves native stack memory (~1MB), consumes kernel scheduling resources, and is genuinely expensive. 100,000 concurrent requests as 100,000 platform threads means real memory pressure, context-switching overhead, and scheduler contention long before any actual CPU work is the bottleneck.

**Virtual threads** change the model to **many virtual threads running on a few carrier (OS) threads**. They aren't just "smaller threads" — they're implemented using **continuations**: a paused computation that can be resumed later. When a virtual thread blocks on I/O (a DB call, an HTTP request, `sleep()`), the JVM captures its stack state into **heap-based stack chunks**, unmounts the virtual thread from its carrier thread, and frees that carrier thread to run other work. So "a blocked request" no longer means "a blocked OS thread" — this decoupling is the actual scalability breakthrough (Project Loom), not just "threads got cheaper." Platform thread stacks are contiguous native memory; virtual thread stacks live on the heap, which is what makes millions of concurrently-waiting virtual threads practical where millions of platform threads never would be.

**The pinning caveat:** in Java 21–23, a virtual thread blocking *inside a `synchronized` block* couldn't unmount — it pinned its carrier thread, defeating the benefit right where you'd most want it. **Java 24's JEP 491** made JVM monitors virtual-thread-aware, fixing most of this (synchronized blocking no longer pins in the common case). Some pinning still exists for native/JNI calls, since native execution can't always be safely suspended.

**What virtual threads do *not* do:** improve CPU-bound performance — they optimize blocking/I/O-bound workloads, concurrency efficiency, and memory usage, not raw computation speed. And the idiom is different from pooling: **use one virtual thread per task** (they're cheap enough that pooling them defeats the purpose) rather than reusing a fixed pool as you would with platform threads.

**Enabling in Spring Boot 3.2+:** `spring.threads.virtual.enabled=true` switches the embedded Tomcat's request-handling threads to virtual threads with no other code changes — the appeal is that application code keeps writing ordinary blocking-looking code, while the JVM transparently handles suspension/scheduling/remounting underneath.

**How this maps to your project:** since the whole platform is on Java 21, this is a legitimate design decision point to be explicit about, not just a trivia answer — services making a lot of blocking calls (any service calling another service synchronously, or doing blocking DB I/O) are exactly where virtual threads help; worth deliberately comparing with/without on one service once there's real inter-service traffic to measure (Phase 1+), rather than enabling it blindly everywhere.

**Follow-ups to expect:**
- Would virtual threads make the custom `ThreadPoolTaskExecutor` for `@Async` unnecessary? (Partially — virtual threads remove "threads are expensive" as a reason to bound concurrency, but you may still want bounding for *backpressure* reasons — protecting a downstream dependency from unlimited concurrent calls, which is a different concern than thread cost.)
- Why doesn't this help a CPU-bound batch computation? (No amount of unmounting/remounting helps if the thread is never actually blocked/waiting — it's just computing, so there's nothing for a continuation to free up.)

---

## Q: How does ThreadPoolExecutor actually decide core size, max size, and the queue — walk through what happens when a task is submitted?
**Asked/relevant on:** 2026-10-20 (added — the mechanics behind the `ThreadPoolTaskExecutor` config already used in the `@Async` entry in `spring-boot.md`; worth knowing cold since it's the same four knobs everywhere a bounded pool shows up in this project)
**Your answer:**

**Refined answer:**

On each submitted task, `ThreadPoolExecutor` follows a specific order — **not** "always create a thread up to max first":
1. If fewer than **core pool size** threads exist, create a new one to run this task immediately (even if other core threads are idle — this is a common misconception; it does create up to core size first, though).
2. Once at core size, **queue** the task instead of creating a new thread — up to the queue's capacity.
3. Only once the **queue is full** does it create additional threads beyond core size, up to **max pool size**.
4. Once both the queue is full *and* max pool size is reached, the **rejection policy** kicks in (`AbortPolicy` throws, `CallerRunsPolicy` runs the task on the calling thread itself as backpressure, `DiscardPolicy` silently drops it).

**The common misconfiguration this question is probing for:** setting a large queue capacity with a small core size — tasks will sit queued for a long time before the pool ever scales toward max, because step 3 (creating extra threads) only triggers once the queue is actually full. If bursty load needs faster scale-up, a smaller queue (or a `SynchronousQueue`, which has zero capacity and forces immediate thread creation up to max) gets there faster, at the cost of rejecting sooner once truly maxed out.

**Follow-ups to expect:**
- Why might `CallerRunsPolicy` be a better default than `AbortPolicy` for a service under load? (It throttles the *caller* instead of losing the task outright — the caller doing the work itself slows down whatever's submitting tasks in the first place, which is a natural backpressure signal rather than a silent drop or a thrown exception deep in a background thread.)
