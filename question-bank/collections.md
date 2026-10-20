# Collections

## Q: Why does HashMap resize at load factor 0.75, and what changed in Java 8+ for collision handling (treeification)?
**Asked/relevant on:** 2026-10-16 (warm-up)
**Your answer:**


**Refined answer:**

The load factor controls the trade-off between memory usage and lookup speed. A high load factor packs more entries into the bucket array before resizing — saving memory but increasing collisions, which degrades average O(1) `get`/`put` toward O(n) within an overloaded bucket. A low load factor keeps buckets sparse (fast) but wastes memory. **0.75** is the value the JDK authors settled on empirically as the best balance for typical workloads.

Concretely: once `size > capacity * loadFactor`, `HashMap` resizes — it doubles the internal table capacity and rehashes all entries — to keep the average chain length per bucket short.

**Java 8 change — treeification:** before Java 8, each bucket was a plain linked list, so a worst-case bucket (many keys hashing to the same bucket, e.g. via a poor or adversarial `hashCode()`) degraded to O(n) lookup. From Java 8 onward, if a bucket's linked list grows beyond `TREEIFY_THRESHOLD` (8 entries) **and** the table's overall capacity is at least 64, that bucket is converted into a red-black tree, turning worst-case lookup within that bucket into **O(log n)** instead of O(n). This specifically hardens `HashMap` against hash-flooding-style degradation. (If capacity is still below 64, `HashMap` prefers to resize rather than treeify; a treeified bucket converts back to a list if it shrinks below `UNTREEIFY_THRESHOLD`, 6 entries.)

This is also exactly why a key's class must override both `equals()` and `hashCode()` consistently: `hashCode()` determines the bucket, `equals()` determines correctness within that bucket (or the comparison used within a treeified bucket). Getting this contract wrong breaks lookups silently — `map.get(key)` can fail to find an entry that's actually present.

**Follow-ups to expect:**
- What's the time complexity of get() in the worst case pre- and post-Java 8, and why?
- Why must a key's class override both equals() and hashCode() consistently for HashMap to work correctly?

---

## Q: Why must equals() and hashCode() be overridden together? What happens if two different keys have the same hashCode()?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 1; asked repeatedly)
**Your answer:**

**Refined answer:**

`hashCode()` decides *which bucket* a key lands in; `equals()` decides *which entry within that bucket* actually matches. The contract: if two objects are `equals()`, they **must** have the same `hashCode()` (the reverse isn't required — different objects can share a hash code; that's an expected collision, not a bug). Breaking this contract — e.g. overriding `equals()` to compare by business ID but leaving the default (identity-based) `hashCode()` — means two "equal" objects can land in *different* buckets, so `map.get(key)` can fail to find an entry that's logically present, because it never even looks in the bucket where it actually lives.

**Two different keys with the same `hashCode()`** is not an error — it's a normal **collision**, which `HashMap` is explicitly designed to handle: both keys land in the same bucket, and `equals()` is then used to distinguish between them within that bucket (walking the bucket's linked list, or for a treeified bucket, the tree's comparison — see the load-factor/treeification entry above). The map still works correctly; it's just slightly slower for that bucket, which is exactly why a poor `hashCode()` (one that produces lots of collisions) degrades performance without causing incorrect results.

**Follow-ups to expect:**
- Why is it fine for `hashCode()` to be the same for unequal objects, but not fine for `equals()` to be true with different `hashCode()`s? (The former is just a collision, handled by design; the latter breaks the bucket-lookup invariant itself — the map can no longer guarantee it looks in the right place at all.)

---

## Q: Why is HashMap not thread-safe, and how does ConcurrentHashMap work internally?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**See:** `multithreading-concurrency.md` — answered there in full, since it's fundamentally a concurrency question (bucket-level locking + CAS) rather than a Collections-structure question. Cross-referenced here because it's commonly asked alongside HashMap internals.

---

## Q: ArrayList vs LinkedList — when do you choose each?
**Asked/relevant on:** 2026-10-20 (added — standard 6+ YOE staple, worth having cold even though not in the raw dump)
**Your answer:**

**Refined answer:**

`ArrayList` is backed by a contiguous array — O(1) random access by index, but inserting/removing from the middle is O(n) (everything after the index must shift). `LinkedList` is a doubly-linked list — O(1) insertion/removal at the ends (and O(1) once you already hold a reference to the node), but O(n) random access by index (must walk from a head/tail), and each element carries extra memory overhead (two pointers per node) compared to a plain array slot.

**The practical answer most interviewers want:** default to `ArrayList` — in practice, index-based access and iteration (the common case) are far more frequent than middle insertion/removal, and `ArrayList`'s cache-friendly contiguous memory layout tends to outperform `LinkedList` even on operations where `LinkedList`'s Big-O looks better, because of CPU cache locality. Reach for `LinkedList`/`Deque` implementations specifically when you need genuine queue/stack behavior (`ArrayDeque` is usually an even better choice for that than `LinkedList`, since it avoids the per-node pointer overhead).

---

## Q: What's the difference between a fail-fast and a fail-safe iterator? What causes ConcurrentModificationException?
**Asked/relevant on:** 2026-10-20 (added — common follow-up when discussing Collections + concurrency)
**Your answer:**

**Refined answer:**

A **fail-fast** iterator (the default for `ArrayList`, `HashMap`, etc.) detects structural modification to the underlying collection *during* iteration (tracked via an internal `modCount` that the iterator checks on every `next()`) and throws `ConcurrentModificationException` immediately rather than risking undefined/corrupted iteration behavior — it's a deliberate fail-loud safety net, not a bug. This can happen even single-threaded (removing an element directly from a list while iterating it with a `for-each` loop, instead of via `Iterator.remove()`).

A **fail-safe** iterator (e.g. `CopyOnWriteArrayList`, or iterating a `ConcurrentHashMap`) operates on a snapshot or tolerates concurrent structural changes without throwing — but then it may not reflect modifications made during the iteration at all, which is its own trade-off (you might miss an element added mid-iteration, not just avoid an exception).

**Follow-ups to expect:**
- How do you safely remove elements while iterating an `ArrayList`? (`Iterator.remove()`, or `list.removeIf(predicate)` — both update the iterator's internal bookkeeping correctly, unlike a direct `list.remove(element)` call inside a for-each loop.)
