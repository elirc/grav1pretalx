# Codex Onboarding & growth Curriculum: pretalx Training Lab
### For Software Engineers Ascending to Mid-Level and Senior Roles

Welcome to the Codex pretalx growth Lab. This curriculum is a rigorous, codebase-native onboarding program designed to transition junior software engineers to mid-level proficiency and prepare mid-level engineers for senior leadership and architectural responsibility. By studying this codebase as a primary textbook, you will master transferable engineering concepts: from multi-tenant scoping and declarative permission predicates to transactional outboxes and background workers.

---

## 🎯 Repo Identity

**pretalx** is a comprehensive multi-tenant event management platform tailored to organize conferences, call for papers (CFPs), reviewer scorecards, and scheduling timetables. Written in Python >= 3.13 and built on Django 6.0, Django REST Framework, Celery, and a Svelte frontend schedule editor, it acts as a robust production system. The application enforces strict multi-tenant boundary safety at the database layer using `django-scopes` and leverages the `rules` predicate authorization library. The entire application runs on traditional SQLite in development and scales to Postgres/Redis in production.

---

## 📖 The "Senior Mindset" Shift

As your software engineering path matures, your guiding question changes:
*   **Junior Mindset:** *"How do I make it work?"* (Aims to satisfy the immediate happy path, writing arbitrary syntax to resolve the ticket without errors).
*   **Mid-Level Mindset:** *"Is this the right pattern?"* (Ensures code conforms to structural guidelines, separates concerns, exposes clear APIs, and is backed by coverage).
*   **Senior Mindset:** *"What does this commit us to, who pays the cost, and how do we reduce risk?"* (Evaluates database locking thresholds, transactional failures, API backwards-compatibility, security boundary leaks, and data isolation failures).

---

## 🗺️ Curriculum Timelines

*   **The One Weekend Sprint:** Complete `00-fast-track.md` and `08-reference/verification-log.md`. You will configure your local workspace, run your first tests, and trace a submission flow.
*   **The Two Weeks Intensive:** Work through `01-codebase-cartography/05-key-flows.md` and `02-stack-and-language-mastery/`. Annotate core models and map request/response lifecycles.
*   **The Eight Weeks Ownership:** Complete the drills in `04-code-reading-gym/01-annotation-drills.md` and select a Good First Ticket to build from `06-contribution-practice/`.
*   **Ongoing Practice:** Refactor legacy models, conduct security audits using the checklists, and complete live-coding patterns.

---

## 🚦 Recommended Learning Paths

### Path A: Brand-New Junior Developer
*   **Focus:** Core CRUD mechanisms and modular MVC structures.
*   **Plan:** Proceed sequentially starting at `00-fast-track.md`. Focus heavily on learning forms, models, and trace tables in the reading gym.

### Path B: Stack-Familiar Junior
*   **Focus:** Django idioms and permissions enforcement.
*   **Plan:** Start at `02-stack-and-language-mastery/`, then dive into `03-architecture-and-patterns/03-validation-auth-and-permissions.md` to see real-world authorization.

### Path C: Mid-Level Engineer New to Repo
*   **Focus:** Onboarding to start shipping production changes.
*   **Plan:** Complete `01-codebase-cartography/` completely, and then pick a Medium-difficulty ticket in `06-contribution-practice/`.

### Path D: Senior Architect / Auditor
*   **Focus:** Operational safety, security, and scalability critiquing.
*   **Plan:** Read `03-architecture-and-patterns/06-architecture-critique.md` and `05-quality-engineering/05-security-checklist.md`.

---

## ⚙️ Documentation Conventions

*   **File Anchors:** Every assertion is linked directly to relative repository files, such as `[submission.py](../../src/pretalx/submission/models/submission.py#L89)`.
*   **Fake Code Labeling:** Any code constructed purely for illustration is explicitly labeled as `// Illustrative fake code`.
*   **Active Drills:** Every concept ends with a mandatory drill and a self-grading rubric.
