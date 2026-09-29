# Key Execution Flows

This module provides exhaustive, line-by-line runtime trace routes of the most critical operational paths in **pretalx**.

---

## Flow: Proposal Submission (CFP Wizard)

**Why this flow matters:** This is the entrypoint for speakers to submit proposals. It manages multi-step state caching, session memory validation, and queues acknowledgment emails.

**Open these files first:**
*   [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) - Manages multi-step HTTP session form state wizard.
*   [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) - Triggers state calculations and side-effects.

**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | `SubmitWizard` | [`wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) | User navigates to CfP path. Formwizard initializes session variables. | `WSGIRequest` | Low (depends on active `request.event` middleware context). |
| 2 | Form Cleaners | [`wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) | Standard forms clean parameters. Validates formats. | Cleaned fields dict | Fails if standard validation rules are violated. |
| 3 | Domain Service | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | Calls `create_submission` to persist the model row in `DRAFT` state. | `Submission` model instance | SQL integrity failure if invalid foreign keys are provided. |
| 4 | Signal Triggers | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | Dispatches `submission_state_change` signal. | Signals context dict | A custom plugin can fail or veto changes. |
| 5 | Celery Enqueue | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | Calls `queue_initial_mails` inside a post-commit transaction hook. | Celery task parameters | If Celery broker is offline, email notifications stall. |

**Validation and authorization:**
*   CfP wizard verifies deadline timeline rules in `is_cfp_open` permission predicate.
*   Track limits are verified in `available_tracks_for_submitter` ([`submission.py#L619-L633`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L619-L633)).

**Persistence and side effects:**
*   Saves speaker profile associations within `SpeakerRole` tables.
*   Fires custom signals that allow third-party plugin integrations.

**Tests that cover it:**
*   [`src/tests/cfp/test_cfp_wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/cfp/test_cfp_wizard.py)

**What juniors usually miss:**
*   Juniors assume that `create_submission` saves the row immediately with standard states. In reality, proposals default to `DRAFT` state during wizard cycles and only transition to `SUBMITTED` upon finalizing.

**What seniors notice:**
*   Seniors notice the 60-second delay countdown on `task_send_initial_mails`. This allows the speaker to correct mistakes without triggering immediate duplicated notifications.

**Drill:**
Open `src/pretalx/cfp/views/wizard.py` and trace how `SubmitWizard.done` fetches and combines cleaned form outputs.

**Self-grade:**
*   **Basic:** Understood the wizard steps but missed the background queue delay logic.
*   **Solid:** Correctly traced data states from Form classes to the database service.
*   **Strong:** Identified N+1 prefetching issues and why post-commit database callbacks prevent phantom email notifications on transaction failures.

---

## Flow: Organizer State Transition (Accepting a Talk)

**Why this flow matters:** Coordinates accepting speaker submissions, setting schedules, generating time slots, and queuing speaker accept mails.

**Open these files first:**
*   [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) - Coordinates `set_submission_state` and reconciles slots.
*   [`src/pretalx/orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py) - Triggering views.

**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Controller View | [`orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py) | Organizer clicks "Accept" in Orga dashboard. | HTTP POST | Minor (requires organizer permissions). |
| 2 | Model Delegate | [`models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py) | Model class calls `.accept()` delegate. | None | Delegates direct state mutations. |
| 3 | Domain Service | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | `set_submission_state` intercepts, triggers before-change signals. | State transition values | Exception thrown if signal plugins block. |
| 4 | Database Sync | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | Updates state fields, deletes or populates `TalkSlot` entries. | DB writes | Multi-tenant scoping crash if event scope fails. |
| 5 | Mail Queue | [`domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) | Invokes `send_state_mail` to render and persist outbox alerts. | `QueuedMail` instance | Mail rendered incorrectly if template is corrupt. |

**Validation and authorization:**
*   Permissions verified in `@rules.predicate def orga_can_change_submissions`.

**Persistence and side effects:**
*   Adjusts slots on WIP schedule dynamically using `update_talk_slots` ([`submission.py#L264-L298`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L264-L298)).

**Tests that cover it:**
*   [`src/tests/orga/views/test_orga_views_submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/orga/views/test_orga_views_submission.py)

**What juniors usually miss:**
*   Juniors assume that talk slots are only scheduled manually. Actually, slots are instantiated as blank WIP placeholders automatically when a talk is accepted.

**What seniors notice:**
*   Seniors notice the transaction-commit coordination: if any part of the talk slot reconciliation fails, the parent transaction rolls back completely, ensuring data consistency.
