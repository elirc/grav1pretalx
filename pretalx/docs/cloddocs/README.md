# cloddocs — Supplementary Architecture & System Design Deep-Dives

**Author:** Claude (Supplementary Module)  
**Purpose:** This folder fills the gaps left by the primary upskill curriculum, specifically in the areas of distributed systems theory, production observability, concurrency safety, and API design trade-offs — all grounded in the pretalx codebase.

The existing training material builds an excellent foundation in Django patterns, ORM mechanics, and the four signature pretalx design patterns. What it does not teach is how to reason about the *system as a whole* under realistic conditions: when nodes fail, when two users edit simultaneously, when traffic spikes 10x in a minute, and when you need to know whether things are actually working.

These modules are not replacements for anything in `docs/upskill/`. They are extensions. Read them after you have worked through the primary curriculum.

---

## 📂 Modules in This Folder

### [`01-cap-theorem-and-consistency.md`](./01-cap-theorem-and-consistency.md)
Why pretalx makes the consistency choices it does — explained through CAP theorem, ACID vs BASE trade-offs, and why the transactional outbox pattern is the correct engineering choice for at-least-once notification delivery.

### [`02-observability-in-production.md`](./02-observability-in-production.md)
How to actually know pretalx is healthy. Covers the three pillars of observability (logs, metrics, traces), what to instrument, what alerts to set, and how to diagnose a production incident without `print()` statements.

### [`03-concurrency-and-optimistic-locking.md`](./03-concurrency-and-optimistic-locking.md)
Two organizers drag the same talk slot simultaneously. Who wins? Covers optimistic locking, pessimistic locking, idempotency tokens, and why the pretalx schedule editor's API design is an excellent concurrency case study.

### [`04-api-design-tradeoffs.md`](./04-api-design-tradeoffs.md)
pretalx uses Django REST Framework. Why not GraphQL? Why not gRPC? This module teaches you to reason about API paradigms — REST, GraphQL, gRPC, and webhooks — and explain trade-offs at a senior engineering level.

### [`05-database-index-design.md`](./05-database-index-design.md)
The most under-taught system design topic. Covers B-tree indexes, composite index column ordering, covering indexes, partial indexes, and how to read an `EXPLAIN ANALYZE` query plan — using real pretalx query patterns as examples.

### [`06-failure-isolation-and-resilience.md`](./06-failure-isolation-and-resilience.md)
What happens when Redis is down? When Celery workers crash? When the mail server times out? Covers circuit breakers, bulkheads, retry strategies with exponential backoff, and dead-letter queues — with pretalx's production paths as the concrete case study.

---

## Who Should Read These

| Role | Recommended Modules |
|:--- |:--- |
| Mid-Level Engineer targeting a Senior role | All six, in order |
| Senior Engineer preparing for system design interviews | 01, 03, 04, 05 |
| DevOps / SRE transitioning into backend | 02, 05, 06 |
| JS developer learning backend architecture | 01, 03, 04 |

---

> [!NOTE]
> These documents deliberately avoid re-explaining content already in `docs/upskill/`. Cross-references to the primary curriculum are provided where relevant. If you encounter a term that seems unfamiliar, check the Rosetta Stone at `docs/upskill/09-reference/js-python-rosetta-stone.md` first.
