# Spring Boot & JPA/Hibernate

## Q: What is the N+1 query problem in Hibernate/JPA, and how do you solve it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — asked by PayPal, PwC, and general Spring posts; very high frequency)
**Your answer:**

**Refined answer:**

The N+1 problem happens when Hibernate fetches a list of N parent entities with one query, then — because a related entity is lazily loaded — issues one **additional** query per parent to fetch its related data. Fetching 500 employees and then accessing each one's department (lazy `@ManyToOne`) results in 1 query for employees + 500 queries for departments = **501 queries** for what should be a single join.

**How to detect it:**
- Enable SQL logging (`spring.jpa.show-sql=true`, `logging.level.org.hibernate.SQL=DEBUG`) and watch for the same query shape repeating with different IDs.
- Symptom: API response time grows roughly linearly with record count, even though the "logic" is simple.

**How to fix it:**
- **`JOIN FETCH`** in a JPQL query — fetches the parent and the association in one SQL query:
```java
@Query("SELECT e FROM Employee e JOIN FETCH e.department")
List<Employee> findAllWithDepartment();
```
- **`@EntityGraph`** — declaratively specify which associations to eagerly fetch for a given query, without changing the entity's default fetch type globally.
- **Batch fetching** (`@BatchSize` / `hibernate.default_batch_fetch_size`) — when you can't avoid N queries entirely, Hibernate groups them into fewer `IN (...)` queries instead of one-by-one.

**How this maps to your project:** `reference-data-service` and `trade-capture-service` will have exactly this shape — a `Trade` referencing `Instrument`/`Counterparty` reference data. When we build the trade listing endpoint (Phase 1), we'll deliberately start with a lazy association, observe the N+1 in the SQL log, then fix it with `JOIN FETCH` — so you have a real, reproducible before/after you built yourself, not a borrowed anecdote.

**Follow-ups to expect:**
- Why not just make every association `EAGER` to avoid this? (Answer: eager loading pulls data you may not need on every query, and can itself cause unnecessary joins/cartesian products when multiple collections are eager.)
- What's the difference between `JOIN FETCH` and a regular `JOIN` in JPQL?

---

## Q: @Component vs @Service vs @Repository vs @Controller vs @RestController — what's the real difference?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch; PwC round 3)
**Your answer:**

**Refined answer:**

All five are Spring stereotype annotations — functionally, Spring registers each as a bean the same way. The difference is **semantic intent** (and a couple of real behavioral extras):

- **`@Component`** — generic "this is a Spring-managed bean" marker. Use for things that don't fit a more specific role (validators, utility beans).
- **`@Service`** — marks a class holding business logic/orchestration. No extra framework behavior beyond `@Component`, but it documents intent and lets tools/AOP target the service layer specifically.
- **`@Repository`** — marks the data-access layer. It additionally enables **persistence exception translation**: Hibernate/JDBC-specific exceptions get converted into Spring's unified `DataAccessException` hierarchy, so callers don't need to catch provider-specific exceptions. With Spring Data JPA, repository interfaces are usually auto-implemented, so you rarely hand-write a `@Repository` class.
- **`@Controller`** — handles web requests (Spring MVC), typically returning a view name (e.g. a Thymeleaf template).
- **`@RestController`** — `@Controller` + `@ResponseBody` combined: return values are serialized straight into the HTTP response body (JSON), not resolved as a view name. This is what you'd use for all the REST APIs in this project.

**How this maps to your project:** every service module will have this exact layering — e.g. in `trade-capture-service`: `TradeController` (`@RestController`), `TradeService` (`@Service`), `TradeRepository` (`@Repository`, Spring Data JPA interface).

**Follow-ups to expect:**
- If you autowire by type and have two beans of the same interface, how do you disambiguate? (`@Qualifier`, `@Primary`)
- Does `@Repository`'s exception translation work automatically, or does something need to be configured? (Needs a `PersistenceExceptionTranslationPostProcessor`, which Spring Boot auto-configures for you.)

---

## Q: How does @Transactional actually work?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch; PwC round 3; AccelGrowth)
**Your answer:**

**Refined answer:**

`@Transactional` works via a **proxy** — Spring wraps the bean in a dynamic proxy (or CGLIB subclass) at startup. When you call an `@Transactional` method *from outside the bean*, the call goes through the proxy first, which opens a transaction, invokes the real method, then commits or rolls back based on whether an exception was thrown.

Key consequence — **the same self-invocation trap as `@Async`**: if a method inside the same class calls another `@Transactional` method via `this.otherMethod()`, the proxy is bypassed entirely, and no transaction boundary is applied to that inner call.

