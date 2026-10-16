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
