# Code Reading Annotation Drills

This gym module contains real code snippets from **pretalx**. Your task is to analyze and annotate each drill according to the structured inputs, outputs, invariants, and side-effects, comparing them to standard JS equivalents.

---

## Drill 1: The Tag Permission Predicate

### Source Code
From [`src/pretalx/submission/rules.py#L11-L17`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py#L11-L17):
```python
@rules.predicate
def reviewer_can_create_tags(user, obj):
    event = obj.event
    return bool(
        event.active_review_phase
        and event.active_review_phase.can_tag_submissions == "create_tags"
    )
```

### 📝 Your Annotation Sheet
Fill out this schema before checking the solution:
1.  **Inputs (Type & Shape):**
2.  **Outputs (Type & Shape):**
3.  **Core Invariant (What must always be true?):**
4.  **Implicit Dependencies:**
5.  **Express/React Equivalent:**

---

### Key & Solution
*   **Inputs:** `user` (User model instance or AnonymousUser), `obj` (Any model with an `.event` ForeignKey, typically a Submission).
*   **Outputs:** `bool` (True if reviewer can create tags, else False).
*   **Core Invariant:** Authorizes reviewer actions *only* if the currently active review phase permits explicit tag creation ("create_tags").
*   **Implicit Dependencies:** `obj.event.active_review_phase` must be loaded. If missing or `None`, the check fails.
*   **Express/React Equivalent:** A custom permissions middleware or hook:
    ```javascript
    function reviewerCanCreateTags(user, obj) {
      const activePhase = obj.event.activeReviewPhase;
      return !!(activePhase && activePhase.canTagSubmissions === "create_tags");
    }
    ```

---

## Drill 2: Fallback Duration Evaluator

### Source Code
From [`src/pretalx/submission/models/submission.py#L375-L379`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L375-L379):
```python
    def get_duration(self) -> int:
        if self.duration is None:  # We permit zero-length duration
            return self.submission_type.default_duration
        return self.duration
```

### 📝 Your Annotation Sheet
1.  **Inputs:**
2.  **Outputs:**
3.  **Why checking `is None` explicitly matters in Python:**
4.  **JS Equivalent Pattern:**

---

### Key & Solution
*   **Inputs:** `self` (Submission model instance).
*   **Outputs:** `int` (Duration in minutes).
*   **Why None checking matters:** In Python, checking `if not self.duration` is a bug because zero (`0`) evaluates as falsy. Since zero-length durations are valid (e.g. for lightning talks), checking `is None` ensures we only fall back to the default if no value has been recorded at all.
*   **JS Equivalent Pattern:** Uses the nullish coalescing operator `??`:
    ```javascript
    getDuration() {
      return this.duration ?? this.submissionType.defaultDuration;
    }
    ```

---

## Drill 3: WIP Talk Slot Reconciliation

### Source Code
From [`src/pretalx/submission/domain/submission.py#L264-L298`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L264-L298):
```python
def update_talk_slots(submission):
    wip = submission.event.wip_schedule
    talks = wip.talks.filter(submission=submission)
    scheduling_allowed = (
        submission.state in SubmissionStates.accepted_states
        or submission.pending_state in SubmissionStates.accepted_states
    )

    if not scheduling_allowed:
        talks.delete()
        return

    diff = talks.count() - submission.slot_count
    if diff > 0:
        to_delete = list(
            talks.order_by("start", "room", "is_visible")[:diff].values_list(
                "id", flat=True
            )
        )
        wip.talks.filter(pk__in=to_delete).delete()
    elif diff < 0:
        for __ in range(-diff):
            wip.talks.create(submission=submission)
    talks.update(is_visible=submission.state == SubmissionStates.CONFIRMED)
```

### 📝 Your Annotation Sheet
1.  **Inputs:**
2.  **Side Effects:**
3.  **Explain the `diff > 0` logic in plain English:**
4.  **Why slice `[:diff]` uses a subquery rather than direct delete:**
5.  **JS Equivalent Pattern:**

---

### Key & Solution
*   **Inputs:** `submission` (Submission model instance).
*   **Side Effects:** Mutates database state by deleting, creating, or updating `TalkSlot` rows attached to the Work-In-Progress (`wip`) schedule.
*   **Plain English Logic:** Compares the actual number of scheduled slots against the submission's target `slot_count`. If we have too many slots, it deletes the extra ones (prioritizing unscheduled ones). If we don't have enough, it creates raw empty slots.
*   **Why subquery is used:** Django ORM does not support calling `.delete()` directly on sliced queries (like `[:diff]`). We must pull the Primary Keys (`values_list("id", flat=True)`) into memory and execute a bulk delete on those IDs.
*   **JS Equivalent:** A Prisma transaction synchronizing child records:
    ```typescript
    async function updateTalkSlots(submission) {
      const wip = await prisma.schedule.findFirst({ where: { eventId: submission.eventId, version: null } });
      const talks = await prisma.talkSlot.findMany({ where: { submissionId: submission.id, scheduleId: wip.id } });
      ...
    }
    ```

---

## 🏆 Self-Grading Rubric

When complete, rate your annotations against these performance indicators:
*   **Basic (Junior):** Correctly identified basic input/output types but missed the `is None` vs truthiness gotcha and side-effect limits.
*   **Solid (Mid-Level):** Correctly traced database-level side effects (updates/deletes) and wrote accurate JS equivalents using standard operators.
*   **Strong (Senior Staff):** Identified the N+1 risks, lazy evaluation boundaries, and explained why Django ORM does not support sliced deletes.
