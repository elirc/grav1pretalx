# Onboarding Fast-Track & Initial Walkthrough

This weekend-sized fast track guides you from cloning the repository to running the local dev server, running unit tests, tracing a core flow, and implementing your first safe model edit.

---

## 🛠️ Workspace Installation & Setup (Windows)

Follow these steps to set up pretalx on your Windows machine.

### 1. Initialize Python Environment
*   **Command (Verified):**
    ```powershell
    python -m venv .venv
    ```
*   **Activation (Verified):**
    ```powershell
    .venv\Scripts\Activate.ps1
    ```
    *(Bridge Note: Keeps dependency packages separate from your global system environment, similar to local `node_modules`).*

### 2. Install Development Dependencies
*   **Command (Verified):**
    ```powershell
    pip install -e .[dev]
    ```
    *(Bridge Note: Installs the project locally in editable mode alongside all optional testing and quality packages).*

### 3. Setup Configuration
*   **Command (Verified):**
    ```powershell
    Copy-Item src/pretalx.example.cfg src/pretalx.cfg
    ```
    *(Bridge Note: Clones the default configuration, equivalent to setting up `.env` from `.env.example`).*

### 4. Run SQLite Migrations
*   **Command (Verified):**
    ```powershell
    python src/manage.py migrate
    ```
    *(Bridge Note: Creates SQLite database tables and indexes, matching `npx prisma migrate dev`).*

### 5. Create Superuser Account
*   **Command (Verified):**
    ```powershell
    python src/manage.py createsuperuser
    ```

### 6. Create Demo Event
*   **Command (Verified):**
    ```powershell
    python src/manage.py create_test_event
    ```

### 7. Run Server
*   **Command (Verified):**
    ```powershell
    python src/manage.py runserver
    ```

Log in to the administrative backend at `http://127.0.0.1:8000/orga/` using your superuser login.

---

## 🔎 First 10 Files to Open

Open these files in order to build your initial layout map:

1.  [`src/pretalx/settings.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/settings.py)
    *   **Landmark:** Review `INSTALLED_APPS` (lines 58–90) and the `MIDDLEWARE` chain (lines 476–491) to see request hooks.
2.  [`src/pretalx/urls.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/urls.py)
    *   **Landmark:** Trace how sub-modules register urls.
3.  [`src/pretalx/submission/models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py)
    *   **Landmark:** Study the core `Submission` database model, states (lines 14–22), and declarative rules permissions.
4.  [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py)
    *   **Landmark:** Review `set_submission_state` (lines 193–262). This is the core workflow mutation handler.
5.  [`src/pretalx/submission/rules.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py)
    *   **Landmark:** Examine standard rules predicates like `is_speaker` and `has_reviewer_access`.
6.  [`src/pretalx/common/middleware/event.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py)
    *   **Landmark:** Observe how `EventPermissionMiddleware` loads `request.event` and enforces scopes.
7.  [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py)
    *   **Landmark:** Trace form wizard steps.
8.  [`src/pretalx/orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py)
    *   **Landmark:** Trace `SubmissionStateChange` view class.
9.  [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py)
    *   **Landmark:** Look at outbox `QueuedMail` and templates models.
10. [`src/tests/conftest.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/conftest.py)
    *   **Landmark:** Review how mock events and client logins are initialized for pytest.

---

## 🔄 Core Flow to Trace: Accessing Public Agenda

1.  **Entrypoint:** User loads public URL `/{event_slug}/talk/{code}/` routing to agenda talk details view.
2.  **Middlewares:** `EventPermissionMiddleware` intercepts, parses `event_slug`, binds `request.event`, and enforces scope constraints.
3.  **Scope Verification:** Any subsequent queries on `Submission` must fall within the event boundaries, protecting against multi-tenant leaks.
4.  **Template Render:** Fetches talk properties (e.g. sorted speakers, active slot duration fallback) and compiles server-rendered templates.

---

## ⚡ First Small Safe Change to Attempt

Let's modify how default speaker bios fallback to a generic message when empty.

### Python Edit
Open [`src/pretalx/person/models/profile.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/person/models/profile.py):
Let's find biography retrieval and add a default fallback:
```python
    @property
    def biography_display(self):
        # Inferred change: Fallback to an empty string or standard display if empty
        return self.biography or ""
```

### JS/TS Equivalent Approach
```typescript
class SpeakerProfile {
  biography?: string;
  get biographyDisplay(): string {
    return this.biography ?? "";
  }
}
```

---

## 🧪 One Test to Run
Run this single command inside your virtual environment to verify submission models pass checks:
*   **Command (Verified):**
    ```powershell
    python -m pytest src/tests/submission/models/test_submission.py
    ```

---

## 🚫 What This Fast Track Does NOT Cover
*   Deploying to AWS or Kubernetes with PostgreSQL and Redis.
*   Modifying the rich visual Svelte drag-and-drop schedule planner widget.
*   Writing custom database migration scripts.
