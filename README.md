# IB Trade Platform — Interview Prep Project

Personal learning project built to prepare for a Java 6+ YOE / Full-Stack
Developer role. Simulates an investment-bank-style trade lifecycle system
(trade capture → risk check → settlement → audit) using the microservices
patterns and tech stack commonly asked about in interviews.

**Honesty note:** this is a personal/portfolio project, not a claim of
employment at any bank. Framed on resume/interviews as "built a trade
settlement & risk microservices platform modeling IB-style trade lifecycle
workflows" — not as professional experience at a specific firm.

## Repo layout

- `services/` — the actual microservices (Java/Spring Boot). You write this code.
- `notes/` — topic-wise structured notes, written as we cover each topic.
- `question-bank/` — Core Java/OOP/Multithreading/Streams rapid-fire questions
  used for the daily 20-min warm-up, with answers.
- `interview-experiences/` — your real interview question dumps, organized
  by topic, mined for practice.
- `progress-log/` — daily log of what was covered, what was built, what's next.
- `design-patterns-checklist.md` — master list of patterns to cover, mapped
  to where each one lives in the project.
- `PROJECT_PLAN.md` — architecture and build roadmap.

## Daily workflow

1. 20-min warm-up: 2-5 Core Java/OOP/Multithreading/Streams questions
2. 2.5-3 hrs: build the next increment of the project (one service/feature/pattern at a time)
3. Log what was done in `progress-log/`
