# Junior Developer Onboarding: 42 Introductory Stories

Welcome to the pretalx Junior Onboarding series! This module contains 42 simple, introductory user stories specifically designed to help new developers get familiar with the codebase. 

Each story represents a low-risk, high-learning-value task (such as UI text adjustments, basic form validations, simple model properties, and email template updates).

---

## 🚦 Junior Onboarding Map

### [**`junior_onboarding_01_syntax_and_ui.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/junior_onboarding_01_syntax_and_ui.md)
*   **Stories 1–10: UI, Syntax & Static Assets**
    *   **Story 1:** Add a helper link in the CfP landing page to the Code of Conduct.
    *   **Story 2:** Change default text for speaker invitations to be warmer.
    *   **Story 3:** Adjust the padding of the review score display badge.
    *   **Story 4:** Fix a typo in the organizer settings help text.
    *   **Story 5:** Show a visual "Required" asterisk next to required CfP fields.
    *   **Story 6:** Update the copyright year footer to dynamically pull the current year.
    *   **Story 7:** Expose speaker biography length constraints in the CfP profile form help text.
    *   **Story 8:** Render the conference logo on the public speaker biography page.
    *   **Story 9:** Customize the placeholder in the submission title text field.
    *   **Story 10:** Add a CSS style to highlight withdrawn talk submissions.

### [**`junior_onboarding_02_views_and_forms.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/junior_onboarding_02_views_and_forms.md)
*   **Stories 11–20: Basic Views & Forms**
    *   **Story 11:** Enforce a minimum character limit on the proposal title.
    *   **Story 12:** Block reviewers from entering negative custom review scores.
    *   **Story 13:** Expose an option to sort speaker listings alphabetically.
    *   **Story 14:** Hide the review tab if no active review phase is set.
    *   **Story 15:** Enforce a checkbox check for speaker terms of service.
    *   **Story 16:** Add a "Reset Filter" button on the organizer submissions list.
    *   **Story 17:** Add a placeholder to the track description text area.
    *   **Story 18:** Log warning message when an organizer updates the CfP dates.
    *   **Story 19:** Add simple character length counter display for the talk abstract.
    *   **Story 20:** Enforce lowercase-only formatting on new event slugs.

### [**`junior_onboarding_03_models_and_queries.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/junior_onboarding_03_models_and_queries.md)
*   **Stories 21–30: Models & Simple Queries**
    *   **Story 21:** Add a property `is_accepted` to the `Submission` model.
    *   **Story 22:** Order tracks alphabetically by default in database queries.
    *   **Story 23:** Add a helper method to check if a user is an organizer.
    *   **Story 24:** Expose the number of speakers on the `Submission` model.
    *   **Story 25:** Add a simple boolean field `is_featured` to the `Submission` model.
    *   **Story 26:** Expose a property `is_cfp_active` on the `Event` model.
    *   **Story 27:** Get the count of total reviews for a submission.
    *   **Story 28:** Add a property `is_confirmed` to the `Submission` model.
    *   **Story 29:** Display track names next to talk slots in schedules.
    *   **Story 30:** Order rooms in visual schedules by capacity.

### [**`junior_onboarding_04_mail_and_alerts.md`**](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/User%20Stories/junior_onboarding_04_mail_and_alerts.md)
*   **Stories 31–42: Mail, Alerts & Operations**
    *   **Story 31:** Update default email templates with a signature line.
    *   **Story 32:** Log an activity entry when a new track is created.
    *   **Story 33:** Send alert warning when an event start date is set in the past.
    *   **Story 34:** Log warning when a mail fails delivery due to missing email address.
    *   **Story 35:** Add simple fallback subject line when a mail subject is blank.
    *   **Story 36:** Display warning if a room's capacity is set to zero.
    *   **Story 37:** Add activity log entry when a talk is withdrawn.
    *   **Story 38:** Hide rooms without scheduled talks from the public agenda page.
    *   **Story 39:** Add simple success banner message when a profile is saved.
    *   **Story 40:** Log action details when an invite is deleted.
    *   **Story 41:** Show error alert when speaker biography is empty.
    *   **Story 42:** Generate warning alert when a review phase ends before it starts.
