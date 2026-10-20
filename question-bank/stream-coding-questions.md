# Java Streams — Coding Questions

Classic Stream-based coding questions asked in interviews, each with working Java code and a plain-language walkthrough of *why* it's built that way — not just the code, so you can explain the reasoning if the interviewer changes the input slightly.

## Q1: Find duplicate elements in a list
```java
List<Integer> nums = List.of(1, 2, 3, 2, 4, 3, 5);

Set<Integer> seen = new HashSet<>();
List<Integer> duplicates = nums.stream()
    .filter(n -> !seen.add(n))   // add() returns false if it was ALREADY in the set
    .distinct()
    .toList();
// [2, 3]
```
**Logic in plain words:** `Set.add(n)` returns `true` the first time you see a value, and `false` every time after — because the set already has it. So `!seen.add(n)` is `false` the first time (keep going, don't filter it in yet) and `true` every repeat time (this is a duplicate — filter it in). `.distinct()` at the end just collapses "appeared 3 times" down to one entry in the output.

---

## Q2: Count the frequency of each word in a list
```java
List<String> words = List.of("apple", "banana", "apple", "orange", "banana", "apple");

Map<String, Long> frequency = words.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),      // group by the word itself
        Collectors.counting()     // count how many landed in each group
    ));
// {apple=3, banana=2, orange=1}
```
**Logic in plain words:** `groupingBy` is "put every element into a bucket based on some key." `Function.identity()` means "the bucket key IS the word itself" (no transformation). `Collectors.counting()` is the *downstream* collector — instead of collecting each bucket's actual elements into a list, just count how many elements landed there. This two-argument `groupingBy` pattern (classifier + downstream collector) is the one to remember — the same shape answers "group by X, then do Y to each group" for almost any Y.

---

## Q3: Group employees by department, then get just their names
```java
record Employee(String name, String department, double salary) {}

List<Employee> employees = List.of(
    new Employee("Asha", "Engineering", 90000),
    new Employee("Ravi", "Engineering", 85000),
    new Employee("Meera", "Sales", 70000)
);

Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
// {Engineering=[Asha, Ravi], Sales=[Meera]}
```
**Logic in plain words:** same `groupingBy` shape as Q2, but now the downstream collector is `mapping(extractFunction, collector)` — "for each group, first transform every element with `extractFunction` (pull out just the name), then collect those transformed values with `collector`." This is the pattern anytime you want to group by one field but the output group shouldn't contain the *whole* object.

---

## Q4: Find the employee with the highest salary in each department
```java
Map<String, Optional<Employee>> topEarnerByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));
```
**Logic in plain words:** `maxBy(comparator)` as a downstream collector means "within each group, keep only the single element the comparator says is biggest." It returns `Optional<Employee>` (not `Employee` directly) because a group is only guaranteed non-empty if it came from `groupingBy` on real data — `Optional` is the honest way to represent "there is one," rather than risking a `null`.

---

## Q5: Flatten a list of lists into one list (flatMap)
```java
List<List<Integer>> nested = List.of(
    List.of(1, 2, 3),
    List.of(4, 5),
    List.of(6)
);

List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .toList();
// [1, 2, 3, 4, 5, 6]
```
**Logic in plain words:** `map(List::stream)` alone would give you a `Stream<Stream<Integer>>` — a stream of streams, still nested. `flatMap` does the same transformation but then automatically unwraps/merges all those inner streams into one single flat stream. Whenever each element "contains more elements you actually want at the top level," reach for `flatMap`, not `map`.

---

## Q6: Partition a list into two groups based on a condition (pass/fail, even/odd)
```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8);

Map<Boolean, List<Integer>> partitioned = nums.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {false=[1, 3, 5, 7], true=[2, 4, 6, 8]}
```
**Logic in plain words:** `partitioningBy` is a special case of `groupingBy` where the key can only ever be `true` or `false` — so the result map always has exactly two entries (even if one is empty), unlike `groupingBy` which only creates keys that actually occurred. Use `partitioningBy` specifically when the split is a yes/no condition; use `groupingBy` when there could be more than two buckets.

---

## Q7: Convert a List of objects into a Map, handling duplicate keys
```java
List<Employee> employees = List.of(
    new Employee("Asha", "Engineering", 90000),
    new Employee("Ravi", "Engineering", 85000)
);

Map<String, Double> salaryByName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        Employee::getSalary,
        (existing, incoming) -> existing   // merge function: what to do if a key repeats
    ));
```
**Logic in plain words:** `toMap(keyFn, valueFn)` throws `IllegalStateException` the moment it hits a *second* element with a key it's already seen — it has no idea what you want to do about the collision, so it refuses to guess. The **third argument** (a `BinaryOperator<V>` merge function) tells it exactly what to do: here, "keep whichever one I saw first" (`existing`). Forgetting this third argument is the single most common reason `toMap` blows up in real code with non-unique keys — always ask "could this key repeat?" before using the two-argument form.

---

## Q8: Sum and average of a numeric field across a list
```java
double totalSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .sum();

double avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average()
    .orElse(0.0);
```
**Logic in plain words:** `mapToDouble` (and its siblings `mapToInt`/`mapToLong`) convert a `Stream<Employee>` into a primitive `DoubleStream` — this matters because `DoubleStream` has `sum()`/`average()`/`max()`/`min()` built in directly (no boxing overhead, no need for a separate `Collectors.summingDouble`). `average()` returns `OptionalDouble` because averaging an empty stream is undefined — `.orElse(0.0)` is the explicit decision for what "no data" should mean here, rather than letting it silently become `NaN` or throw.

---

## Q9: Sort a list of objects by multiple fields (e.g. department, then salary descending)
```java
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparing(Employee::getDepartment)
        .thenComparing(Employee::getSalary, Comparator.reverseOrder()))
    .toList();
```
**Logic in plain words:** `Comparator.comparing(...)` builds the *primary* sort key; `.thenComparing(...)` is the tie-breaker — only consulted when the primary comparator says two elements are equal (same department here). `Comparator.reverseOrder()` flips that specific tie-breaker to descending, while the primary key (department) stays ascending — each `.thenComparing()` can independently be ascending or descending, which is exactly what multi-column "ORDER BY department, salary DESC" means in SQL terms.

---

## Q10: Second-highest distinct value in a list, generically (not just salaries — see `coding-dsa.md` for the per-department version)
```java
List<Integer> values = List.of(50, 90, 90, 80, 70);

Optional<Integer> secondHighest = values.stream()
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
// Optional[80]
```
**Logic in plain words:** `distinct()` first removes the duplicate `90` so it doesn't occupy both the "highest" and "second-highest" slots. `sorted(reverseOrder())` puts the biggest value first. `skip(1)` throws away that first (highest) element. `findFirst()` grabs whatever's now at the front — the second-highest. Returns `Optional` because a list with fewer than 2 distinct values has no valid answer, and that has to be representable, not silently wrong.

**Follow-up interviewers love:** "now do it without sorting the whole list" — track two running variables (`highest`, `secondHighest`) in a single pass with `reduce`, updating both as you go; this is O(n) instead of O(n log n), worth mentioning even if you implement the simpler sorted version first.
