# pretalx Pure JS/TS Rewrite Plan
### Architecture, Subtasks, and User Stories for Next-Gen Event Management

This folder contains a complete, production-grade rewrite and migration blueprint to transition the **pretalx** conference organization platform from its native Python/Django architecture to a pure JavaScript, TypeScript, and React full-stack ecosystem.

---

## 🚀 The Target Tech Stack

*   **Frontend Web App:** Next.js (App Router) + React 19 + Tailwind CSS + Radix UI + Lucide icons.
*   **Interactive Visual Editor:** React Flow / Svelte 5 (with Tailwind) + Svelte-to-React adapters.
*   **Backend Server/API:** NestJS (TypeScript) or Next.js API Routes (Serverless-optimized).
*   **Database ORM:** Prisma ORM / Drizzle ORM connecting to **PostgreSQL**.
*   **Multi-Tenant Scoping Isolation:** TypeScript custom middlewares utilizing Node.js **`AsyncLocalStorage`** to enforce global tenant query filters.
*   **Background Jobs & Task Workers:** **BullMQ** running on **Redis** (replacing Celery and Redis/RabbitMQ).
*   **Email Outbox Renders:** **React Email** + Resend/Nodemailer (replacing Django templated mailers).

---

## 📂 Plan Structure & Specification Modules

### [**`01_system_architecture.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/pretalx_js_rewrite_plan/01_system_architecture.md)
*   Detailed overall system architecture, entity-relationship diagrams, API contract specifications, background worker structures, and multi-tenant scoping mechanisms using Node.js context storage.

### [**`02_subtasks_and_migration.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/pretalx_js_rewrite_plan/02_subtasks_and_migration.md)
*   Exhaustive sprint roadmap containing chronological backend, frontend, database, and operational subtasks. Includes a detailed SQLite/PostgreSQL-to-Prisma schema data migration plan.

### [**`03_user_stories.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/pretalx_js_rewrite_plan/03_user_stories.md)
*   High-level user stories representing functional requirements for the Call for Proposals (CfP) Wizard, Organizer Panel, Blind Review Scorecard, Visual Schedule Editor, and Transactional Mail Outbox.
