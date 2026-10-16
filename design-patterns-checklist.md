# Design Patterns Checklist

Tracked against where each pattern will actually appear in the project
(see `PROJECT_PLAN.md`). Check off once implemented AND you can explain
the "why this pattern over alternatives" in interview terms.

## GoF — Creational

- [ ] Singleton — config holders / Spring beans (explain why Spring beans
      are singleton-scoped by default, and the classic non-DI singleton pitfalls)
- [ ] Factory Method — `trade-capture-service` (creating Trade subtypes: Equity/Bond/FX)
- [ ] Abstract Factory — extending the above to create matched families (e.g. trade + its confirmation doc)
- [ ] Builder — constructing complex `Trade` objects with optional fields
- [ ] Prototype — cloning a trade template for amendments

## GoF — Structural

- [ ] Adapter — wrapping a legacy/external market-data format into our internal model
- [ ] Decorator — adding cross-cutting behavior (e.g. logging/metrics) around risk checks
- [ ] Facade — simplifying a multi-step settlement call into one entry point
- [ ] Proxy — lazy-loading or access-control wrapper around a service client
- [ ] Composite — representing a basket/portfolio of trades uniformly with single trades
- [ ] Bridge — decoupling settlement "process" from settlement "implementation" (e.g. different clearing houses)
- [ ] Flyweight — sharing immutable instrument/reference data objects

## GoF — Behavioral

- [ ] Strategy — interchangeable risk-check algorithms in `risk-service`
- [ ] Chain of Responsibility — pipeline of risk/compliance checks
- [ ] Observer — notifying subscribers when market data changes
- [ ] State — trade lifecycle states in `settlement-service`
- [ ] Template Method — common skeleton for all risk checks with hook methods
- [ ] Command — encapsulating a trade action (submit/cancel/amend) as an object
- [ ] Iterator — (mostly via Java Collections — be able to explain, not necessarily hand-roll)
- [ ] Mediator — coordinating multiple services/components without direct coupling
- [ ] Visitor — operations over a trade hierarchy without modifying the classes
- [ ] Memento — capturing/restoring trade state for undo/amend history

## Concurrency patterns (Java)

- [ ] Producer-Consumer — Kafka producer/consumer is the real-world version
- [ ] Thread Pool / Executor — async processing in notification-audit-service
- [ ] Future/CompletableFuture — async inter-service calls
- [ ] Read-Write Lock — concurrent cache access in reference-data-service
- [ ] Immutability — trade/market-data value objects

## Microservices / Enterprise patterns

- [ ] API Gateway — `api-gateway` service
- [ ] Service Discovery — Eureka or K8s-native DNS
- [ ] Circuit Breaker — Resilience4j on inter-service calls
- [ ] Retry / Bulkhead — Resilience4j
- [ ] Saga (orchestration) — `settlement-service` driving multi-service trade settlement
- [ ] CQRS — separating trade write model from reporting/read model
- [ ] Event Sourcing — audit trail built from an event log
- [ ] Outbox Pattern — reliable event publishing from settlement-service to Kafka
- [ ] Database per Service — each service owns its schema
- [ ] Strangler Fig — discuss conceptually (how you'd migrate a monolith incrementally)
- [ ] Backend for Frontend (BFF) — once React dashboard is added
- [ ] Sidecar / Ambassador — via K8s (e.g. Envoy/Istio sidecar for mTLS, discuss conceptually)

## How we use this file

- Updated as we build — don't check a box until you've both coded it
  and can explain trade-offs out loud.
- Revisit weekly: pick 1-2 unchecked "discuss conceptually" patterns
  for the daily warm-up if the project hasn't needed them yet.
