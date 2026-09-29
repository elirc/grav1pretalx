# Software Engineering Interview Prep

This module serves as your dedicated preparation lab for mid-level software engineering interviews, using the production architecture of **pretalx** as your case study.

---

## 🐍 Python & Django Mid-Level Interview Core

### Topic 1: Lazy Query Evaluation & The N+1 Problem
*   **Concept:** Django QuerySets are lazy. Creating a filter like `Submission.objects.filter(state='accepted')` does not hit the database. The query runs only when you iterate, slice, or count the results.
*   **The Gotcha:** If you load submissions and loop over speakers without eager loading, you run one query for submissions and N queries for speakers.
*   **How pretalx Solves It:** Uses custom Managers with `prefetch_related` to fetch the relationship in a single extra batch query, reducing database roundtrips from N+1 down to exactly 2.
*   **File Anchor:** [`src/pretalx/submission/models/submission.py#L46-L53`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L46-L53) (the custom prefetching query manager).

#### Mock Interview Question
> *"Explain how an ORM's lazy evaluation works. What are the performance hazards, and how do you resolve them in a production system?"*

*   **Mid-Level Answer:** "Lazy evaluation delays query execution until the data is actually read. This saves memory and prevents useless database hits. However, it can cause N+1 query loops when accessing related tables. In Django, we solve this using `select_related` for ForeignKeys (which performs an SQL JOIN) or `prefetch_related` for ManyToMany fields (which performs a single extra IN query). In pretalx, this optimization is encapsulated inside custom model managers to keep views clean."

---

### Topic 2: Enforcing Application-Level Security Boundaries
*   **Concept:** SaaS applications must isolate data per client (tenant). Forgetting to filter queries by client ID is a major security flaw.
*   **How pretalx Solves It:** Enforces data isolation at the database manager level using `django-scopes`. Queries that attempt to load records without an active event context will raise a `ScopeError`.
*   **File Anchor:** [`src/pretalx/common/middleware/event.py#L80-L100`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py#L80-L100).

#### Mock Interview Question
> *"How do you design and enforce multi-tenant database isolation in a web application?"*

*   **Mid-Level Answer:** "We enforce multi-tenancy by linking every core table to a tenant ID (like `event_id`). To prevent developers from forgetting to append filters, we use database-level scoping middleware. In pretalx, the scoping middleware extracts the tenant slug, binds it to request context, and enforces that all subsequent database queries remain strictly inside that event scope. This protects against cross-tenant data leaks."

---

## 🪟 JavaScript & TypeScript Mid-Level Interview Core

### Topic 3: The Event Loop & Non-Blocking Concurrency
*   **Concept:** JavaScript is single-threaded. It uses an event loop to handle non-blocking asynchronous operations (network, file I/O).
*   **The Python Comparison:** While Python is synchronous by default and uses distinct worker processes (Celery/Gunicorn) to run heavy tasks, JS handles parallel I/O natively via its event loop.
*   **Where it applies:** The Svelte visual timeline scheduler inside pretalx makes rapid, parallel HTTP requests to the REST API endpoints.

#### Mock Interview Question
> *"Explain the difference between Python's process-based concurrency and JavaScript's event loop model."*

*   **Mid-Level Answer:** "JavaScript handles I/O asynchronously on a single thread. The V8 engine delegates I/O operations to the system (libuv), queuing a callback on the event loop once complete. This makes JS highly efficient for massive parallel connections. Python, on the other hand, runs synchronously by default. In pretalx, we handle high-concurrency and async operations by deploying independent worker processes (Celery) connected to redis queues. This keeps our main HTTP threads responsive."

---

### Topic 4: Discriminated Unions & Type Narrowing
*   **Concept:** A type system should prevent illegal states at compile time.
*   **Where it applies:** Mapping submission state transitions (Draft, Submitted, Accepted, Rejected).

#### Mock Interview Question
> *"What are discriminated unions in TypeScript, and how do they improve code safety?"*

*   **Mid-Level Answer:** "A discriminated union combines literal fields (like `type: 'accept'`) with distinct payload shapes in a union type. When TypeScript checks the literal field, it automatically narrows the type in that block scope. This prevents accessing fields that do not exist in the current state, catching bugs at compile time. In Python, we can model this using `typing.Literal` or pattern matching `match/case` blocks."
