# Weekend Fast-Track Setup & Code Walk

If you only have a single weekend to explore, this guide is designed to get you from absolute zero to running the pretalx server, submitting a test CFP talk, running the test suite, and making your first safe change.

---

## 🛠️ Complete Local Setup (Windows-Focused)

Follow these steps to set up pretalx on your Windows development environment.

### 1. Create a Python Virtual Environment
Unlike JS where dependencies go to local `node_modules` inside the workspace directory, Python isolates dependencies inside a virtual environment folder (`.venv`).
*   **Command (Verified):**
    ```powershell
    python -m venv .venv
    ```
*   **Activation (Verified):**
    ```powershell
    .venv\Scripts\Activate.ps1
    ```
    *(JS Comparison: This is equivalent to setting up `node_modules` but keeps all binaries on an isolated environment PATH).*

### 2. Install Dependencies
We will install pretalx in editable development mode along with its dev dependencies.
*   **Command (Verified):**
    ```powershell
    pip install -e .[dev]
    ```
    *(JS Comparison: Equivalent to `npm install` based on a local `package.json` package configuration).*

### 3. Initialize Configuration
Pretalx looks for configuration at `src/pretalx.cfg`. We will copy the provided example cfg.
*   **Command (Verified):**
    ```powershell
    Copy-Item src/pretalx.example.cfg src/pretalx.cfg
    ```
    *(JS Comparison: Equivalent to copying `.env.example` to `.env`).*

### 4. Run Database Migrations
Pretalx uses SQLite by default in development mode. We will execute migrations to create the database schema.
*   **Command (Verified):**
    ```powershell
    python src/manage.py migrate
    ```
    *(JS Comparison: Equivalent to running `npx prisma migrate dev` or `npx sequelize db:migrate`).*

### 5. Create a Superuser
We need an administrator account to log into the Orga management panel.
*   **Command (Verified):**
    ```powershell
    python src/manage.py createsuperuser
    ```

### 6. Create Seed (Demo) Data
Pretalx provides a management command to create a demo event with tracks, rooms, submission types, and speakers.
*   **Command (Verified):**
    ```powershell
    python src/manage.py create_test_event
    ```
    *(JS Comparison: Equivalent to running a database seed script `node seed.js` or `npx prisma db seed`).*

### 7. Start the Development Server
Launch Django's built-in single-threaded development server.
*   **Command (Verified):**
    ```powershell
    python src/manage.py runserver
    ```
    *(JS Comparison: Equivalent to `npm run dev` in Express or Next.js projects).*

Now open your browser to `http://127.0.0.1:8000/orga/` and log in with your superuser credentials.

---

## 🔍 First 10 Files to Open (In Order)

To understand how pretalx works, open these files in your editor and look for the specific landmarks:

1.  [`src/pretalx/settings.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/settings.py)
    *   **What to look for:** Look at lines 58–90 for the `INSTALLED_APPS` registry. Notice how the codebase is split into modular Django apps (`agenda`, `cfp`, `orga`, `submission`). Examine the middleware stack in `MIDDLEWARE` (lines 476–491) to see the processing chain.
2.  [`src/pretalx/urls.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/urls.py)
    *   **What to look for:** Look at the main routing registry. Note how patterns are delegated to sub-applications via `include()` (e.g. `orga/` and `api/`).
3.  [`src/pretalx/submission/models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py)
    *   **What to look for:** The `Submission` model (lines 89–213). Notice the `state` Choices field, permissions mapped declaratively in the `Meta.rules_permissions` dictionary, and the lazy-loaded cached properties.
4.  [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py)
    *   **What to look for:** The `set_submission_state` method (lines 193–262). Observe how a proposal transitions from `SUBMITTED` to `ACCEPTED`, how talk slots are updated, signals are dispatched, and emails queued.
5.  [`src/pretalx/submission/rules.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py)
    *   **What to look for:** The `@rules.predicate` functions like `is_speaker` and `has_reviewer_access`. This is pretalx's core authorization layer.
