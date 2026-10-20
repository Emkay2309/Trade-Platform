# CI/CD & Deployment Strategy

Added proactively — complements the hands-on git/GitHub/CI-CD work you're doing yourself in `PROJECT_PLAN.md` Phase 10 with the conceptual questions an interviewer asks around it.

## Q: What does a typical CI/CD pipeline for a microservice look like?
**Your answer:**

**Refined answer:**

**CI (on every push/PR):** checkout → build → run unit tests → run integration tests (Testcontainers-backed, see `testing.md`) → static analysis/lint → build a container image → push the image to a registry, tagged with the commit SHA (and often a semantic version on release branches). The pipeline should fail fast — cheap checks (lint, unit tests) before expensive ones (integration tests, image build).

**CD (deployment):** the built image gets deployed to an environment — often automatically to a dev/staging environment on merge to main, and to production via a manual approval gate or a separate release pipeline. For Kubernetes targets specifically, this usually means updating a Deployment's image tag (`kubectl set image` or, more robustly, a GitOps tool like ArgoCD reconciling a Git-stored desired state) rather than imperatively SSHing anywhere.

**How this maps to your project:** this is exactly Phase 10 — a GitHub Actions workflow that builds/tests each service on push, builds its Docker image, and (once Phase 9's EKS is up) deploys it.

---

## Q: Blue-green vs canary deployment — what's the difference, and when would you choose each?
**Your answer:**

**Refined answer:**

**Blue-green:** run two complete environments ("blue" = current live, "green" = new version), fully deploy and verify the new version in green while blue still serves all traffic, then switch all traffic to green atomically (a load balancer/router flip). Rollback is just flipping back to blue. Downside: needs double the infrastructure running simultaneously during the switch, and it's all-or-nothing — a bug only visible under 1% of real traffic patterns might not surface until 100% of traffic has already moved.

**Canary:** route a small percentage of real traffic (e.g. 5%) to the new version while the rest stays on the old one, monitor error rates/latency on that slice, and gradually increase the percentage if it looks healthy (or roll back immediately if not). Catches problems that only show up under real traffic diversity, with much smaller blast radius than blue-green's all-or-nothing switch — at the cost of more complex routing/monitoring to pull off correctly, and tolerating two versions running simultaneously for longer.

**Rule of thumb:** canary when you want gradual risk exposure and have the traffic-splitting/monitoring infrastructure to support it; blue-green when you want a clean, fast, fully-reversible switch and can afford running two full environments briefly.

**How this maps to your project:** worth explicitly picking one for the Phase 9 AWS/K8s deployment and being able to justify why — a financial trade-processing system is exactly the kind of system where "small blast radius if something's wrong" (canary) is usually the stronger default over a full all-at-once switch.

---

## Q: What's a rollback strategy, and why does it matter even with good testing?
**Your answer:**

**Refined answer:**

No amount of pre-deployment testing catches everything that only manifests under real production load/data — the question isn't "how do we never have a bad deploy," it's "how fast and safely can we undo one when it happens." A good rollback strategy means: deployments are versioned/tagged so "go back to the previous version" is a specific, known action, not a guess; database migrations are backward-compatible for at least one version (a rollback shouldn't require an equally risky reverse migration under pressure); and the deployment tool itself supports a fast rollback path (Kubernetes `kubectl rollout undo`, or redeploying the previous image tag) rather than requiring a fresh manual deploy of old code. The real metric to care about isn't "will we ever deploy a bug" — it's **MTTR** (mean time to recovery) once one ships.
