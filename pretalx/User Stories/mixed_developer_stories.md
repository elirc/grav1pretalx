# Mixed Developer Stories: 25 User Stories & High-Level Plans

This file contains 25 user stories designed for both junior and mid-level developers. Each story features a clear stakeholder requirement followed by a high-level, conceptual implementation plan that you can easily break down into specific subtasks.

---

## Story 1: Add a track filter to the public schedule page

*   **Target Level:** Junior
*   **User Story:** As a public attendee browsing the conference agenda, I want to filter the schedule by track (e.g., only show "Security" talks) so that I can focus on my specific interests.
*   **High-Level Implementation Plan:**
    1.  Fetch all active event tracks inside the public schedule view context loader.
    2.  Render a track filter dropdown in the public schedule HTML template.
    3.  Use JavaScript to show or hide talk slot container elements on the client side when a track selection is toggled.

---

## Story 2: Display speaker email address in talk details tooltips for organizers

*   **Target Level:** Junior
*   **User Story:** As an organizer reviewing talks in the dashboard list, I want to quickly see the speaker's email address when hovering over their name so that I can coordinate external communications faster.
*   **High-Level Implementation Plan:**
    1.  Expose the speaker email address property in the template context list for organizers.
    2.  Add a `title` or tooltip attribute to the speaker link elements in the dashboard table markup.

---

## Story 3: Set default review phase start date to today's date

*   **Target Level:** Junior
*   **User Story:** As an organizer configuring review phases, I want the start date input to default to today's date so that I don't have to manually select it when creating a new phase.
*   **High-Level Implementation Plan:**
    1.  Locate the review phase configuration form.
    2.  Set the initial value of the `start` date field to `timezone.now().date()` in the form's `__init__` or field definition.

---

## Story 4: Allow organizers to override talk slot colors

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer scheduling sessions, I want to assign custom background colors to specific talk slots in the visual editor so that I can visually highlight keynote slots or sponsored talks.
*   **High-Level Implementation Plan:**
    1.  Add a `color` field (hex code string) to the `TalkSlot` database model and run a schema migration.
    2.  Expose a color picker widget in the organizer's session slot edit form.
    3.  Pass the custom hex code values to the visual schedule editor API serializers and render them as custom CSS styles.

---

## Story 5: Auto-link social media handles in speaker profile views

*   **Target Level:** Junior
*   **User Story:** As a public site visitor browsing speaker profiles, I want to click on listed Twitter, Mastodon, or GitHub handles to visit the speaker's external profile page.
*   **High-Level Implementation Plan:**
    1.  Verify speaker profile models support fields for social media handles.
    2.  Create a custom template filter helper to parse plain handles (e.g. `@twitteruser`) into fully qualified hyperlinks.
    3.  Apply this template filter to social handles rendered in public profile HTML pages.

---

## Story 6: Hide canceled talks from the speaker directory listings

*   **Target Level:** Junior
*   **User Story:** As a public attendee searching speakers, I don't want to see speakers whose talks have been canceled so that I only see profiles of active presenters.
*   **High-Level Implementation Plan:**
    1.  Locate the public speaker directory view query loader.
    2.  Update the database filters to exclude speakers associated only with submissions in the `CANCELED` state.

---

## Story 7: Allow organizers to clone an existing track

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer configuring event tracks, I want a "Clone Track" button to copy track attributes (color, review criteria overrides, permissions) to a new track entry quickly.
*   **High-Level Implementation Plan:**
    1.  Register a new `/orga/track/{id}/clone/` view endpoint in URLs.
    2.  In the clone view controller, retrieve the target track, clear its primary key (`pk = None`), append a `" (Copy)"` suffix to the name, and save the new cloned instance.
    3.  Redirect back to the track list with a success message.

---

## Story 8: Enforce maximum file size check on submission resource attachments

