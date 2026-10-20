# AI/LLM Integration (Spring AI, LangChain4j, Agentic AI)

**Priority note:** lower priority for a core Java backend/microservices role unless the specific job description mentions AI/LLM integration. Keeping brief definitions here so you're not blank if asked, without spending deep practice time unless it comes up.

## Q: Have you used agentic AI components (planner/executor)? How would you orchestrate planner, executor, and Kafka in an AI workflow? How do you mitigate LLM hallucination (RAG)?
**Asked/relevant on:** 2026-10-20 (LinkedIn batch — AI-interviewer round)
**Your answer:**

**Refined answer (definitions only — this is not project-backed experience; answer honestly as conceptual knowledge if asked, don't claim hands-on use you haven't had):**

An **agentic** workflow splits an LLM-driven task into a **planner** (decides the sequence of steps/tools needed to accomplish a goal) and an **executor** (actually invokes each tool/API and feeds results back). **Kafka** fits in as the backbone for async, multi-step agent workflows — a planner's decided steps can be published as events, executed asynchronously by worker services, with results flowing back via further events, rather than one long synchronous call chain.

**RAG (Retrieval-Augmented Generation)** mitigates hallucination by retrieving relevant real documents/data (via a vector database/embedding search) and feeding them into the LLM's prompt as grounding context, so the model answers from retrieved facts rather than purely from its trained parameters — it doesn't eliminate hallucination, but substantially reduces it for domain-specific questions by giving the model something true to work from.

**Spring AI vs LangChain4j** — both are Java abstraction layers over LLM providers (prompt templates, tool-calling, RAG pipelines, vector store integrations); Spring AI integrates natively with the Spring ecosystem (auto-configuration, Spring Boot starters), while LangChain4j is a more general, framework-agnostic port of the LangChain concepts. If asked "which would you choose," the honest answer is "Spring AI, for a Spring Boot-based system, for the same reason you'd pick any framework that integrates natively with your existing stack rather than bolting on a separate one."

**How this maps to your project:** not currently in `PROJECT_PLAN.md`. If a specific interview loop weights this heavily, flag it and we can scope a small add-on (e.g. an LLM-based trade-note summarizer) rather than answering purely from theory.
