# Codebase Verification Log

**Timestamp:** 2026-05-18  
**Auditor:** Antigravity (Senior Staff Engineer)

---

## Step 1: Identify the Project

*   **Language(s):** Python >= 3.13 (as specified in `pyproject.toml`'s `requires-python = ">=3.13"`).
*   **Framework(s) & Versions:** Django 6.0 (`"Framework :: Django :: 6.0"` inside classifiers in `pyproject.toml`), Celery 5.6, Django REST Framework 3.17, `rules` 3.5 for predicates, `django-scopes` 2.0.0 for multi-tenancy.
*   **Type of Repo:** A single modular web application comprising standard Django "apps" (separated by concern into subdirectories), along with a pre-built Vite-based Svelte/JS frontend editor module under `src/pretalx/frontend/schedule-editor`.
*   **Domain & Purpose:** Conference Organization Platform (CFP submission, review workflows, scheduling talks via visual editor, publishing interactive event timetables, and managing speaker/registrant mailers).

---

## Step 2: Structure Map

```
pretalx/
├── pyproject.toml              # Project configuration, dependencies (dev & optional), pytest options
├── justfile                    # Task runner definitions (setup, run, lint, tests, release checks)
├── src/                        # Source container
│   ├── manage.py               # Django CLI management entrypoint
│   └── pretalx/                # Core Django project package
│       ├── __init__.py         # Package declaration and version definition
│       ├── settings.py         # Main Django settings module
│       ├── urls.py             # Top-level routing connecting CFP, agenda, orga, and API
│       ├── agenda/             # Public schedule view and speaker list pages
│       ├── api/                # REST endpoints and serializers
│       ├── cfp/                # Call for Proposals submission flow and speaker profile editing
│       ├── common/             # Middlewares, DB utilities, signals, exception reporting, caching, common forms
│       ├── event/              # Event and organizer models/admin settings
│       ├── frontend/           # Rich JS/Svelte frontend scheduling drag-and-drop workspace
│       ├── mail/               # Email rendering, templating, outbox, and background dispatcher tasks
│       ├── orga/               # Heavyweight backend management dashboard for reviewers/administrators
│       ├── person/             # Custom Django User accounts, speaker profiles, preferences
│       ├── schedule/           # Timetables, draft vs public schedules, room and slot models
│       ├── static/             # Assets (CSS/JS modules) for traditional templates
│       └── submission/         # Submissions, CFP configurations, tags, review logic, comments, questions
└── doc/                        # Documentation directory
```

---

## Step 3: Top 20 Most Important Files

1.  [`src/pretalx/settings.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/settings.py) - Core configurations, middleware registry, databases, templates, and storage setup. **Audience:** Senior.
2.  [`src/pretalx/urls.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/urls.py) - Top-level routing registry separating concerns. **Audience:** Mid.
3.  [`src/pretalx/submission/models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py) - The central data model for conference talk proposals and states. **Audience:** Mid.
4.  [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) - Central state transition engine handling acceptance, rejection, and side-effects. **Audience:** Mid.
5.  [`src/pretalx/submission/rules.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py) - Permissions rules using predicate logic to authorize speaker, reviewer, and organizer actions. **Audience:** Mid.
6.  [`src/pretalx/event/models/event.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/event/models/event.py) - Represents the multi-tenant context boundary (the Event object). **Audience:** Senior.
7.  [`src/pretalx/common/middleware/event.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/event.py) - Core middleware resolving `request.event` and enforcing multi-tenant isolation scopes. **Audience:** Senior.
8.  [`src/pretalx/common/middleware/domains.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/middleware/domains.py) - Middleware handling custom domain lookup and routing. **Audience:** Senior.
9.  [`src/pretalx/person/models/user.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/person/models/user.py) - Extends standard Django User with preferences, administrators, and permissions. **Audience:** Mid.
10. [`src/pretalx/person/models/profile.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/person/models/profile.py) - Bridges user accounts with event-specific speaker biographies. **Audience:** Junior.
11. [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) - Form-wizard controller for proposal submissions. **Audience:** Mid.
12. [`src/pretalx/orga/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/submission.py) - Organizer views for proposal listings, metadata, and status toggles. **Audience:** Junior.
13. [`src/pretalx/orga/views/review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/orga/views/review.py) - Review dashboard, score grids, and reviewer comment view controllers. **Audience:** Mid.
14. [`src/pretalx/schedule/models/slot.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/schedule/models/slot.py) - Bridges submissions to particular rooms, times, and schedules. **Audience:** Mid.
15. [`src/pretalx/schedule/models/schedule.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/schedule/models/schedule.py) - The schedule revision coordinator tracking WIP vs published timetables. **Audience:** Senior.
16. [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py) - Model representing transactional outbox mail drafts and templating. **Audience:** Mid.
17. [`src/pretalx/api/views/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/api/views/submission.py) - Read/write REST endpoints for submissions with API-level token scoping. **Audience:** Mid.
18. [`src/pretalx/celery_app.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/celery_app.py) - Declares the Celery application and auto-discovers async background tasks. **Audience:** Senior.
19. [`src/pretalx/common/forms/renderers.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/common/forms/renderers.py) - Modular forms layout renderer matching Bootstrap UI patterns. **Audience:** Junior.
20. [`src/tests/conftest.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/tests/conftest.py) - Custom base fixtures and test configurations for pytest setup. **Audience:** Mid.

---

## Step 4: Domain Glossary

*   **Event:** A single conference or scheduling instance that groups CfPs, speakers, and submissions together. (Key Boundary: [`src/pretalx/event/models/event.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/event/models/event.py)).
*   **Submission:** A proposed talk or tutorial submitted by a speaker to a Call for Proposals. (Key Boundary: [`src/pretalx/submission/models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py)).
*   **Speaker Profile:** An event-specific projection of a user's biographical detail. (Key Boundary: [`src/pretalx/person/models/profile.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/person/models/profile.py)).
*   **CFP (Call for Papers / Proposals):** The active entry wizard config defining timelines, fields, and options. (Key Boundary: [`src/pretalx/submission/models/cfp.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/cfp.py)).
*   **Schedule:** A specific compilation of Talk Slots. An event has one "WIP" (Work In Progress) schedule and multiple published frozen schedules. (Key Boundary: [`src/pretalx/schedule/models/schedule.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/schedule/models/schedule.py)).
*   **Talk Slot:** An individual scheduled session connecting a Submission to a start time, duration, and Room. (Key Boundary: [`src/pretalx/schedule/models/slot.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/schedule/models/slot.py)).
*   **Review Phase:** A period during which reviews can be submitted, configuration of reviewer names visibility, and scoring parameters. (Key Boundary: [`src/pretalx/submission/models/review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/review.py)).
*   **Mail Template:** Pre-defined text files used to email speakers, accepted/rejected notifications. (Key Boundary: [`src/pretalx/mail/models.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/mail/models.py)).

---

## Step 5: Key Flows mapped

*   **Proposal Submission Flow:** Speaker registers → Enters wizard → Submits proposal. (Crosses `src/pretalx/cfp/views/wizard.py`, `src/pretalx/submission/domain/submission.py` `.create_submission`).
*   **State Transition Flow:** Organizer accepts/rejects submission. (Crosses `src/pretalx/orga/views/submission.py`, `src/pretalx/submission/domain/submission.py` `.set_submission_state`, queues emails via `queue_initial_mails`).
*   **Review Submission Flow:** Reviewer rates submission. (Crosses `src/pretalx/orga/views/review.py`, updates scoring via `recalculate_submission_scores` in `src/pretalx/submission/domain/review.py`).
*   **Talk Scheduling Flow:** Drag-and-drop or manual slot assignment. (Crosses Svelte drag-and-drop frontend, API update views, and calls `move_slot` inside `src/pretalx/schedule/domain/slot.py`).
*   **Scope Isolation Enforcement:** A user attempts to read database rows across multiple tenants. (Crosses `django-scopes` decorator in common middleware `src/pretalx/common/middleware/event.py`).

---

## Step 6: JS↔Python Bridge Points

1.  **Scope and Middleware:** Express router layers `req.event` vs Django middleware setting `request.event` and scoped databases.
2.  **Database Queries:** `Prisma` promise chains vs Django's *lazy evaluations* where `QuerySet` objects only run SQL once iterated.
3.  **Permissions Enforcement:** Express middleware arrays like `[auth, checkRole]` vs rule-based predicate authorization `@rules.predicate` in `rules.py`.
4.  **Form Wizards:** React multi-step state hooks vs Django's `SessionWizardView` standard implementation in the CFP path.
5.  **Event Workers:** Node.js worker threads / Bull queue processors vs Celery tasks using redis/postgres backends in `celery_app.py`.
6.  **Static Files and Assets Collection:** Webpack/Vite bundlers vs Django's `collectstatic` mixed with Whitenoise static cache.

---

## Step 7: Gaps & Infrastructure

*   **Service Layer:** Distinctly isolated to the `domain/` directories (e.g. `src/pretalx/submission/domain/submission.py`). This acts as an API-agnostic controller for data mutability.
*   **Task Queue:** Django can run synchronously in dev using `CELERY_TASK_ALWAYS_EAGER = True`. In prod, it connects to Celery using Redis or PostgreSQL database backends.
*   **API Serialization:** Managed by Django REST Framework (DRF) serializers under `src/pretalx/api/serializers/`.
*   **Frontend Bundle:** Composed of standard Django HTML templates for CFP and Orga side, complemented by a Svelte scheduling editor under `src/pretalx/frontend/schedule-editor`.
*   **Static Type Checking:** Type hints are scattered across newer service files, but strict static typing (mypy/pyright) is optional.

---

## Step 8: Environment Setup Audit

*   **Venv setup:** Run `python -m venv .venv` and activate via `.venv\Scripts\activate` (on Windows).
*   **Dependencies:** `pip install -e .[dev]` or via `uv sync` (if `uv` is installed, although not available by default on the host environment).
*   **Settings variable:** Pretalx loads config from `src/pretalx.example.cfg` which must be saved as `src/pretalx.cfg` or specified via `PRETALX_CONFIG_FILE`.
*   **Database Migrations:** Run `python src/manage.py migrate` to apply SQLite schemas.
*   **Starting server:** Run `python src/manage.py runserver` or `python src/manage.py devserver`.
*   **Testing suite:** Run `pytest src/tests/`.