*   **Target Level:** Junior
*   **User Story:** As an organizer, I want to restrict session resource attachments to a maximum of 10MB so that speakers do not exhaust our local server storage.
*   **High-Level Implementation Plan:**
    1.  Locate the resource attachment upload form class.
    2.  Implement a form clean method on the file field that checks `file.size`.
    3.  Raise a standard `ValidationError` if the upload size exceeds 10MB.

---

## Story 9: Validate event slug uniqueness in custom settings views

*   **Target Level:** Junior
*   **User Story:** As a system administrator, I want to ensure no two events are created with the exact same slug (even with trailing whitespace) so that public routing matches distinct event profiles.
*   **High-Level Implementation Plan:**
    1.  Locate the event creation form class.
    2.  Add a `clean_slug` method that strips whitespace and runs a QuerySet lookup checking if the slug exists on another event record.
    3.  Throw a validation error on match.

---

## Story 10: Support markdown rendering inside track descriptions

*   **Target Level:** Junior
*   **User Story:** As an organizer, I want to write markdown (bullets, bold text) in track descriptions and have it render formatted on public pages.
*   **High-Level Implementation Plan:**
    1.  Locate where the track description is rendered on public templates.
    2.  Wrap the template output variable in the default `rich_text` or `markdown` template tags.

---

## Story 11: Implement a "Mark all emails as read" action in organizer mail panel

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer managing notifications, I want a single button to mark all inbox email alerts as read so that I can quickly clear my dashboard alerts.
*   **High-Level Implementation Plan:**
    1.  Create a POST controller view in orga mail viewpoints.
    2.  Run a bulk database transaction updating all unread mail rows for that event to an "archived" or "read" state.
    3.  Inject a "Mark all as read" button on the UI sidebar.

---

## Story 12: Expose talk slot room names in the API schedule endpoints

*   **Target Level:** Junior
*   **User Story:** As a third-party developer building a conference mobile app, I want room details exposed inside the REST API `/schedule/` endpoint so that I can display session locations.
*   **High-Level Implementation Plan:**
    1.  Locate schedule serializers inside the API views module.
    2.  Expose the `room` representation field in the serializer meta-definitions, fetching the room name and capacity attributes.

---

## Story 13: Allow reviewers to toggle a "Show only reviewed talks" filter on dashboards

*   **Target Level:** Mid-Level
*   **User Story:** As a reviewer grading 100 talks, I want to filter the dashboard to display only the talks I have already graded so that I can quickly double-check my scores.
*   **High-Level Implementation Plan:**
    1.  Add a checkbox filter filter `"Show Reviewed"` to the review list form inputs.
    2.  In the view query filter class, check if the request query parameter is active and append a filter matching reviews written by the active user.

---

## Story 14: Add a simple activity log entry when an event name is changed

*   **Target Level:** Junior
*   **User Story:** As an administrator auditing changes, I want to see an Activity Log entry whenever an event's public name is edited in the settings.
*   **High-Level Implementation Plan:**
    1.  Locate the event settings view saving routine.
    2.  Call the event's log action helper passing `"pretalx.event.updated"` with the old and new event names.

---

## Story 15: Hide co-speaker invitation code inputs once redeemed

*   **Target Level:** Junior
*   **User Story:** As a speaker redeeming an invite, I don't want to see the raw input code text box once the system has processed my token, so that I know the invitation was successfully redeemed.
*   **High-Level Implementation Plan:**
    1.  Locate the template rendering speaker invitation status.
    2.  Wrap the input form container elements in a conditional check: `{% if not invitation.is_redeemed %}`.

---

## Story 16: Bulk download all review scorecards for a submission as a combined text document

*   **Target Level:** Mid-Level
*   **User Story:** As a track chair preparing for deliberation, I want to download all reviewer scores and comments for a single talk as a single plain text file so that I can read them offline.
*   **High-Level Implementation Plan:**
    1.  Create an download view controller matching `/orga/submission/{code}/reviews/download/`.
    2.  Read all non-draft reviews for the talk, write scores and text reviews to a string buffer, and return the buffer as an attachment response with `content_type="text/plain"`.