By default, a **runtime (unchecked) exception** triggers a rollback; a **checked exception** does not, unless you declare `@Transactional(rollbackFor = SomeCheckedException.class)`.

**How this maps to your project:** `settlement-service`'s state transitions will be `@Transactional` — e.g. moving a trade from `PENDING` to `SETTLED` should roll back cleanly if any step fails partway through, and this is exactly where the self-invocation trap could bite if we're not careful about which class calls which method.

**Follow-ups to expect:**
- What's the default propagation behavior (`REQUIRED`), and when would you use `REQUIRES_NEW`?
- What happens if you catch an exception inside an `@Transactional` method and don't rethrow it? (Spring never sees the exception, so it commits as if nothing happened — a classic silent bug.)

---

## Q: What is Circular Dependency in Spring, and how do you resolve it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch; PwC-style rounds)
**Your answer:**

**Refined answer:**

A circular dependency happens when bean A needs bean B injected, and bean B needs bean A injected — Spring can't fully construct either one first via constructor injection, since each requires the other to already exist. With pure constructor injection, this fails at startup with a `BeanCurrentlyInCreationException`.

**Resolutions, roughly in order of preference:**
1. **Redesign** — a circular dependency between two services is usually a sign they should be merged, or a piece of shared logic should be extracted into a third bean both depend on. This is almost always the *right* fix, not a workaround.
2. **`@Lazy`** on one of the injected dependencies — defers resolving that dependency until it's actually used, breaking the construction-time cycle.
3. **Setter/field injection** instead of constructor injection for one side — Spring can construct both beans first, then wire the circular reference afterward. (Field injection is generally discouraged for other reasons — harder to test, hides dependencies — so treat this as a last resort, not a default.)

**How this maps to your project:** worth watching for between `risk-service` and `settlement-service` if they ever need to call back into each other — a sign we should extract a shared contract/interface instead.

**Follow-ups to expect:**
- Why does field injection "solve" this when constructor injection doesn't?
- Is a circular dependency ever acceptable, or always a design smell? (Treat it as a design smell — in a microservices project at this level, it should never be "acceptable," just redesigned.)

---

## Q: @Async pitfalls — why did my Spring Boot app crash with OutOfMemoryError after adding @Async?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Three separate traps compound here:

1. **No default thread pool ceiling.** Without an explicit executor, Spring uses `SimpleAsyncTaskExecutor` — despite the name, it is *not* a pool. It spins up a brand-new OS thread for every single `@Async` call and never reuses them. Under real traffic (thousands of concurrent calls), this creates thousands of OS threads and the JVM dies with `OutOfMemoryError: unable to create new native thread`.
2. **The same-class proxy trap as `@Transactional`.** `@Async` also relies on a Spring proxy. Calling an `@Async` method on `this` from inside the same class bypasses the proxy — it runs synchronously, silently, with no error, which is often more dangerous than a crash because nothing looks wrong.
3. **The fix — a bounded executor:**
```java
@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);      // steady worker count
    executor.setMaxPoolSize(50);       // extra workers under burst load
    executor.setQueueCapacity(100);    // requests wait here once core is busy
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy()); // slow the caller down instead of crashing
    return executor;
}
```

