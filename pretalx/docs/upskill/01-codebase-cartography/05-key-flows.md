# Key Execution Flows

This module provides exhaustive, line-by-line runtime trace routes of the three most critical operational paths in the **pretalx** conference application.

---

## 1. Flow: Proposal Submission (CFP Wizard)

**Why this flow matters:** This is the primary ingestion point for all conference speaker proposals. It spans multi-step session wizard validation, user profile binding, and asynchronous email scheduling.
**JS/TS equivalent pattern:** Next.js Server Actions or Express controller coordinating a multi-step React wizard and saving via a Prisma transaction.

**Open these files first:**
*   [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) — Handles the HTTP session wizard controller logic.
*   [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) — Coordinates database writes, invite code validation, and Celery triggers.

### Trace Path
| Step | Owner | File:Lines | What happens | Data shape | JS equivalent | Risk |
|:--- |:--- |:--- |:--- |:--- |:--- |:--- |
| 1 | `SubmitWizard` | [`wizard.py#L90-L120`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py#L90-L120) | Intercepts HTTP request, verifies if CFP is open and loads active steps. | `WSGIRequest` | `NextRequest` checking session | Minor (requires active `request.event`). |
| 2 | `SubmitWizard` | [`wizard.py#L182-L215`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py#L182-L215) | Process step form validation. Cleans data inputs. | Form fields dict | `yup` or `zod` schema check | Validation failures return view with errors. |
| 3 | `SubmitWizard` | [`wizard.py#L294-L352`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py#L294-L352) | Calls `done()` on final step. Instantiates a `Submission` model row. | `Submission` | Prisma write call | Model is in `DRAFT` state during this phase. |
| 4 | Service layer | [`submission.py#L36-L66`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L36-L66) | Calls `create_submission` service to persist and apply initial side-effects. | `Submission` db row | Service layer controller write | Fails if database constraints are violated. |
| 5 | Celery task | [`submission.py#L159-L168`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L159-L168) | Triggers `queue_initial_mails` inside a post-commit transaction hook. | Task ID | Bull queue dispatch | **GIL/Task Delay:** If Celery worker crashes, outbox notifications stall. |

**Validation and authorization:**
*   CfP wizard deadlines verified in `is_cfp_open` decorator.
*   Access codes and track constraints checked in `available_tracks_for_submitter` ([`submission.py#L619-L633`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L619-L633)).

**Persistence and side effects:**
*   Saves speaker bindings inside the through model `SpeakerRole`.
*   Fires Django `submission_state_change` signals.

**Tests that cover it:**
*   [`src/tests/cfp/test_cfp_wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/cfp/test_cfp_wizard.py)

---

## 2. Flow: State Transition Flow (Accept/Reject Proposal)

**Why this flow matters:** Triggers crucial workflow mutations where organizer decisions are calculated, talk slot schedules are created or deleted, and notification templates are rendered.
**JS/TS equivalent pattern:** Admin action handler resolving state transitions and updating schema models.

**Open these files first:**
*   [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) — Exposes state handlers `.accept`, `.reject`, and `set_submission_state`.
*   [`src/pretalx/orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py) — Handles state mutation HTTP triggers.

### Trace Path
| Step | Owner | File:Lines | What happens | Data shape | JS equivalent | Risk |
|:--- |:--- |:--- |:--- |:--- |:--- |:--- |
| 1 | View Controller | [`views/submission.py#L380-L400`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py#L380-L400) | Organizer triggers submission state update from Orga dashboard. | HTTP POST parameters | Express route controller | Minor (requires organizer permissions). |
| 2 | Model Actions | [`models/submission.py#L410-L417`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L410-L417) | Calls model method `submission.accept()`. | None | Prisma method | Leads directly to domain service. |
| 3 | Domain Service | [`domain/submission.py#L193-L215`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L193-L215) | Verifies `before_submission_state_change` signal rules. | Veto list | Express pre-hook | A plugin can veto the state transition. |
| 4 | State Resolver | [`domain/submission.py#L227-L238`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L227-L238) | Updates `state` column, clears `pending_state`, and saves model. | Updated `Submission` | Prisma save | If save fails, state is rolled back. |
| 5 | Schedule Sync | [`domain/submission.py#L264-L298`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L264-L298) | Calls `update_talk_slots` to adjust slots on the current WIP schedule. | `TalkSlot` updates | Room slots registry sync | Multi-tenant leak if scoped query fails. |
| 6 | Email outbox | [`domain/submission.py#L354-L375`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L354-L375) | Renders decision notification template and saves as outbox drafts. | `QueuedMail` row | Transactional mail queue | Multi-tenant boundary leak if email routes wrong event context. |

**Validation and authorization:**
*   Permission predicates authorized via `@rules.predicate def orga_can_change_submissions` inside `rules.py`.

**Tests that cover it:**
*   [`src/tests/orga/views/test_orga_views_submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/orga/views/test_orga_views_submission.py)

---

## 3. Flow: Proposal Review & Scoring

**Why this flow matters:** Reviewers rate talk pitches based on weighted criteria (e.g. alignment, tech-depth). Score categories are computed dynamically based on the proposal's selected Track.
**JS/TS equivalent pattern:** Drag-and-drop feedback components updating DB metrics and aggregates.

**Open these files first:**
*   [`src/pretalx/submission/domain/review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/review.py) — Handles score calculations.
*   [`src/pretalx/orga/views/review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/review.py) — Proposal review dashboard and views.

### Trace Path
| Step | Owner | File:Lines | What happens | Data shape | JS equivalent | Risk |
|:--- |:--- |:--- |:--- |:--- |:--- |:--- |
| 1 | Review view | [`orga/views/review.py#L120-L160`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/review.py#L120-L160) | Reviewer fills scores and saves review page. | POST data dictionary | Form submission parser | Throws errors if validation fails. |
| 2 | Score calculator | [`domain/review.py#L11-L24`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/review.py#L11-L24) | Recomputes review score value weighted by its active categories. | `score` float value | Weighted average score | If track is updated, score categories must be recalculated. |
| 3 | Aggregators | [`models/submission.py#L549-L560`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L549-L560) | Cached properties `median_score` and `mean_score` recalculate statistics. | Medians/Means | Array reduce metrics | N+1 performance bottleneck if evaluated in lists without select_related. |

**Tests that cover it:**
*   [`src/tests/orga/views/test_orga_views_review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/orga/views/test_orga_views_review.py)

---

## ⚠️ What a JS Developer Would Assume (That's Wrong Here)

1.  **"Querysets run instantly."** 
    In Node/Express, when you call `await prisma.submission.findMany()`, it triggers the query immediately. In pretalx, when you construct `talks = Submission.objects.filter(is_featured=True)`, **no SQL runs yet**. If you pass this queryset directly to a context without iterating it, you skip execution.
2.  **"Shared middleware can leak data."**
    In Express, if your multi-tenant router has bugs, one sub-request could theoretically query other event tables. In pretalx, **django-scopes** locks the database connection using local context variables (`request.event`). Running queries outside active scopes throws an explicit exception, rendering cross-tenant leaks impossible at the ORM layer.