---

## Story 17: Allow custom labeling for submission types

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer, I want to rename our submission formats (e.g., label 'Talk' as 'Keynote' or 'Workshop' as 'Lab') so that the submission wizards match our branding.
*   **High-Level Implementation Plan:**
    1.  Add a `display_label` custom text field to the `SubmissionType` model.
    2.  If this custom label is present, display it in public CfP wizards instead of the standard database name.

---

## Story 18: Add custom warning alerts when track capacities are exceeded

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer, I want the system to warn me if I allocate more talk slots to a track than its configured maximum capacity allows.
*   **High-Level Implementation Plan:**
    1.  Introduce a `max_capacity` integer field on the `Track` model.
    2.  When a scheduler attempts to save or slot a new talk inside the visual editor, count all active slots for the track and check against `max_capacity`, returning a warning block if exceeded.

---

## Story 19: Append an organizer notes field on speaker profile pages

*   **Target Level:** Junior
*   **User Story:** As an organizer reviewing speakers, I want to write internal private notes on speaker profiles (e.g. "needs AV help") that are only visible to other organizers.
*   **High-Level Implementation Plan:**
    1.  Add a `notes` TextField to the speaker profile model and execute database migrations.
    2.  Render the field as an editable form input inside the organizer speaker detail panel.

---

## Story 20: Display total review scorecard averages in the review dashboard list

*   **Target Level:** Junior
*   **User Story:** As an organizer, I want to see the average score of each submission in the master review dashboard table so that I can sort submissions by performance.
*   **High-Level Implementation Plan:**
    1.  Locate the review list view query loader.
    2.  Annotate the QuerySet with the average score of all active reviews.
    3.  Display the annotated value as a new column in the dashboard table template.

---

## Story 21: Auto-generate confirmation emails when talk is moved to 'confirmed'

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer, I want the system to automatically queue a "Welcome & logistics" email draft to speakers as soon as their session state transitions to 'confirmed'.
*   **High-Level Implementation Plan:**
    1.  Hook into state transitions inside domain services.
    2.  When the state changes to `CONFIRMED`, fetch the active event's confirmation email template, render variables, and save a new `QueuedMail` row.

---

## Story 22: Order tracks in public schedules by the order set in organizer settings

*   **Target Level:** Junior
*   **User Story:** As an attendee, I want tracks in the public schedule to display in the custom sequential order configured by the event team rather than alphabetically.
*   **High-Level Implementation Plan:**
    1.  Ensure the Track model supports an integer field `position`.
    2.  Update public schedule context queries to order track listings by `position` instead of alphabetically.

---

## Story 23: Allow custom help text overrides for track selection fields

*   **Target Level:** Mid-Level
*   **User Story:** As an organizer, I want to write custom help text instructions specifically for the track selection input field in the speaker submission forms.
*   **High-Level Implementation Plan:**
    1.  Add a `track_field_help_text` field to the Event settings configuration parameters.
    2.  In the speaker submission form class, read this setting and override `self.fields["track"].help_text` dynamically.

---

## Story 24: Validate email domains to block anonymous email providers from speaker signups

*   **Target Level:** Mid-Level
*   **User Story:** As a system administrator, I want to prevent speakers from registering accounts using temporary or disposable email addresses so that we have stable communication lines.
*   **High-Level Implementation Plan:**
    1.  Write a form validation helper checking if the input email domain matches a blocklist of disposable email domains (e.g. mailinator.com).
    2.  Attach this validator class to standard signup and registration forms.

---

## Story 25: Expose talk slot start and end dates inside the speaker proposal view

*   **Target Level:** Junior
*   **User Story:** As a speaker, I want to see the finalized date, start time, and room name of my talk inside my speaker dashboard once the schedule is published.
*   **High-Level Implementation Plan:**
    1.  Locate speaker dashboard templates and views.
    2.  Fetch published scheduled slots for the active submission.
    3.  If a published slot exists, display its room name and time interval in the template.