**How this maps to your project:** `notification-audit-service` is the natural place for `@Async` (writing audit events shouldn't block the caller) — this is exactly the executor config we'll wire there in Phase 3, and a good place to deliberately trigger the unbounded-executor failure first so you've actually seen the crash before fixing it.

**Follow-ups to expect:**
- What does `CallerRunsPolicy` actually do, and why is it safer than the default `AbortPolicy` under load?
- Would Java 21 virtual threads make a custom executor unnecessary here? (Partially — see the Virtual Threads entry in `multithreading-concurrency.md`; virtual threads remove the "thread is expensive" problem but you'd still want bounding/backpressure for downstream protection.)

---

## Q: Global exception handling in Spring Boot — how do you implement it?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Use `@RestControllerAdvice` with `@ExceptionHandler` methods to centralize exception-to-HTTP-response mapping across all controllers, instead of try/catch in every endpoint:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(TradeValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(TradeValidationException ex) {
        return ResponseEntity.badRequest().body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error"));
    }
}
```
This keeps error response format consistent across the whole API surface, and keeps controllers focused on the happy path.

**How this maps to your project:** every service gets one `GlobalExceptionHandler` — worth building this in `trade-capture-service` as soon as validation is added (Phase 0/1), so the pattern is established before more services exist.

**Follow-ups to expect:**
- How do you avoid leaking internal details (stack traces, SQL) in the error response while still logging them server-side?
- How would this interact with validation errors from `@Valid` (`MethodArgumentNotValidException`)?

---

## Q: Lazy vs Eager fetch types in JPA — how do you decide?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

`FetchType.LAZY` loads an association only when it's actually accessed (default for `@OneToMany`/`@ManyToMany`); `FetchType.EAGER` loads it immediately with the parent (default for `@ManyToOne`/`@OneToOne`). Lazy avoids pulling data you might not need; eager avoids extra round-trips for data you almost always need together — but indiscriminate eager loading is how you accidentally create N+1-adjacent problems or expensive cartesian-product joins when multiple collections are eager on the same entity.

Practical rule: default to `LAZY` everywhere, and deliberately fetch what you need per-query with `JOIN FETCH`/`@EntityGraph` rather than changing the mapping's global fetch type.

**Follow-ups to expect:**
- What is `LazyInitializationException` and when do you hit it? (Accessing a lazy association after the Hibernate session/transaction that loaded the parent has closed — common when a DTO mapping happens outside the transactional boundary.)

---

## Q: @RequestParam vs @PathVariable?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

`@PathVariable` binds a value from the URI path itself — part of the resource's identity (`GET /trades/{tradeId}`). `@RequestParam` binds a value from the query string — optional filters/parameters (`GET /trades?status=SETTLED`). Rule of thumb: if removing it changes *what resource* you're addressing, it's a path variable; if it changes *how you're querying/filtering* a resource, it's a request param.

---

## Q: Caching strategies in Spring Boot — what are the options?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Spring's `@Cacheable`/`@CacheEvict`/`@CachePut` abstraction works with a pluggable backend (`ConcurrentMapCacheManager` for local/in-memory, or Redis via `spring-boot-starter-data-redis` for a distributed cache shared across instances). The strategy question is really about **cache-aside vs write-through** — see the Redis file for the detailed comparison, since that's where this gets asked in more depth.

**How this maps to your project:** `reference-data-service` is the natural `@Cacheable` candidate — instrument/counterparty lookups are read-heavy and change rarely.

**Follow-ups to expect:**
- How do you handle cache invalidation when the underlying reference data *does* change?

---

## Q: Pagination & sorting with a JPA repository?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Extend `PagingAndSortingRepository` (or just `JpaRepository`, which already includes it), and accept a `Pageable`:
```java
Page<Trade> findByStatus(TradeStatus status, Pageable pageable);
```
Callers build `Pageable.ofSize(20).withPage(0)` or add `Sort.by("tradeDate").descending()`. Spring Data translates this into `LIMIT`/`OFFSET` (or keyset-based paging depending on the dialect) plus `ORDER BY` automatically.

**Follow-ups to expect:**
- Why can offset-based pagination get slow on very large tables, and what's the alternative? (Keyset/cursor-based pagination — seek from the last seen key instead of counting through `OFFSET`.)

---

## Q: Monolithic vs Microservices architecture — trade-offs?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

A monolith is simpler to develop, test, and deploy as a single unit, with easy in-process calls and shared transactions — but it couples deployment (one bug anywhere can require redeploying everything) and scaling (you scale the whole app even if only one piece is hot). Microservices let each service scale, deploy, and fail independently, and let teams own services end-to-end — at the cost of network calls replacing in-process calls, distributed transaction complexity (hence Saga), and significant operational overhead (service discovery, observability, deployment pipelines per service).

**How this maps to your project:** this whole project *is* that trade-off made concrete — e.g. `trade-capture-service` and `settlement-service` could have been one Spring Boot app, but we split them to practice exactly the distributed-transaction and resilience patterns a monolith wouldn't force us to confront.

---

## Q: What is the Spring Bean lifecycle? How does Spring Boot auto-configuration work internally?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — frequently asked for 3-6 YOE)
**Your answer:**

**Refined answer:**

**Bean lifecycle, in order:** instantiate (constructor) → populate properties (dependency injection) → `BeanNameAware`/`BeanFactoryAware` callbacks if implemented → `@PostConstruct` / `InitializingBean.afterPropertiesSet()` → bean is ready and in use → on shutdown, `@PreDestroy` / `DisposableBean.destroy()`. `BeanPostProcessor` implementations can hook in before and after initialization for any bean (this is how many Spring features, like AOP proxy creation, are actually implemented under the hood).

**Auto-configuration internals:** Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (or the older `spring.factories`) for `@AutoConfiguration` classes bundled in every starter JAR on the classpath. Each one is annotated with conditions — `@ConditionalOnClass` (only apply if a certain class is on the classpath, e.g. only configure a `DataSource` if a JDBC driver is present), `@ConditionalOnMissingBean` (only apply if the user hasn't already defined their own bean of that type — so your own `@Bean` always wins over the auto-configured default), `@ConditionalOnProperty`, etc. This is why adding a dependency (like `spring-boot-starter-data-redis`) "just works" with sensible defaults, but defining your own bean of the same type silently overrides it rather than conflicting.

**Follow-ups to expect:**
- Why does your own `@Bean` override an auto-configured one without an explicit `@Primary`? (Because the auto-configuration itself is conditioned on `@ConditionalOnMissingBean` — it only ever supplies a *default*, deliberately designed to get out of the way once you provide your own.)

---

## Q: How would you implement request correlation IDs across multiple microservices?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Generate a correlation ID (UUID) at the system's entry point — typically the API Gateway, or the first service to receive an external request if there's no gateway yet — and propagate it on every downstream call as an HTTP header (e.g. `X-Correlation-Id`) and on every Kafka message as a header/metadata field. Each service reads the incoming correlation ID (generating a new one only if genuinely absent, e.g. for an internally-triggered job), includes it in every log line for that request (via an MDC — Mapped Diagnostic Context — entry so it's automatically attached to every log statement without threading it through manually), and forwards it on any further calls it makes. The payoff: when something goes wrong, you can grep/query logs across *every* service involved in that one request by a single ID, instead of trying to correlate by timestamp guessing.

**Implementation detail:** this needs the correlation ID stored somewhere thread-local-ish for the duration of the request (Spring's `ThreadLocal`-backed MDC, or — important given the Virtual Threads entry in `multithreading-concurrency.md` — if the request hops across async boundaries/different threads, the correlation ID must be explicitly carried across that boundary, since `ThreadLocal` by definition doesn't follow the logical request onto a different thread automatically.

**How this maps to your project:** this is the same correlation/Saga ID already called out as a senior-practice requirement in the Saga pattern entry (`microservices-system-design.md`) — build it once, in `api-gateway`, and it serves both tracing and Saga-workflow tracking.

---

## Q: How would you validate requests involving multiple dependent fields (cross-field validation)?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Standard Bean Validation annotations (`@NotNull`, `@Positive`) validate one field in isolation — they can't express "settlementDate must be after tradeDate" or "if side is SELL, quantity must not exceed available position." For that, write a **class-level custom constraint**: a `@Constraint`-annotated annotation plus a `ConstraintValidator<Annotation, TargetClass>` implementation that receives the whole object and can compare fields against each other, applied at the class level (`@ValidTradeDates class TradeRequest { ... }`) rather than on a single field.

**How this maps to your project:** `TradeRequest` in `trade-capture-service` is exactly where this is needed — e.g. validating that `settlementDate` isn't before `tradeDate` — a good concrete exercise once the basic DTO/Builder work (Phase 0) is done.

---

## Q: How would you prevent sensitive information from appearing in application logs?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Several layers, because any single one is easy to miss a case for: (1) never log full request/response bodies by default for endpoints handling sensitive data (PII, account numbers, auth tokens) — log structured, deliberately-chosen fields instead; (2) override `toString()` on sensitive domain objects to mask fields (e.g. show only the last 4 digits of an account number) so an accidental `log.info("{}", trade)` doesn't leak everything; (3) a logging filter/interceptor that scans for known sensitive field names/patterns (card numbers, SSNs) and redacts them before the log line is written, as a safety net for cases 1 and 2 miss; (4) be careful with exception logging specifically — a stack trace from a failed validation can embed the exact input value that failed, including the sensitive one.

**How this maps to your project:** worth deciding this policy explicitly before `trade-capture-service` logs anything involving counterparty/account details — retrofitting log redaction after sensitive data has already been written to log files (which may be retained, shipped to a log aggregator, etc.) is much harder than designing it in from the start.

---

## Q: How would you test an application that depends on multiple external services?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch)
**Your answer:**

**Refined answer:**

Different layers of the test pyramid handle this differently — see `testing.md` for the full breakdown. The short version: **unit tests** mock the external dependency entirely (Mockito) so the test is fast and only exercises your own logic's handling of various responses (including failure responses, timeouts). **Integration tests** use **Testcontainers** (spin up a real, disposable instance of a real dependency — Postgres, Kafka, even a WireMock-stubbed HTTP server — in Docker, for the duration of the test) so you're testing against something that behaves like the real thing without needing the actual external service available or risking hitting it. **Contract tests** (e.g. Spring Cloud Contract) verify that your service and the service you depend on agree on the shape of their interaction, catching breaking changes before they reach a shared environment.

**How this maps to your project:** `trade-capture-service` calling `risk-service` is exactly this — mock `risk-service`'s client in unit tests, and once both services exist, consider a Testcontainers-based integration test that runs both together.