6.  [`src/pretalx/common/middleware/event.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py)
    *   **What to look for:** The `EventPermissionMiddleware`. See how it resolves the event slug from the URL path, binds `request.event`, and enforces database isolation scopes via `django-scopes`.
7.  [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py)
    *   **What to look for:** How the multi-step proposal submission form is modeled as a wizard.
8.  [`src/pretalx/orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py)
    *   **What to look for:** Look for `SubmissionStateChange` view class. Notice how it intercepts state transitions and enforces permission parameters.
9.  [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py)
    *   **What to look for:** The `QueuedMail` and `MailTemplate` models. Notice how emails are prepared as drafts before Celery releases them.
10. [`src/tests/conftest.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/conftest.py)
    *   **What to look for:** The fixture definitions like `event`, `submission`, `orga_client`. Essential to understand how tests set up states.

---

## 🔄 Two End-to-End Flows to Trace

### Flow 1 (Write Flow): Submitting a Proposal (CFP Wizard)
1.  **Entrypoint:** User navigates to `/event-slug/cfp/` which routes to [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py).
2.  **Processing Steps:**
    *   The `SubmitWizard` subclasses Django formtools `SessionWizardView`, tracking draft progress in session memory.
    *   Upon completion, `done()` (inside `SubmitWizard`) calls `create_submission` in [`src/pretalx/submission/domain/submission.py#L36-L104`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L36-L104).
    *   `create_submission` saves the `Submission` row, binds the speaker, and queues transactional speaker confirmation emails via `queue_initial_mails`.
3.  **Side-Effects:** A Celery task `task_send_initial_mails` is triggered after a 60-second countdown delay to dispatch confirmation alerts.

### Flow 2 (Read Flow): Viewing Published Talk Details (Public Agenda)
1.  **Entrypoint:** A public attendee requests `/{event_slug}/talk/{code}/` which routes to [`src/pretalx/agenda/views/talk.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/agenda/views/talk.py).
2.  **Processing Steps:**
    *   `EventPermissionMiddleware` loads `request.event`.
    *   The view retrieves the submission by its unique `code` within the scoped boundaries of `request.event` (`django-scopes` enforces that no SQL queries can cross events).
    *   The model property `public_slots` returns slot availability on the current frozen schedule.
3.  **Side-Effects:** None (pure read flow), but triggers standard Django template rendering injecting talk variables.

---

## ⚡ First Small Safe Change to Attempt

Let's modify the default talk duration fallback value inside the `Submission` model!

### Python Change
Open [`src/pretalx/submission/models/submission.py#L375-L379`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L375-L379):
```python
    def get_duration(self) -> int:
        if self.duration is None:  # We permit zero-length duration
            return self.submission_type.default_duration
        return self.duration
```
Let's add a default override or a logging statement:
```python
    def get_duration(self) -> int:
        if self.duration is None:  # We permit zero-length duration
            # Safe illustrative tweak: Default to a minimum fallback of 30 mins if not specified
            return self.submission_type.default_duration or 30
        return self.duration
```

### JS/TS Equivalent Approach
If we were building this in a Node/TypeScript NestJS or Express system, this would be a simple class getter or a schema validator default fallback:
```typescript
class Submission {
  duration?: number;
  submissionType: SubmissionType;

  getDuration(): number {
    return this.duration ?? (this.submissionType.defaultDuration || 30);
  }
}
```

---

## 🧪 One Test to Run

Verify that your environment works by executing a single test file using pytest.
*   **Command (Verified):**
    ```powershell
    pytest src/tests/submission/test_submission_models.py
    ```

---

## 🗣️ Teach-Back Exercise

To solidify your fast-track experience, explain these three concepts to an imaginary teammate in 5 minutes:
1.  Why pretalx requires `request.event` to be bound in its middleware stack before running any database SQL query.
2.  The mutable default argument gotcha in Python (e.g. `def add_speaker(..., speakers=[])`) and why pretalx uses `speakers=()` instead.
3.  How Celery async workers are coordinated in `celery_app.py` for deferred tasks like email alerts.

---

## 🚫 What This Fast Track Does NOT Cover

*   Setting up PostgreSQL or Redis for high-concurrency production deployments (uses SQLite and synchronous Celery in memory instead).
*   Writing complex custom plugins or extending the Svelte schedule editor widget.
*   Tackling permission boundaries and custom predicate rule definitions.
