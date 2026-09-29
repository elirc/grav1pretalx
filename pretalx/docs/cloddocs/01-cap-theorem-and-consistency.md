# CAP Theorem & Consistency Models

**Supplementary Module — cloddocs 01**  
**Prerequisite Reading:** `docs/upskill/03-architecture-and-patterns/05-pattern-catalog.md` (Pattern 4: Transactional Outbox)

---

## Why This Matters

The existing pattern catalog correctly describes *what* the transactional outbox pattern does. It does not explain *why* it is the correct architectural choice rather than simply sending emails synchronously. The answer requires understanding the fundamental constraints on distributed systems, which every senior engineer must be able to articulate clearly.

---

## 1. The CAP Theorem — A Real Definition

The CAP theorem, proven by Eric Brewer in 2000, states that any distributed data store can satisfy at most **two** of the following three guarantees simultaneously:

| Property | Definition |
|:--- |:--- |
| **Consistency (C)** | Every read receives the most recent write, or an error. All nodes see the same data at the same moment. |
| **Availability (A)** | Every request receives a response (not necessarily with the most recent data). The system stays operational even if some nodes fail. |
| **Partition Tolerance (P)** | The system continues to operate even when network messages between nodes are lost or delayed. |

The critical insight that is almost always omitted in tutorials: **network partitions are not optional in real deployments**. Networks fail. Therefore the real choice is not "pick two of three" — it is always:

> **In the presence of a network partition, do you prioritize Consistency or Availability?**

This means the practical choice is between **CP** systems (halt operations if consistency cannot be guaranteed) and **AP** systems (continue operating, accept temporary inconsistency, reconcile later).

---

## 2. How pretalx Fits Into This Framework

pretalx runs on a single PostgreSQL database. At first glance this sidesteps CAP by not being a distributed database. But the moment you introduce Celery workers and Redis, you have a distributed system. Consider:

```
┌─────────────────────────────────────────────────────┐
│                  The pretalx System                  │
│                                                      │
│  Django Web Process ────────────────┐                │
│       │                             │                │
│       │ writes state                │ enqueues task  │
│       ▼                             ▼                │
│  PostgreSQL ────────────────► Redis/Celery           │
│  (QueuedMail rows)           (Task broker)           │
│                                     │                │
│                                     │ processes      │
│                                     ▼                │
│                               Celery Worker          │
│                               (sends email)          │
└─────────────────────────────────────────────────────┘
```

There are now **two distinct data stores** that must agree: PostgreSQL (ground truth) and Redis (task queue). What happens when they disagree?

### The Naive Approach (Broken)

```python
# ❌ DO NOT DO THIS — violates consistency guarantees
def accept_submission(submission):
    submission.state = "accepted"
    submission.save()                    # Step 1: Write to PostgreSQL
    send_email_immediately(submission)   # Step 2: Call mail server directly
```

**Failure scenario:** Step 1 succeeds (talk is now accepted in the database). Step 2 fails because the mail server is temporarily unavailable. The speaker is now accepted but never notified. The database says "accepted", the speaker's inbox says nothing. **The system is in an inconsistent observable state.**

### The Wrong Async Fix (Also Broken)

```python
# ❌ ALSO BROKEN — task can be enqueued before commit
def accept_submission(submission):
    submission.state = "accepted"
    submission.save()
    send_acceptance_email.delay(submission.id)  # Celery task dispatched to Redis
    # Django commits transaction here
```

**Failure scenario:** The database transaction **rolls back** (e.g., a constraint violation was triggered after the Celery dispatch). Redis already has the task. Celery processes it. The speaker gets a congratulatory email for a proposal that is still in "draft" state in the database.

### The Correct Approach: Transactional Outbox

The correct solution is exactly what pretalx implements. Study this carefully:

```python
# ✅ The correct pattern — see src/pretalx/submission/domain/submission.py
def accept_submission(submission):
    with transaction.atomic():
        submission.state = "accepted"
        submission.save()
        # The QueuedMail row is written IN THE SAME ATOMIC TRANSACTION
        QueuedMail.objects.create(
            event=submission.event,
            to=submission.speaker.email,
            subject="Your talk was accepted",
            text=render_acceptance_template(submission),
        )
    # Celery worker polls QueuedMail table separately and dispatches
```

