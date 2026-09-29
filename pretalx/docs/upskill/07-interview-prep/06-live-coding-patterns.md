# Live-Coding Patterns

This guide prepares you for live-coding interview rounds by walking through three programming patterns found inside **pretalx**, comparing them side-by-side with JavaScript/TypeScript implementations.

---

## Pattern 1: Weighted Calculations & Aggregation

### The Real-World Challenge
In a review board, a candidate proposal receives multiple reviewer scores. We must calculate a single aggregated final score. However, each score category has a different weighted multiplier (e.g. "technical depth" weighs twice as much as "entertainment value"). We must filter categories based on the talk's Track and aggregate the total.

### 🐍 Python Implementation (pretalx)
From [`src/pretalx/submission/domain/review.py#L11-L24`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/review.py#L11-L24):
```python
def update_review_score(review):
    scores = list(
        review.scores.select_related("category").filter(
            category__in=review.submission.score_categories
        )
    )
    review.score = sum(s.value * s.category.weight for s in scores) if scores else None
    review.save()
```

### 🪟 JS/TS Equivalent (Node / Prisma)
```typescript
interface Score {
  value: number;
  category: {
    weight: number;
    trackIds: string[];
  };
}

function calculateScore(scores: Score[], trackId: string): number | null {
  const applicableScores = scores.filter(s => s.category.trackIds.includes(trackId));
  if (applicableScores.length === 0) return null;

  return applicableScores.reduce((sum, s) => sum + s.value * s.category.weight, 0);
}
```

### ⏱️ 15-Minute Challenge
Write a function that calculates the weighted average. If any score has a negative category weight, exclude it completely. If no scores remain, return `0`.
*   **Basic:** Able to loop and compute simple averages.
*   **Solid:** Uses list comprehensions (Python) or `.reduce()` (JS) cleanly with filter checks.
*   **Strong:** Correctly handles edge cases (like division by zero or empty arrays).

---

## Pattern 2: Multi-Step Input Wizard Redirection

### The Real-World Challenge
When walking users through multi-step forms (like the CFP Wizard), we must validate data sequentially. If a step fails validation, we must redirect back to that specific step with errors. If it passes, we store temporary progress in session memory and advance.

### 🐍 Python Implementation (pretalx)
From [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) (conceptual shape of Django's `SessionWizardView`):
```python
class SubmitWizard(SessionWizardView):
    def get_form(self, step=None, data=None, files=None):
        form = super().get_form(step, data, files)
        # Binds event-specific parameters to fields
        form.event = self.request.event
        return form

    def done(self, form_list, **kwargs):
        # Executes only after all steps pass validation
        submission = create_submission_from_forms(form_list)
        return render_success_page(submission)
```

### 🪟 JS/TS Equivalent (React / State Machine)
```typescript
type WizardState = {
  currentStep: number;
  formData: Record<string, any>;
};

function wizardReducer(state: WizardState, action: { type: "NEXT" | "PREV"; payload?: any }) {
  switch (action.type) {
    case "NEXT":
      return {
        currentStep: state.currentStep + 1,
        formData: { ...state.formData, ...action.payload },
      };
    case "PREV":
      return { ...state, currentStep: Math.max(0, state.currentStep - 1) };
    default:
      return state;
  }
}
```

### ⏱️ 15-Minute Challenge
Create a basic multi-step session manager class in Python that accepts step payloads, validates fields using dict checks, and only completes when all required keys are filled.
*   **Basic:** Implements simple step switching.
*   **Solid:** Handles partial progress storage in a mock session dictionary.
*   **Strong:** Implements validation logic that prevents skipping steps.

---

## Pattern 3: Safe DDL/DML Migrations and Data Backfills

### The Real-World Challenge
You need to change a column name from `talk_length` to `duration` in a high-traffic production table containing 10,000,000 rows. A direct rename locks the database table and causes downtime.

### 🐍 Python/Django Strategy
1.  Add a new field `duration` to the model.
2.  Deploy step: Update model save override to write to both fields:
    ```python
    def save(self, *args, **kwargs):
        if self.talk_length:
            self.duration = self.talk_length
        super().save(*args, **kwargs)
    ```
3.  Run a background Celery batch script to copy historical rows from `talk_length` to `duration` in chunks of 5,000 to prevent long-running table locks.
4.  Once backfill is complete, deploy code reading strictly from `duration`.
5.  Deploy final cleanup migration dropping the old `talk_length` column.

### 🪟 JS/TS Equivalent (Prisma / SQL)
```sql
-- Step 1: Add new column
ALTER TABLE "Submission" ADD COLUMN "duration" INTEGER;

-- Step 2: Backfill in batches
UPDATE "Submission" SET "duration" = "talk_length" WHERE "id" IN (
  SELECT "id" FROM "Submission" WHERE "duration" IS NULL LIMIT 5000
);
```

### ⏱️ 15-Minute Challenge
Explain how you would write a zero-downtime database migration to split a compound `speaker_name` column into separate `first_name` and `last_name` fields.
*   **Basic:** Suggests dropping the old column and writing a basic split function.
*   **Solid:** Proposes a multi-step deploy process using a temporary write-both phase.
*   **Strong:** Explicitly addresses edge cases (like middle names or single-word names) and database locking concerns.
