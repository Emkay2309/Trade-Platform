# AWS & Cloud Infrastructure

## Q: Difference between EC2, ECS, and Lambda — when would you use each?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round)
**Your answer:**

**Refined answer:**

- **EC2** — raw virtual machines. You manage the OS, runtime, scaling, patching — maximum control, maximum operational burden. Use when you need full control over the environment or are running something that doesn't fit a container/function model cleanly.
- **ECS (Elastic Container Service)** — AWS's container orchestrator. You package your app as a container; AWS handles scheduling it onto infrastructure (EC2-backed, or **Fargate** for fully serverless containers with no instance management at all). This is the natural fit for a set of long-running microservices like this project's.
- **Lambda** — functions-as-a-service. You deploy just a function; AWS handles everything (scaling to zero, per-invocation billing, no servers to manage at all), but with constraints — execution time limits, cold-start latency, and a programming model suited to short, event-triggered work rather than a long-running stateful service.

**Rule of thumb:** long-running services with steady traffic and their own lifecycle → ECS/EKS; sporadic, event-triggered, short-lived work (a file-upload handler, a scheduled cleanup task) → Lambda; needing OS-level control or something that doesn't containerize well → EC2.

**How this maps to your project:** the trade platform's services are long-running Spring Boot apps with persistent connections (DB, Kafka) — a poor fit for Lambda's execution model. `PROJECT_PLAN.md` Phase 9 targets EKS (Kubernetes on AWS) rather than plain ECS, specifically so the Kubernetes concepts from Phase 8 transfer directly to a cloud target instead of being thrown away.

---

## Q: How do you monitor applications in AWS?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — PayPal round)
**Your answer:**

**Refined answer:**

**CloudWatch** is the baseline — metrics (CPU, memory, custom app metrics), Logs (centralized log aggregation from containers/instances), and Alarms (trigger on a metric crossing a threshold, e.g. error rate or latency). For a microservices system specifically, metrics alone aren't enough to debug a cross-service issue — you also want **distributed tracing** (AWS X-Ray, or an OpenTelemetry-based stack) to follow a single request's path across multiple services and see where time was actually spent, which is exactly the kind of thing that diagnoses a "low CPU, high latency" symptom (see `production-war-stories.md`) by showing *where* a request was blocked, not just *that* it was slow.

**How this maps to your project:** this is the natural companion to Phase 9 (AWS deployment) — CloudWatch for metrics/alarms on each service, and a correlation ID (already needed for the Saga/idempotency work) doubling as the trace identifier across services.
