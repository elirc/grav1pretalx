# Concurrency Safety & Optimistic Locking

**Supplementary Module — cloddocs 03**  
**Prerequisite Reading:** `docs/upskill/03-architecture-and-patterns/05-pattern-catalog.md` (Pattern 3: State Machine transitions)

---

## The Problem the Existing Material Misses

The pattern catalog correctly describes how pretalx transitions submission states through a domain service, wrapped in `transaction.atomic()`. What it does not discuss is **what happens when two people trigger state changes at exactly the same time.**

This is not a theoretical concern. Pretalx is used by conferences where organizers from different continents review the same pool of submissions simultaneously. Two organizers can look at the same submission dashboard, both decide to accept the same talk, and both click "Accept" within 500 milliseconds of each other.

Without deliberate engineering, one of these writes silently overwrites the other. In some cases, the second write restores the data to its pre-first-write state, making it appear as though the first action never happened.

---

## 1. The Lost Update Problem

The classic database concurrency problem:

```
Time →

Organizer A:                     Organizer B:
  READ submission (state=submitted)
                                   READ submission (state=submitted)
  Process...                       Process...
  WRITE (state=accepted)
                                   WRITE (state=rejected)   ← Wins. A's change lost.
```

Organizer B's write wins simply because it completed last. Neither the system nor Organizer A was warned. The talk is now rejected, but Organizer A believes it was accepted and already told the speaker verbally.

This problem class is called a **Lost Update** and is one of the concurrency anomalies that database isolation levels are designed to prevent.

---

## 2. Database Isolation Levels — What They Actually Protect

PostgreSQL offers four isolation levels. They prevent different concurrency anomalies:

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Lost Update |
|:--- |:---: |:---: |:---: |:---: |
| **Read Uncommitted** | Possible | Possible | Possible | Possible |
| **Read Committed** (default) | Prevented | Possible | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Prevented | Prevented |
| **Serializable** | Prevented | Prevented | Prevented | Prevented |

**Django's default is Read Committed.** This means the lost update scenario above is fully possible in a standard pretalx deployment unless additional application-level protection is added.

> [!IMPORTANT]
> Moving the entire application to `SERIALIZABLE` isolation is not the answer. Serializable transactions have significantly higher lock contention and abort rates. The correct approach is to add concurrency control only where concurrent writes are a real risk.

---

## 3. Two Strategies for Solving the Lost Update

### Strategy A: Pessimistic Locking (`SELECT FOR UPDATE`)

Lock the row at read time. No other transaction can read or write this row until the lock is released.

```python
from django.db import transaction

def accept_submission_pessimistic(submission_pk, user):
    with transaction.atomic():
        # This SQL becomes: SELECT ... FROM submission WHERE id=X FOR UPDATE
        submission = Submission.objects.select_for_update().get(pk=submission_pk)
        
        if submission.state != "submitted":
            raise ValueError(f"Cannot accept — submission is already in state: {submission.state}")
        
        submission.state = "accepted"
        submission.save()
        # Lock is released when the transaction.atomic() block exits
```

**When Organizer B tries to run this simultaneously:** PostgreSQL blocks Organizer B's `SELECT FOR UPDATE` until Organizer A's transaction commits. When it finally reads the row, the state is `accepted` and the `ValueError` fires immediately.

**Trade-offs of pessimistic locking:**
- ✅ Simple to reason about. Guarantees no lost updates.
- ❌ Creates lock contention. Under load, queries queue up waiting for locks.
- ❌ Deadlock risk: if code path A locks row 1 then row 2, and code path B locks row 2 then row 1, they deadlock.
- ❌ Held locks block reads too, slowing dashboard page loads.

Use pessimistic locking when: concurrent writes on the same row are frequent and transactions are short.

### Strategy B: Optimistic Locking (Version Counters)

Do not lock at read time. Instead, include a version number in the write condition. If the version has changed between read and write, abort and retry.

```python
# Model change: add a version field
class Submission(models.Model):
    state = models.CharField(...)
    version = models.PositiveIntegerField(default=0)  # ← Added
    # ...

def accept_submission_optimistic(submission_pk, user, expected_version):
    # Attempt to update ONLY if the version matches what we read
    updated_count = Submission.objects.filter(
        pk=submission_pk,
        version=expected_version,  # ← The optimistic check
        state="submitted",
    ).update(
        state="accepted",
        version=expected_version + 1,  # ← Increment atomically
    )
    
    if updated_count == 0:
        # Either the state changed, or someone else updated it first
        raise ConcurrentModificationError(
            "This submission was modified by another user. Please refresh and try again."
        )
```

