# Core Java & OOP

## Q: Java doesn't support multiple inheritance of classes but does of interfaces — what problem does diamond inheritance actually cause, and how do Java 8 default methods partially reintroduce it? How is it resolved?
**Asked/relevant on:** 2026-10-16 (warm-up)
**Your answer:**


**Refined answer:**

Java restricts a class to a single superclass specifically to avoid the classic diamond problem: if class `C` could inherit from both `A` and `B`, and both defined a method `foo()` with conflicting implementations (or conflicting instance state), there would be no unambiguous way to decide which version `C` should use. By allowing only one superclass, Java guarantees every inherited method and field has exactly one unambiguous origin.

Interfaces, by contrast, historically only declared method signatures with no implementation, so implementing several interfaces never created a runtime conflict — there was nothing to collide over.

Java 8 introduced **default methods**, giving interfaces a method body, and this reintroduces a *limited* version of the diamond problem: if interfaces `A` and `B` both declare a default method `foo()` with different implementations, and a class `C` implements both without overriding `foo()`, the compiler raises a hard error:

```
class C inherits unrelated defaults for foo() from types A and B
```

This is a **compile-time error**, not a silent runtime ambiguity — the key difference from C++-style multiple inheritance. Java forces the conflict to be resolved before the code can even build.

**Resolution rules, in order:**
1. A concrete method in a class always wins over any default method from an interface.
2. If one interface extends another and overrides the default method, the more specific interface's version wins.
3. Otherwise, the implementing class must explicitly override the method — and can still choose to delegate to a specific parent:

```java
interface A { default void foo() { System.out.println("A"); } }
interface B { default void foo() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void foo() {
        A.super.foo(); // explicitly resolve which default to use
    }
}
```

**Follow-ups to expect:**
- What happens if two interfaces give the same default method and a class implements both without overriding?
- Why doesn't this same ambiguity happen with fields?

---

## Q: Interface vs Abstract class — when do you choose one over the other?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — asked repeatedly across multiple companies)
**Your answer:**

**Refined answer:**

An abstract class can hold shared state (instance fields) and a mix of implemented + unimplemented methods, but a class can only extend *one* abstract class. An interface defines a contract (plus, since Java 8, default/static methods), has no instance state, and a class can implement *many* of them. Choose an abstract class when subclasses share actual implementation/state ("is-a" with common behavior to inherit); choose an interface when you're defining a capability unrelated classes can plug into ("can-do" — e.g. `Comparable`, `Serializable`, or this project's `PaymentStrategy`-style contracts), especially when a class may need to satisfy several unrelated contracts at once.

**Follow-ups to expect:**
- Since Java 8 default methods let interfaces carry implementation too, what can an abstract class do that an interface still can't? (Hold instance fields/constructor logic, and control member visibility — `private`/`protected` members — which interfaces still can't fully replicate.)

---

## Q: Why is String immutable in Java?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PwC round 1)
**Your answer:**

**Refined answer:**

Several reasons compound: **security** (a `String` used as a file path or network address can't be mutated after a security check but before use); **safe sharing/caching** via the **String Pool** (`String` literals are interned and reused — only safe because nothing can change one string and corrupt every other reference pointing at the same pooled instance); **hashcode caching** (since the value never changes, `String` caches its `hashCode()` on first computation, making repeated `HashMap` lookups on string keys cheap); and **thread-safety** (an immutable object can be freely shared across threads with no synchronization needed, since there's nothing to race on).

**Follow-ups to expect:**
- See the next entry (String Pool / `new String(...)`) — it's the direct continuation of this question.

---

## Q: String Pool vs heap memory — what happens with `new String("hello")` if `"hello"` already exists in the pool? Difference between `==` and `.equals()` for Strings?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A string literal (`"hello"`) is placed in the **String Pool** (part of the heap since Java 7, previously PermGen) — if an identical literal already exists there, the JVM reuses that single instance rather than creating a new one (this is only safe *because* strings are immutable). `new String("hello")`, however, explicitly forces creation of a **new object on the regular heap**, even though `"hello"` the literal already exists in the pool — so you end up with two distinct objects: the pooled one, and this new heap one.

`==` compares **references** (are these two variables pointing at the exact same object) — so `new String("hello") == "hello"` is `false`, even though their content is identical. `.equals()` compares **content**, so `new String("hello").equals("hello")` is `true`. This is precisely the classic trap this question is testing: `==` on strings looks like it's comparing values but it's actually comparing identity.

**Follow-ups to expect:**
- How would you force the `new String(...)` instance back into the pool? (`.intern()`.)

---

## Q: Abstraction vs Encapsulation — what's the actual difference?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round)
**Your answer:**

**Refined answer:**

