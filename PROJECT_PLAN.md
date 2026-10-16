# Project Plan — IB Trade Platform

## System overview

A trade lifecycle platform modeled on real investment-bank workflows:
a trade is captured, checked against risk/compliance rules, settled through
a multi-step process, and audited — with live market data streaming in via
Kafka and reference data cached in Redis.

## Services

| Service | Responsibility | Key patterns/tech |
|---|---|---|
| **api-gateway** | Single entry point, routing, auth enforcement | Spring Cloud Gateway, JWT validation, Rate limiting |
| **auth-service** | User auth, token issuance | Spring Security, OAuth2/JWT, BCrypt |
| **trade-capture-service** | Receives/validates new trades | Builder, Factory, Bean Validation |
| **risk-service** | Runs risk/compliance rules against a trade | Strategy, Chain of Responsibility, Rules engine |
| **settlement-service** | Drives trade through lifecycle states | State, Saga (orchestration), Outbox pattern |
| **market-data-service** | Streams simulated live prices | Kafka Producer, Observer |
| **reference-data-service** | Counterparties/instruments, cached | Redis cache-aside, Repository pattern |
| **notification-audit-service** | Consumes events, writes audit trail | Kafka Consumer, Event Sourcing |

Cross-cutting: Circuit Breaker + Retry (Resilience4j) on all inter-service
calls, Service Discovery (Eureka or K8s-native), centralized config
(Spring Cloud Config), Sidecar/Ambassador via K8s where relevant.

## Build phases (rough order, adjust as we go)

1. **Phase 0 — Skeleton**: multi-module Maven/Gradle setup, one service
   (trade-capture) with a basic REST endpoint, local Postgres, Dockerfile.
2. **Phase 1 — Core domain + patterns**: trade-capture fully modeled
   (Builder/Factory), reference-data-service + Redis caching.
3. **Phase 2 — Risk engine**: risk-service with Strategy + Chain of
   Responsibility rule pipeline.
4. **Phase 3 — Messaging**: Kafka wired in — market-data-service producing,
   notification-audit-service consuming. Outbox pattern introduced.
5. **Phase 4 — Settlement + Saga**: settlement-service with explicit state
   machine, saga orchestration across services.
6. **Phase 5 — Security**: auth-service, JWT across all services, API gateway.
7. **Phase 6 — Resilience**: Circuit breaker, retry, bulkhead on inter-service calls.
8. **Phase 7 — Containerization**: Dockerfiles per service, docker-compose
   for local multi-service run.
9. **Phase 8 — Kubernetes**: manifests/Helm chart, deploy to local cluster (kind/minikube).
10. **Phase 9 — AWS**: deploy to EKS or ECS, RDS, MSK/SQS, S3, IAM roles.
11. **Phase 10 — CI/CD**: GitHub Actions pipeline (build, test, Docker push, deploy).
12. **Phase 11 — React dashboard** (after backend is solid): basic UI to view trades/risk/settlement status.

Each phase = several days of the 2.5-3 hr project block. We'll adjust order
based on how fast concepts land.