**What happens with Organizer A and B:**
1. Both read version `7`.
2. Organizer A submits first: `UPDATE ... WHERE version=7` → 1 row updated, version becomes 8. ✅
3. Organizer B submits: `UPDATE ... WHERE version=7` → 0 rows updated. The `ConcurrentModificationError` fires. The UI shows "Someone else changed this submission. Please refresh." ✅

**Trade-offs of optimistic locking:**
- ✅ No blocking. Reads are fast. Concurrency is high.
- ✅ No deadlock risk.
- ❌ Conflict detection is after the fact. UI must handle and communicate the failure gracefully.
- ❌ Useless when the same row is written with very high frequency (high contention → high abort rate → retry storms).

Use optimistic locking when: concurrent writes are rare but possible, and a short retry or user notification is an acceptable response.

---

## 4. The Schedule Editor: A Real Concurrency Case Study

The pretalx schedule editor (the Svelte drag-and-drop frontend under `src/pretalx/frontend/schedule-editor`) presents the highest-risk concurrency scenario in the entire application. Consider:

- Two conference co-organizers are both editing the schedule grid simultaneously.
- Co-organizer A drags "Introduction to Rust" from Room A to Room B, 11am slot.
- Co-organizer B drags "Advanced WebAssembly" to Room B, 11am slot at the same time.

Without slot-level concurrency control, the second drag-and-drop completes, and the slot contains whatever the last API write contained — leaving a dangling reference to the first talk in an inconsistent position.

### What the API Needs

Every `PATCH /api/events/{slug}/talks/{code}/` or slot update endpoint should:

1. **Accept and return a `version` or `updated_at` timestamp** on the `TalkSlot` resource.
2. **Include the client's last-known version in the write request** (as an `If-Match` HTTP header or request body field).
3. **Reject the write with HTTP `409 Conflict`** if the server-side version does not match.

The HTTP standard has a mechanism for this:

```http
PATCH /api/events/pycon2026/talks/XJ34K/ HTTP/1.1
Content-Type: application/json
If-Match: "slot-etag-v47"

{ "room": "B", "start": "2026-10-15T11:00:00Z" }
```

If the slot has been updated since the client last loaded it, the server responds:

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{
  "error": "conflict",
  "message": "This slot was modified by another organizer. Reload the schedule to see current state.",
  "current_version": "slot-etag-v49"
}
```

---

## 5. Idempotency — The Third Concurrency Tool

Beyond lost updates, there is a second concurrency problem: **duplicate operations**.

A user clicks "Accept" and the network is slow. Their browser retries the request. Pretalx receives two identical acceptance requests. If the second is processed after the first commit, the state machine runs again on an already-accepted submission, potentially sending a second congratulatory email.

**Idempotency** means: executing the same operation multiple times produces the same result as executing it once.

For state transitions, the simplest idempotency guarantee is the guard clause:

```python
def accept_submission(submission):
    if submission.state == "accepted":
        # Already in the target state. Return success without side effects.
        return  
    if submission.state not in ("submitted", "confirmed"):
        raise InvalidTransitionError(...)
    # Proceed with state change and side effects
```

For API endpoints, the more robust pattern uses a client-generated **idempotency key**:

```http
POST /api/events/pycon2026/submissions/XJ34K/accept/
Idempotency-Key: "a4f3c2e1-organizer-A-1747555200"
```

The server stores a mapping from `Idempotency-Key → response`. If the same key arrives twice, return the cached first response without reprocessing. Stripe, PayPal, and most financial APIs use this exact mechanism.

---

## 6. JS/TS Equivalents

| Python / Django concept | JS / Prisma equivalent |
|:--- |:--- |
| `select_for_update()` | `prisma.$queryRaw\`SELECT ... FOR UPDATE\`` |
| `filter(version=X).update(version=X+1)` | `prisma.submission.updateMany({ where: { id, version }, data: { version: version + 1 } })` |
| `transaction.atomic()` | `prisma.$transaction(async (tx) => { ... })` |
| HTTP `If-Match` header | HTTP `If-Match` header (framework-agnostic) |
| `updated_count == 0` check | `result.count === 0` check from `updateMany` |

---

## Self-Assessment Questions

1. Two reviewers submit scores for the same submission at the same moment. The `Review` model has a unique constraint on `(submission, user)`. Does this constraint prevent the lost update problem? Explain why or why not.

2. You are implementing the slot-assignment endpoint for the schedule editor. The product manager says the response time must be under 50ms at p99. Should you use pessimistic or optimistic locking? Justify your choice in terms of lock contention, abort rate, and user experience.

3. A Celery worker picks up an `accept_submission` task. Halfway through, the network connection to PostgreSQL drops. The task retries automatically. What could go wrong, and how does the Celery `acks_late` setting relate to this risk?
