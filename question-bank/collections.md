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
