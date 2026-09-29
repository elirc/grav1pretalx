# User Stories & Detailed Implementation Plans

This folder contains 25 detailed specifications and user stories written from the perspective of a Product Manager / Stakeholder to a Mid-Level Software Engineer. 

For each story, a hyper-detailed, step-by-step technical implementation plan is provided, complete with model additions, schema migrations, view overrides, form validations, and unit tests, making it trivial for a mid-level engineer to implement.

---

## 🗺️ Story Index

### 📣 Module 1: Speaker & CfP Flows
*   [**`01_speaker_and_cfp_flows.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/01_speaker_and_cfp_flows.md)
    *   **Story 1:** Restrict Speaker Abstract Length by Word Count (Instead of Character Count).
    *   **Story 2:** Auto-expire Pending Speaker Invitations after X Days.
    *   **Story 3:** Export Review Scores as CSV for Custom Excel Analysis.
    *   **Story 4:** Allow Speakers to Request Session Room Layout Changes.
    *   **Story 5:** Enforce Password Complexity Validator for Organizer Accounts.

### ⚖️ Module 2: Review & Scoring Flows
*   [**`02_review_and_scoring_flows.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/02_review_and_scoring_flows.md)
    *   **Story 6:** Hide Speaker Identities from Reviewers in "Blind Review" Phase.
    *   **Story 7:** Set a Default Submission Track for Each CfP Flow.
    *   **Story 8:** Custom Email Notification when a New Submission matches Specific Tags.
    *   **Story 9:** Bulk Update Talk Slot Durations from the Orga Panel.
    *   **Story 10:** Enforce "One Proposal Limit per Submitter" per Event.

### 📅 Module 3: Schedule & Admin Flows
*   [**`03_schedule_and_admin_flows.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/03_schedule_and_admin_flows.md)
    *   **Story 11:** Render a "Time Left to Submit" Countdown Widget on CfP Page.
    *   **Story 12:** Enforce Double-Opt-In for New User Registrations.
    *   **Story 13:** Allow Reviewers to Flag a Submission for CoC (Code of Conduct) Review.
    *   **Story 14:** Log Administrative Actions when Deleting a Submission.
    *   **Story 15:** Support Custom File Type Validation for Submission Resources.

### 📬 Module 4: Notifications & API Flows
*   [**`04_notifications_and_api_flows.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/04_notifications_and_api_flows.md)
    *   **Story 16:** Configure a Maximum Number of Reviewers per Submission.
    *   **Story 17:** Support "Tentative" Status for Talk Slot Scheduling.
    *   **Story 18:** Auto-generate speaker badges as PDF downloads.
    *   **Story 19:** Filter Submissions by Tag in the REST API Endpoint.
    *   **Story 20:** Display a "Warning: Schedule Clash" Alert in visual editor.

### 🔒 Module 5: Security & Operations
*   [**`05_security_and_operations.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/05_security_and_operations.md)
    *   **Story 21:** Support custom redirection URLs after CfP completion.
    *   **Story 22:** Allow organizers to bulk-requeue failed email drafts.
    *   **Story 23:** Allow reviewers to save drafts of reviews before submitting.
    *   **Story 24:** Enforce rate limiting on the CfP submission endpoint.
    *   **Story 25:** Display a public speaker directory with biographies search.
