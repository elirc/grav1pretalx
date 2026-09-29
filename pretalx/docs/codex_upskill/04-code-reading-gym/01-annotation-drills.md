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

## 🏆 Self-Grading Rubric

When complete, rate your annotations against these performance indicators:
*   **Basic (Junior):** Correctly identified basic input/output types but missed the `is None` vs truthiness gotcha and side-effect limits.
*   **Solid (Mid-Level):** Correctly traced database-level side effects (updates/deletes) and wrote accurate JS equivalents using standard operators.
*   **Strong (Senior Staff):** Identified the N+1 risks, lazy evaluation boundaries, and explained why Django ORM does not support sliced deletes.
