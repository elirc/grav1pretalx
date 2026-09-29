# Design Pattern Catalog

This catalog outlines the custom design patterns used in **pretalx**, providing standard definitions, concrete file anchors, failure modes, and JS/TS direct equivalents.

---

## Pattern: Multi-Tenant Scoping (Scoped managers)

**Problem it solves:** Prevents cross-tenant data leaks. In a multi-tenant application, it is easy to write queries that accidentally load other conferences' database rows.

**General shape:** A middleware binds the event context to thread-local or context storage. A database manager automatically wraps every database query with a scoping filter matching `request.event`.

**Real example:** [`src/pretalx/common/middleware/event.py#L80-L100`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py#L80-L100) (activates the event scope context manager).

**Second example:** [`src/pretalx/submission/models/submission.py#L209-L210`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L209-L210) (registers the scoping manager on models).

**Why this implementation works:** Instead of relying on developer memory to write filter checks everywhere, the database layer throws an explicit error if you attempt to run queries outside an active scope context, ensuring complete tenant boundary safety.

**Failure modes:** Forgetting to activate scopes manually in background tasks or CLI scripts crashes execution.

**Use it when:**
*   You are building SaaS architectures with shared-database multi-tenant patterns.

**Avoid it when:**
*   You are building single-tenant systems or have models that span globally (like user accounts).

**Drill:**
Open `src/pretalx/submission/models/tag.py` and inspect how `ScopedManager` is declared.

---

## Pattern: Declarative Predicates (Rules authorization)

**Problem it solves:** Decouples complex business permission logic from views and database structures.

**General shape:** Permissions are modeled as pure functions (predicates) returning booleans, combined with standard logic operators (`&`, `|`, `~`).

**Real example:** [`src/pretalx/submission/rules.py#L65-L70`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py#L65-L70) (defines `is_speaker` predicate).

**Second example:** [`src/pretalx/submission/models/submission.py#L219-L258`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L219-L258) (registers permission dict in model Meta).

**Why this implementation works:** Simplifies permissions mapping by linking authorization directly to ORM classes. It allows rule testing in absolute isolation.

**Failure modes:** Calling database checks within predicates inside loops can cause severe N+1 database roundtrips.

**Use it when:**
*   Permissions depend on complex, dynamic object-level attributes and transitions.

**Avoid it when:**
*   Access control is extremely basic (e.g. simple admin role checks).

**Drill:**
Create a predicate `is_short_talk` returning `True` only if `submission.get_duration() <= 20`.

---

## Pattern: Transactional Outbox (Queued Mail outbox)

**Problem it solves:** Prevents transactional inconsistencies. If the database save fails but the email sends, the user receives an alert for a rolled-back action. If the database save succeeds but email fails, the user is never notified.

**General shape:** Renders and persists emails to an outbox table (`QueuedMail`) inside the parent database transaction. Background workers poll and dispatch them asynchronously.

**Real example:** [`src/pretalx/submission/domain/submission.py#L354-L374`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L354-L374) (uses `save_draft` inside state transition methods).

**Second example:** [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py) (declares outbox fields).

**Why this implementation works:** Guarantees at-least-once notification delivery without holding web request threads open.

**Failure modes:** Alerts are not instantaneous and can delay depending on Celery queue processing times.

**Use it when:**
*   Notification delivery must remain strictly consistent with core model state changes.

**Avoid it when:**
*   Mails are purely transient or fire-and-forget (like password reset tokens).

**Drill:**
Open `mail/models.py` and inspect `QueuedMail` fields.
