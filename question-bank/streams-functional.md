# Streams & Functional

## Q: map vs flatMap — real difference, and a scenario where a plain for-loop beats a stream?
**Asked/relevant on:** 2026-10-16 (warm-up)
**Your answer:**


**Refined answer:**

`map` and `flatMap` both transform a stream, but differ in cardinality.

`map` takes a `Function<T, R>` and produces exactly one output element per input element — a one-to-one transformation:

```java
List<String> names = List.of("Alice", "Bob");
List<Integer> lengths = names.stream().map(String::length).toList(); // [5, 3]
```

`flatMap` takes a `Function<T, Stream<R>>` — each input element is itself turned into a stream, and all of those streams are flattened into a single resulting stream. It's used whenever elements are nested collections and a single flat stream is needed:

```java
List<Order> orders = ...; // each Order has a List<Item> items
List<Item> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .toList();
```

Using `map` here would produce a `Stream<List<Item>>` (still nested); `flatMap` collapses it to `Stream<Item>`.

**Where a plain for-loop beats a stream:**
- **Complex control flow** — multiple conditional early exits (`break`/`continue` at different points) are awkward or impossible to express cleanly in a stream pipeline.
- **Checked exceptions** — a lambda body can't throw a checked exception without wrapping it, which gets noisy inside stream operations.
- **Performance-critical hot paths** — stream pipelines add lambda/boxing overhead; in a tight, latency-sensitive loop, a plain indexed for-loop avoids that cost.
- **Debuggability** — stepping through a multi-stage stream pipeline is harder than stepping through an imperative loop.

Example: iterating a primitive `int[]` in a latency-sensitive calculation — a plain `for (int i = 0; i < arr.length; i++)` avoids boxing and lambda-call overhead that even `IntStream` can introduce on the hottest paths.

**Follow-ups to expect:**
- Are Java streams lazy or eager? What does that mean for a stream pipeline with a filter followed by a side-effecting peek()?
- When would parallelStream() actually make things slower?

---

## Q: "Explain Java Streams. When do Streams introduce unnecessary overhead?" (Wells Fargo round)
**See:** this is the same question as the entry above (map vs flatMap, for-loop vs stream) from a different company's phrasing — don't duplicate the answer, just note it was also asked here. Streams are lazy: intermediate operations (`map`, `filter`) build up a pipeline description but don't run anything until a terminal operation (`collect`, `forEach`, `toList`) is invoked — which is also why `peek()` as a side-effect mechanism is fragile (its execution order/frequency depends on what terminal operation and intermediate optimizations are involved; it's not a reliable place to put real logic).

---

## Q: Comparable vs Comparator — when do you use each?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

**`Comparable<T>`** is implemented *by the class itself* (`compareTo()`), defining that type's single **natural ordering** — appropriate when one default ordering genuinely makes sense for every use of the class (e.g. sorting employees by ID, or dates chronologically).

**`Comparator<T>`** is an *external* strategy (`compare()`), defining a sort order **without modifying the class** — appropriate when there are multiple valid, context-dependent orderings (sort transactions by date in one screen, by amount in another, by account elsewhere). Comparators compose well with lambdas/method references (`Comparator.comparing(Trade::getAmount).reversed().thenComparing(Trade::getTradeDate))`), which is usually cleaner than trying to force every possible ordering into one `compareTo()`.

**One sharp edge worth knowing:** for `TreeSet`/`TreeMap`, the comparison result (not `equals()`) determines which elements/keys are considered duplicates — a `Comparator`/`compareTo()` that returns `0` for two objects that aren't actually `.equals()` will cause one of them to be silently dropped from the set/map.

**How this maps to your project:** `Trade` could reasonably implement `Comparable` for a natural ordering by `tradeDate`, while `settlement-service`'s reporting views use separate `Comparator`s (by amount, by counterparty, by status) without touching the `Trade` class itself each time a new report needs a different order.

---

## Q: What is a Lambda Expression? What is a Functional Interface? Functional Interface vs a normal interface? Anonymous class vs Lambda?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round; foundational Java 8 group)
**Your answer:**

**Refined answer:**

A **functional interface** is any interface with exactly **one abstract method** (it can still have default/static methods) — e.g. `Runnable`, `Comparator<T>`, or a custom one like `@FunctionalInterface interface RiskRule { boolean check(Trade t); }`. This single-method constraint is what makes it possible to provide an implementation inline as a **lambda expression** (`trade -> trade.getAmount() > 0`) — the lambda's parameters and body match that one abstract method, so the compiler knows exactly what it's implementing without naming a class at all.

A **normal interface** can have any number of abstract methods, so there's no single method a lambda could unambiguously implement — lambdas only work against functional interfaces for exactly this reason.

**Anonymous class vs lambda** — both let you provide an implementation inline without a named top-level class, but an anonymous class can implement an interface with *multiple* methods (or extend a class), carries its own `this` (referring to itself, not the enclosing instance), and is more verbose (`new Comparator<Trade>() { public int compare(...) {...} }`). A lambda is restricted to functional interfaces, is syntactically lighter, and captures the *enclosing* `this`/variables directly (effectively-final local variables) rather than having its own identity.

**How this maps to your project:** the risk-rule pipeline (`risk-service`, Strategy + Chain of Responsibility in `design-patterns-checklist.md`) is a natural fit for a small custom functional interface per rule type, with simple rules expressed as lambdas and more complex ones as full classes when they need their own state/dependencies.

---

## Q: What are the important Java 8 features, at a high level?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round)
**Your answer:**

**Refined answer (the headline list — each already has its own deeper entry elsewhere in this bank):** lambda expressions + functional interfaces (above), the Streams API (map/flatMap/collect — see above), `Optional<T>` (an explicit container for "may be absent," reducing accidental `NullPointerException`s versus returning `null` directly), default/static methods on interfaces (see `core-java-oops.md`'s diamond-problem entry), and the new `java.time` package (`LocalDate`/`LocalDateTime`/`Instant` — immutable, thread-safe date/time types replacing the old mutable, not-thread-safe `Date`/`Calendar`).

