# Testing (JUnit, Mockito, Testcontainers)

Added proactively — not from the LinkedIn dump, but a near-certain topic at 6+ YOE, and referenced by the "testing an app with multiple external dependencies" entry in `spring-boot.md`.

## Q: What's the test pyramid, and how does it apply to a microservices project?
**Your answer:**

**Refined answer:**

- **Unit tests** (the base, most numerous, fastest) — test one class in isolation, all dependencies mocked (Mockito). No Spring context, no database, no network — milliseconds per test.
- **Integration tests** (fewer, slower) — test a real slice of the application together (e.g. a controller through to a real-but-disposable database via Testcontainers), verifying the pieces actually wire together correctly, not just that each piece's logic is right in isolation.
- **End-to-end / contract tests** (fewest, slowest) — verify a full user-facing flow, or that two services agree on their API contract (Spring Cloud Contract), across real or realistic service boundaries.

**The point of the pyramid shape:** push as much verification as possible down to the fast, cheap, isolated unit-test layer; use the slower layers only for what they uniquely catch (real wiring, real SQL, real serialization) — not as the default way to test business logic, since a bug reported by a 2-minute integration test is much more expensive to triage in CI than the same bug caught by a 50ms unit test.

**How this maps to your project:** every service's risk/settlement business logic belongs at the unit layer (mock the repository, mock the Kafka producer); the `JOIN FETCH`/N+1 fix and the idempotency unique-constraint behavior specifically need an integration test against a real database, because that's exactly the kind of bug a mocked repository can't reveal.

---

## Q: Mockito — mock vs spy vs stub, and what's the difference between @Mock, @MockBean, and @InjectMocks?
**Your answer:**

**Refined answer:**

A **mock** is a fully fake implementation of a dependency — every method returns a default/configured value unless you stub it; it records every interaction so you can verify calls happened (`verify(mock).save(any())`). A **spy** wraps a *real* object — calls go to the real implementation unless you explicitly stub a specific method, useful when you want most of the real behavior but need to override or verify one specific call. A **stub** is the narrower concept of "a canned answer to a specific call" (`when(mock.findById(1)).thenReturn(trade)`) — mocks and spies are both configured using stubbing.

- **`@Mock`** — creates a plain Mockito mock (used in pure unit tests, no Spring context).
- **`@MockBean`** — a Spring Boot Test annotation that replaces a real bean in the Spring application context with a Mockito mock, for use in a test that *does* load Spring context (e.g. a `@WebMvcTest` or `@SpringBootTest` for a controller) but wants one specific dependency mocked rather than real.
- **`@InjectMocks`** — tells Mockito to create the class under test and inject the `@Mock`-annotated fields into it automatically (via constructor/setter/field injection), so you don't have to wire them manually in every test's setup.

**Follow-ups to expect:**
- Why avoid `@SpringBootTest` (full context) for most tests? (It's slow — loads the entire application context; `@WebMvcTest`/`@DataJpaTest`/plain Mockito-based unit tests load only the slice actually under test, keeping the suite fast.)

---

## Q: What is Testcontainers, and why use it instead of an in-memory database (like H2) for integration tests?
**Your answer:**

**Refined answer:**

Testcontainers spins up real dependencies (Postgres, Kafka, Redis — anything with a Docker image) in disposable Docker containers, scoped to the test run, and tears them down afterward. The key advantage over an in-memory substitute like H2: **you're testing against the actual database engine you run in production**, including its specific SQL dialect quirks, constraint behavior, and index behavior — H2's "Postgres compatibility mode" is an approximation, and a subtle difference (a constraint violation that behaves differently, a query that H2 happens to handle but real Postgres doesn't) can pass in tests and fail in production, which defeats the point of having the test at all.

**How this maps to your project:** the idempotency unique-constraint test (does a duplicate request actually get rejected at the DB level under real concurrency?) and the N+1/`JOIN FETCH` test both need to run against real Postgres via Testcontainers — an H2-backed test could give a false pass on either.

**Follow-ups to expect:**
- What's the trade-off versus H2? (Slower — starting a real container per test class has real overhead — and requires Docker available in the CI environment; worth it specifically for tests where DB-engine-specific behavior is exactly what's being verified.)
