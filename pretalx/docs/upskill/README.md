# Codebase Training Lab — pretalx Edition
## Full-Stack Engineering Mastery via Open Source Codebase Analysis
### Target: JS/TS Developers Learning Python & Preparing for CRUD Enterprise Roles

Welcome to the **pretalx Training Lab**. This interactive curriculum leverages a battle-tested, open-source Python/Django conference management project to accelerate your journey from a JavaScript/TypeScript background to advanced Python/Django proficiency, codebase cartography, and interview readiness for enterprise software engineering roles.

---

## 📖 The "Senior Mindset" Paradigm Shift

As you move through this codebase, your standard for evaluating code must evolve:
*   **Junior:** *"How do I make it work?"* (Focuses entirely on satisfying a happy path, writing any syntax that executes without errors).
*   **Mid-Level:** *"Is this the right pattern?"* (Focuses on modularity, reuse, framework compliance, and structural aesthetics).
*   **Senior:** *"What does this commit us to, who pays the cost, and how do we reduce risk?"* (Focuses on tenant boundary isolation, data lifecycle, performance hotspots, database connection limits, and rollback simplicity).

---

## 🎯 Repo Identity

**pretalx** is an end-to-end, multi-tenant conference management system built with Python, Django, Celery, and Django REST Framework. Used globally by hundreds of tech conferences, it automates the full event lifecycle: Call for Papers (CFP), abstract submissions, drag-and-drop schedule editing, peer review boards, and transactional speaker email management. Architecturally, pretalx stands out due to its strict multi-tenant database isolation enforced via `django-scopes` and its modular plugin system. It features a traditional, robust server-rendered HTML template interface, coupled with an interactive Svelte visual timeline scheduler.

---

## 🗺️ The Four Learning Tracks

1.  **Codebase Cartography (Module 01):** Learn how to visually map an unfamiliar multi-tenant system, establish reading order, and trace data-flow paths from HTTP request to DB transactions.
2.  **Language Bridge (Module 02):** Leverage your JS/TS (Express, React, Prisma, Promises) mental models to grasp Python idioms, decorators, lazy ORM queries, and task runner queues.
3.  **Architecture & Patterns (Module 03):** Analyze structural isolation boundaries, custom multi-tenant middlewares, declarative rules, and database transaction fences.
4.  **Contribution Practice & Quality (Modules 04, 05, 06):** Complete hands-on drills, trace tables, review simulated PRs, and tackle graded engineering feature tickets.

---

## 🚦 Recommended Paths

### Path A: Brand-New Junior Developer
*   **Goal:** Learn how a modular MVC database-backed web application hangs together.
*   **Plan:** Start at `00-fast-track.md` to get the system running locally, proceed sequentially through `01-codebase-cartography`, complete the drills in `04-code-reading-gym`, and attempt the Easy tickets in `06-contribution-practice`.

### Path B: JS/TS Developer Learning Python
*   **Goal:** Bridge JavaScript concepts (async/await, Express router, Prisma, prototype chain) to Python/Django equivalents.
*   **Plan:** Jump directly into `02-language-bridge/` to digest comparisons. Then return to `00-fast-track.md` to run the server, and study `03-architecture-and-patterns` to master Django's ORM and request lifecycles.

### Path C: Mid-Level Engineer New to pretalx
*   **Goal:** Fast-track onboarding to start contributing high-impact features with minimal guidance.
*   **Plan:** Read `00-fast-track.md` + `01-codebase-cartography` to construct the layout map. Dive into `03-architecture-and-patterns` (specifically the Pattern Catalog), then proceed to the Medium/Hard tickets in `06-contribution-practice`.

### Path D: Senior Architect / Reviewer
*   **Goal:** Audit the project's security boundaries, scalability limits, database indexes, and architecture trade-offs.
*   **Plan:** Jump straight to `03-architecture-and-patterns/06-architecture-critique.md`, then explore `05-quality-engineering/05-security-checklist.md`. Follow up by tackling `06-contribution-practice/04-refactor-and-design-katas.md`.

### Path E: Interview Candidate
*   **Goal:** Practice language implementation differences, system design questions, live-coding sessions, and architectural behavioral narratives based on a real-world codebase.
*   **Plan:** Focus intensely on `07-interview-prep/`. Answer each question out loud, comparing your answers against the junior, mid-level, and senior answers mapped directly to pretalx files.

---

## ⏳ Curriculum Timelines

*   **The Weekend Sprint (2 Days):** Complete `00-fast-track.md` and `09-reference/js-python-rosetta-stone.md`. You will get the app running, submit your first demo proposal, and run the test suite.
*   **The Deep Dive (2 Weeks):** Read `01-codebase-cartography` and `02-language-bridge` thoroughly. Complete all interactive drills in `04-code-reading-gym`.
*   **The Mastery Program (8 Weeks):** Work through `03-architecture-and-patterns`. Select one Good First Ticket (Easy) and one Mid-Level Feature Ticket (Medium) from `06-contribution-practice`, implement them locally, write tests in pytest, and draft mock Pull Requests.