**Abstraction** is about *what* is exposed — hiding complexity behind a simpler interface, so a caller deals with "what it does" rather than "how it does it" (an interface/abstract class is the typical Java tool for this). **Encapsulation** is about *how* it's protected — bundling data with the methods that operate on it and controlling access (private fields + public getters/methods), so internal state can't be corrupted from outside. They're often conflated because the same class design usually does both at once, but abstraction is a design-level concept (simplifying what's visible) while encapsulation is an access-control mechanism (restricting what's reachable).

---

## Q: Exception handling in Java — checked vs unchecked, and how do you design exceptions well?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round)
**Your answer:**

**Refined answer:**

**Checked exceptions** (extend `Exception`, not `RuntimeException`) must be either caught or declared (`throws`) — the compiler forces the caller to acknowledge them. Use for recoverable conditions a caller is expected to handle (e.g. a file not found that the caller should retry/prompt for). **Unchecked exceptions** (extend `RuntimeException`) don't require declaration — use for programming errors/invariant violations the caller generally can't meaningfully recover from at the call site (`NullPointerException`, `IllegalArgumentException`, or this project's own `TradeValidationException` if it represents a genuine bug rather than an expected business outcome).

In a service codebase, the practical pattern is usually: throw unchecked exceptions for business/validation errors (so they don't force every intermediate layer to declare `throws`), and let a centralized `@RestControllerAdvice` (see `spring-boot.md`) map them to the right HTTP status — rather than scattering try/catch everywhere.

**Follow-ups to expect:**
- Why do many modern Java codebases avoid checked exceptions almost entirely? (They don't compose well with lambdas/streams, and force every layer in a call chain to either handle or re-declare them, which often just becomes boilerplate rather than genuine handling.)

---

## Q: Java 11 vs Java 17 — key differences?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round)
**Your answer:**

**Refined answer:**

Both are LTS releases. Notable additions between them: **records** (concise immutable data carriers — `record Trade(String id, BigDecimal amount) {}`, generating constructor/accessors/`equals`/`hashCode`/`toString` automatically), **sealed classes** (restrict which classes may extend/implement a type — useful for exhaustive pattern matching over a known, closed set of subtypes), **pattern matching for `instanceof`** (`if (obj instanceof Trade t) { ... t ... }` without a separate cast), **text blocks** (multi-line string literals), and continued performance/GC improvements (e.g. further ZGC maturation). For this project, records are the most immediately useful — a good fit for simple value objects/DTOs that don't need the Builder pattern's optional-field flexibility.

---

## Q: Explain Java Memory Management / the JVM memory areas. What is the PC Register?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AccelGrowth round)
**Your answer:**

**Refined answer:**

- **Heap** — shared across all threads; where objects live. Divided into generations (Young: Eden + Survivor spaces, Old/Tenured) for generational garbage collection.
- **Stack** — one per thread; holds method call frames (local variables, partial results, the call chain). Not shared — this is why local variables are inherently thread-safe without synchronization.
- **Metaspace** (replaced PermGen since Java 8) — class metadata (loaded class definitions, method info).
- **PC (Program Counter) Register** — one per thread; holds the address of the JVM instruction the thread is currently executing. Essential for thread context switching — when a thread is paused and resumed, the PC register is exactly what lets it resume at the right instruction.
- **Native method stacks** — for native (JNI) method calls, separate from the regular Java stack.

---

## Q: What happens internally when a Java class is loaded?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Three phases: **Loading** (the ClassLoader reads the `.class` file's bytecode and creates a `Class` object representing it in memory), **Linking** (verification of bytecode correctness, preparation of static fields with default values, and resolution of symbolic references to other classes), and **Initialization** (static initializer blocks run, static fields are assigned their actual initial values — this happens lazily, on first active use of the class, not at JVM startup for every class). Class loading follows **delegation** — a ClassLoader asks its parent to try loading first before attempting itself (so, e.g., `java.lang.String` always resolves to the JDK's own class, never something a user accidentally puts on the classpath with the same name).

---

## Q: ClassLoader vs Class.forName() — what's the relationship?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round)
**Your answer:**

**Refined answer:**

A `ClassLoader` is the mechanism that actually loads class bytecode into the JVM (see the class-loading entry above — Bootstrap → Platform → Application, delegating upward). `Class.forName("com.example.Foo")` is a *way to trigger* that loading dynamically, by fully-qualified name, at runtime — and critically, unlike simply referencing a class in code, `Class.forName(...)` **initializes** the class (runs static initializers) immediately by default, whereas `ClassLoader.loadClass(...)` alone loads it *without* necessarily initializing it. This distinction matters for JDBC drivers historically registering themselves via a static block — `Class.forName("com.mysql.cj.jdbc.Driver")` was the idiom specifically because it guarantees that static registration block actually runs.

---

## Q: How does the JVM decide an object is eligible for Garbage Collection? What would you check first for OutOfMemoryError or long GC pauses?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

An object becomes GC-eligible once it's **unreachable** from any GC root (active thread stacks, static fields, JNI references) — not based on reference *count*, but reachability, which is why Java doesn't have the cycle problems naive reference-counting would (two objects only referencing each other, with nothing else pointing to either, are still correctly collected).

**For `OutOfMemoryError`, check first:**
- Heap dump analysis (`jmap`/a profiler) for what's actually retaining memory — usually a collection that keeps growing (a cache with no eviction, a listener list that's never cleaned up) rather than "too much data" in any single object.
- Whether it's actually heap exhaustion (`OutOfMemoryError: Java heap space`) vs. something else entirely like unbounded thread creation (`unable to create new native thread` — see the `@Async`/`SimpleAsyncTaskExecutor` entry in `spring-boot.md`) or Metaspace exhaustion (too many classes loaded, common with dynamic class generation/classloader leaks).

**For long GC pauses:** check which collector is in use and the generation sizes — frequent Young-gen collections are normal and cheap; long pauses usually mean either the Old generation is filling up and triggering expensive full GCs, or heap sizing/collector choice doesn't match the allocation rate. `-Xlog:gc` (or a GC log analyzer) shows pause frequency/duration directly rather than guessing.

---

## Q: How does Java's ServiceLoader mechanism work? What are Method Handles? How would you implement a custom Spliterator? How would you handle serialization when a class evolves across versions?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round; grouped — niche/advanced, lower priority unless this specific company comes up again)
**Your answer:**

**Refined answer (brief — these are advanced/niche; deepen only if a specific interview loop calls for it):**

- **`ServiceLoader`** — Java's built-in plugin/SPI mechanism: a provider implementation registers itself by name in a `META-INF/services/<InterfaceName>` file, and `ServiceLoader.load(SomeInterface.class)` discovers and instantiates all registered implementations at runtime, without the consuming code knowing concrete classes in advance. Conceptually the JDK-native version of the same Strategy/Factory idea already used for `PaymentStrategy` above.
- **Method Handles** (`java.lang.invoke`) — a lower-level, more performant alternative to reflection for invoking methods/constructors dynamically, resolved and type-checked mostly at the point you create the handle rather than on every call; the mechanism backing Java's `invokedynamic` and lambda implementation under the hood.
- **Custom `Spliterator`** — implement `tryAdvance` (process one element, return whether more remain) and `trySplit` (split the remaining elements into two parts for parallel processing) to make an arbitrary data source stream-able, including in parallel streams.
- **Serialization across versions** — declare a stable `serialVersionUID` explicitly (don't rely on the compiler-generated one, which changes if the class shape changes); for genuinely evolving schemas, prefer a format with explicit schema versioning (JSON/Avro/Protobuf) over raw Java serialization, which is fragile exactly in this scenario and generally discouraged for anything long-lived or cross-version.

**How this maps to your project:** none of these are central to the Trade Platform's architecture — flag if a specific interview loop asks for depth here and we'll build a small targeted exercise rather than over-indexing on them now.

---

## Q: Explain the SOLID principles, with a real example of each.
**Asked/relevant on:** 2026-10-20 (added — underlies several questions already in the bank, e.g. the Strategy-pattern/OCP entry in `microservices-system-design.md`, but was never named as a full set)
**Your answer:**

**Refined answer:**

- **S — Single Responsibility:** a class should have one reason to change. A `TradeService` that both validates business rules *and* formats email notifications has two unrelated reasons to change — split them.
- **O — Open/Closed:** open for extension, closed for modification — adding new behavior shouldn't require editing existing, tested code. The `PaymentStrategy`/risk-rule examples already in this bank are exactly this (new rule = new class, not an edited `switch`).
- **L — Liskov Substitution:** a subclass must be usable anywhere its superclass/interface is expected, without surprising behavior. A classic violation: a `Square extends Rectangle` that overrides `setWidth`/`setHeight` to keep both equal — code that treats any `Rectangle` generically and sets width/height independently breaks when handed a `Square`.
- **I — Interface Segregation:** prefer several small, focused interfaces over one large one that forces implementers to provide methods they don't need. A `Worker` interface with both `code()` and `designUI()` forces a backend-only implementer to stub out a method it has no real implementation for.
- **D — Dependency Inversion:** depend on abstractions, not concrete implementations — high-level code (`TradeService`) should depend on a `RiskClient` interface, not a concrete `HttpRiskClient` class, so the implementation can be swapped (or mocked in tests) without changing the dependent code. This is exactly what makes Spring's dependency injection useful rather than just ceremony.

**How this maps to your project:** D and O are the two you'll lean on constantly — every inter-service client in this project should be an interface the calling service depends on (D), and every extensible rule set (risk rules, payment/settlement strategies) should be structured so new cases are new classes (O).

