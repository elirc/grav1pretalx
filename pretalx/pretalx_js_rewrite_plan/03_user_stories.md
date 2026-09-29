# JS/TS Rewrite: Functional User Stories & High-Level Requirements

This file details high-level user stories representing the core functional requirements for the TypeScript, React, and Next.js full-stack rewrite of **pretalx**.

---

## 📣 Module 1: The Call for Proposals (CfP) Wizard

### User Story 1.1: Multi-Step Proposal Submission
*   **Story:** As a speaker, I want to submit a talk proposal using a clear, multi-step wizard (Proposal Details, Speaker Profile, Additional Speakers, Completion) so that my progress is saved draft-by-draft.
*   **High-Level Requirements:**
    *   Implement client-side multi-step router state in Next.js.
    *   Expose draft persistence APIs that temporarily save active steps inside Redis.
    *   Enforce field visibility (Required, Optional, Hidden) based on active event configuration tables.

### User Story 1.2: Dynamic Biography Character Length Counter
*   **Story:** As a speaker filling out my profile, I want to see a live visual countdown of characters remaining in my biography so that I don't exceed the configured limit.
*   **High-Level Requirements:**
    *   A React client widget attached to the biography TextArea input.
    *   Read limits dynamically from event settings and check `value.length` on every keyboard input.

---

## 📅 Module 2: The Visual Schedule Planner & Agenda

### User Story 2.1: Drag-and-Drop Session Timetable Allocation
*   **Story:** As an event organizer, I want to assign talk slots to rooms and times using a visual drag-and-drop calendar interface so that I can schedule sessions without typing times.
*   **High-Level Requirements:**
    *   A Next.js calendar grid UI built with React Flow or Svelte-in-React containers.
    *   Detect slot overlaps and display a warning banner if the same speaker is scheduled to two concurrent sessions.
    *   Sync slot movements dynamically with the backend API via atomic database transactions.

### User Story 2.2: Edge-Cached Public Event Agenda
*   **Story:** As a conference attendee, I want the schedule timetable pages to load instantly on my mobile browser even under heavy conference Wi-Fi loads.
*   **High-Level Requirements:**
    *   Server-render schedule views using Next.js Server Components (RSC) to bypass client-side data fetching latency.
    *   Enable CDN edge-caching on public schedule routes with cache revalidation triggers when schedule versions are published.

---

## ⚖️ Module 3: Blind Peer Reviews & Scorecards

### User Story 3.1: Anonymized Submissions for Unbiased Grading
*   **Story:** As a reviewer grading submissions during the "Blind Review" phase, I want to read talk descriptions without seeing speaker names, emails, or biographies so that my scoring is unbiased.
*   **High-Level Requirements:**
    *   If active `ReviewPhase` settings has `isBlindReview = true`, omit speaker details from API response data payloads.
    *   Render a warning placeholder in the review interface: *"Speaker identity hidden (Blind Review active)."*

### User Story 3.2: Custom Review Scorecard Creation
*   **Story:** As an organizer, I want to configure customized scorecards (e.g. scoring criteria like "Technical Depth" from 0 to 5) so that reviewers can grade submissions on specific dimensions.
*   **High-Level Requirements:**
    *   Expose forms in Next.js admin dashboard to create and edit score categories with custom integer weights.
    *   Sum and multiply categories dynamically to compute aggregate review averages.

---

## 📬 Module 4: Transactional Mail Outbox

### User Story 4.1: Transactional Speaker Acceptance Alerts
*   **Story:** As an organizer accepting or rejecting proposals, I want the system to prepare mail drafts synchronously with the database transition, so that notifications are queued cleanly without blocking web request execution times.
*   **High-Level Requirements:**
    *   Save email metadata in `QueuedMail` database rows within the same Prisma transaction that updates the talk state.
    *   Enqueue background dispatch actions inside BullMQ.
    *   Expose a "Mail Outbox" dashboard for organizers to inspect, edit, or bulk-requeue mail drafts before they are sent.