**Why this is correct:** PostgreSQL guarantees ACID atomicity within a single transaction. If the outer transaction rolls back (for any reason), the `QueuedMail` row is also rolled back. The email is never sent. If the transaction commits, the `QueuedMail` row persists. The Celery worker will eventually find it and dispatch it. The two halves (state + notification intent) are always in sync.

---

## 3. ACID vs BASE — Knowing When Each Applies

| Property | ACID (Relational DBs) | BASE (NoSQL / Distributed) |
|:--- |:--- |:--- |
| **A** | **Atomic** — all operations succeed or all fail | **Basically Available** — system responds, even if stale |
| **C** | **Consistent** — schema constraints always valid | **Soft state** — state may change without explicit input |
| **I** | **Isolated** — concurrent transactions don't interfere | **Eventually consistent** — updates propagate over time |
| **D** | **Durable** — committed writes survive failures | — |

pretalx uses ACID guarantees from PostgreSQL for all critical state transitions. The Celery/Redis layer operates on BASE principles: the email *will eventually* be sent, but not necessarily within 50 milliseconds.

This is a deliberate, correct design choice for this domain. A speaker waiting 2 seconds longer to receive their acceptance email is tolerable. A speaker receiving an acceptance email for a rejected proposal is not.

---

## 4. Consistency Levels You Must Know

These terms appear in distributed systems interviews and in the documentation of every database from DynamoDB to CockroachDB:

| Level | Guarantee | Example |
|:--- |:--- |:--- |
| **Linearizability** (Strict) | Every operation appears instantaneous and in global real-time order. Slowest and safest. | Single-node PostgreSQL with `SERIALIZABLE` isolation |
| **Sequential Consistency** | All operations appear to happen in some sequential order that all nodes agree on. | Consensus protocols like Raft/Paxos |
| **Eventual Consistency** | If no new writes occur, all replicas will converge to the same value. May read stale data temporarily. | PostgreSQL read replicas, Redis replication |
| **Read-Your-Writes** | A client always sees its own writes, even if other clients may see stale data. | Session-based sticky routing to primary DB |

In pretalx's scaling architecture (described in `docs/upskill/07-interview-prep/03-system-design-from-this-repo.md`), when read replicas are introduced, you move the public agenda pages to **eventual consistency**. An organizer publishes a new schedule; CDN-cached readers may see the old version for up to 60 seconds. This is an explicit trade-off: latency and cost vs. strict global consistency. Know that you made this trade-off, and know what the window of inconsistency is.

---

## 5. Interview Application: When They Ask About Consistency

When an interviewer asks "How do you guarantee data consistency between your services?" — a junior answer is "I use transactions." A senior answer distinguishes between:

1. **Intra-service consistency:** ACID transactions within PostgreSQL.
2. **Cross-service consistency:** Transactional outbox to avoid dual-write problems.
3. **Read consistency windows:** Explicit cache TTLs with known staleness bounds.
4. **Distributed sagas:** For workflows spanning multiple autonomous services (e.g. if pretalx ever separated the mailer into its own microservice).

Grounding each of these in the pretalx architecture lets you give a concrete, code-anchored answer rather than abstract theory.

**Key file references:**
- [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) — The `set_submission_state` function and outbox writes inside `transaction.atomic()`.
- [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py) — The `QueuedMail` model that serves as the outbox table.
- [`src/pretalx/celery_app.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/celery_app.py) — The asynchronous consumer side.

---

## Self-Assessment Questions

1. You add a Redis cache in front of your PostgreSQL read queries. A speaker updates their biography. A visitor views their profile 3 seconds later from a CDN-cached page. What consistency level is this, and what is the maximum staleness window?

2. An organizer sends a bulk-acceptance email to 500 speakers. The Celery worker crashes after processing 200. When it restarts, what happens? Which property of the transactional outbox guarantees correctness here?

3. A colleague proposes replacing the `QueuedMail` table with a direct call to the Resend API inside the transaction. Explain precisely why this would break and under what network conditions.
