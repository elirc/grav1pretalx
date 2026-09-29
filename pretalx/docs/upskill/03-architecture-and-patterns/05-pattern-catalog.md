# Design Pattern Catalog

This catalog outlines the custom design patterns used in **pretalx**, providing standard definitions, concrete file anchors, failure modes, and JS/TS direct equivalents.

---

## Pattern 1: Multi-Tenant Scoping (Database Scope Isolation)

*   **Problem it solves:** In multi-tenant SaaS applications, developers can accidentally write queries that leak data across tenants (e.g. showing Event A's submissions on Event B's admin panel).
*   **General shape:** A global middleware binds the tenant context (`request.event`) to thread-local or context-local storage. The database manager automatically intercept all SQL queries to inject a `WHERE tenant_id = current_tenant` filter.
*   **JS/TS name for this:** Prisma Client Extensions / Multitenancy Middleware.
*   **Real example:** [`src/pretalx/common/middleware/event.py#L80-L100`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py#L80-L100) (activates the `scope` context block).
*   **Second example:** [`src/pretalx/submission/models/submission.py#L209-L210`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L209-L210) (defines custom manager `ScopedManager(event="event")`).
*   **Why this works:** Rather than relying on developers to manually type `.filter(event=request.event)` on every single query, the scope is enforced globally. Any attempt to run a query without an active scope context raises a `ScopeError`, making data leaks virtually impossible.
*   **Failure modes:** Accessing objects from outside a request thread context (like in background tasks or management CLI tools) requires explicitly opening a scope block manually via `with scope(event=event):`. Forgetting to do so crashes the script.
*   **Use when:** Designing multi-tenant databases where leakage carries severe legal, privacy, or commercial consequences.
*   **Avoid when:** The system is single-tenant or models are globally shared across all domains (like global user accounts).
*   **Drill:** Open `src/pretalx/submission/models/tag.py` and identify how the custom `ScopedManager` is registered on the model class.

---

## Pattern 2: Declarative Rule Engine (Predicate-based Permissions)

*   **Problem it solves:** Standard RBAC (Role-Based Access Control) systems are highly rigid. Authorizing actions based on object-level states (e.g. "a speaker can edit a proposal only if the CFP is still open") results in giant, messy nested `if/else` views.
*   **General shape:** Permissions are modeled as pure functions (predicates) that return boolean values. These predicates are combined using logical operators (`&`, `|`, `~`) and registered in a centralized rules registry.
*   **JS/TS name for this:** CASL / Predicate-based authorization libraries.
*   **Real example:** [`src/pretalx/submission/rules.py#L65-L70`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py#L65-L70) (defines `is_speaker` predicate).
*   **Second example:** [`src/pretalx/submission/models/submission.py#L219-L258`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L219-L258) (binds permissions dictionary in the model Meta class).
*   **Why this works:** Decouples authorization rules from the database representation and the HTTP view. Permissions can be unit-tested in isolation without mocking full HTTP request cycles.
*   **Failure modes:** Performance bottlenecks. If predicates contain un-optimized database lookups (like checking user profiles in a loop), querying permissions on lists can cause severe N+1 database bottlenecks.
*   **Use when:** Object-level permissions depend on complex, dynamic states or relations.
*   **Avoid when:** Access is extremely simple (e.g. only administrators can access).
*   **Drill:** Write a new rule predicate `is_proposal_short` that evaluates as `True` only if `submission.get_duration() < 30`.

---

## Pattern 3: State Machine transitions via isolated Domain Actions

*   **Problem it solves:** When a model's state changes, a sequence of secondary side-effects must occur (e.g. when accepting a talk: change state to accepted, adjust room slots, render notification, send emails). Mixing this inside views couples the controller to side-effects.
*   **General shape:** A standalone domain-service module encapsulates all state mutations. View controllers are strictly banned from directly editing model state variables. Instead, they call a centralized, single state-transition method.
*   **JS/TS name for this:** DDD Service layer / Domain-driven state controllers.
*   **Real example:** [`src/pretalx/submission/domain/submission.py#L193-L262`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L193-L262) (the `set_submission_state` service).
*   **Second example:** [`src/pretalx/submission/models/submission.py#L410-L417`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L410-L417) (models delegate `.accept()` directly to the service).
*   **Why this works:** Ensures that side-effects never run partially. Wrapping these operations inside database transactions ensures complete integrity.
*   **Failure modes:** Swallowing errors. If a sub-task fails silently (like Celery failing to queue the email), the state is updated but the side-effect fails, creating a state mismatch.
*   **Use when:** Model state mutations have complex, chained side-effects.
*   **Avoid when:** Standard CRUD updates have zero side-effects.

---

## Pattern 4: Transactional Outbox (Queued Mail drafts)

*   **Problem it solves:** Sending an email synchronously during an HTTP request blocks the server, hurting UX. Triggering the email asynchronously runs the risk of a failure: if the email fails, the speaker is accepted but gets no alert; if the transaction fails, the email is sent but the data rolls back!
*   **General shape:** Instead of sending emails immediately, the service renders a template and saves a "Draft" record in a database table within the same ACID transaction. A background worker periodically polls the outbox table to dispatch outstanding emails.
*   **JS/TS name for this:** Transactional Outbox Pattern / Message Queue Outbox.
*   **Real example:** [`src/pretalx/submission/domain/submission.py#L354-L374`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L354-L374) (uses `save_draft` inside transition methods).
*   **Second example:** [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py) (declares the `QueuedMail` database representation).
*   **Why this works:** Guarantees **at-least-once delivery**. If the email service is down, the draft stays safely in the database outbox and is retried. If the parent database transaction rolls back, no email is sent, preventing phantom notifications.
*   **Failure modes:** Delay. Speakers do not receive emails instantly. They remain in the "Outbox" draft queue until Celery processes them or the organizer manually releases them.
*   **Use when:** Transactional notifications are critical and must align perfectly with database state changes.
*   **Avoid when:** Notifications are purely fire-and-forget or transient (like password reset tokens).
