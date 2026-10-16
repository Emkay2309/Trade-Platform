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
