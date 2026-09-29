# Module 2: Review & Scoring Flows

This module contains User Stories 6 to 10, targeting scoring policies and reviewer controls.

---

## Story 6: Hide Speaker Identities from Reviewers in "Blind Review" Phase

### Product Manager Specification
*   **Actor:** Event Reviewer.
*   **Action:** Enable a "Blind Review" checkbox in the Event Review Phase settings.
*   **Value:** Eliminates unconscious reviewer bias by hiding speaker names, biographies, and organizations while scorecards are being recorded.
*   **Functional Requirements:**
    1.  Add a boolean flag `is_blind_review` on the `ReviewPhase` model.
    2.  If an active review phase has `is_blind_review = True`, the reviewer-facing submission page must hide the speaker's name, email, biography, and profile links.
    3.  Instead of speaker names, display: *"Speaker profile hidden (Blind Review enabled)."*

### Mid-Level Implementation Plan
1.  **Add Database Field:**
    *   Open `src/pretalx/submission/models/review.py` (or search for `class ReviewPhase` using `grep_search`).
    *   Inject field:
        ```python
        is_blind_review = models.BooleanField(default=False)
        ```
2.  **Generate Migration:**
    *   Execute:
        ```powershell
        python src/manage.py makemigrations submission
        ```
3.  **Adjust Review View Serialization:**
    *   Open `src/pretalx/orga/views/review.py` or inspect review details view controllers.
    *   Check if `event.active_review_phase.is_blind_review` is enabled.
    *   If active, scrub context profiles before rendering templates:
        ```python
        context = super().get_context_data(**kwargs)
        if self.request.event.active_review_phase and self.request.event.active_review_phase.is_blind_review:
            context["speakers"] = []  # Clear speaker profiles context
        ```
4.  **Scrub HTML Template:**
    *   Open review templates in `orga/templates/orga/review/detail.html`.
    *   Add condition checks around speaker details blocks:
        ```html
        {% if not active_review_phase.is_blind_review %}
            <!-- Show speakers details -->
        {% else %}
            <p><em>Speaker profiles hidden (Blind Review enabled).</em></p>
        {% endif %}
        ```

---

## Story 7: Set a Default Submission Track for Each CfP Flow

### Product Manager Specification
*   **Actor:** Speaker (Submitter).
*   **Action:** Allow organizers to configure a "Default Track" on the Event settings. Any proposal submitted during that CfP cycle that does not explicitly pick a track is auto-assigned to this default track.
*   **Value:** Prevents submissions from getting lost or unallocated when speakers skip optional track fields.
*   **Functional Requirements:**
    1.  Add a `default_track = ForeignKey(Track)` field to the `Event` or `Cfp` configuration tables.
    2.  If the submitter does not select a track (or the field is hidden), default it to the designated track.

### Mid-Level Implementation Plan
1.  **Add Model Foreign Key:**
    *   Open `src/pretalx/submission/models/cfp.py`.
    *   Add:
        ```python
        default_track = models.ForeignKey(
            "submission.Track", null=True, blank=True, on_delete=models.SET_NULL, related_name="+"
        )
        ```
2.  **Model Validation and Fallback:**
    *   Open `src/pretalx/submission/domain/submission.py` and modify `create_submission`.
    *   In the save routine:
        ```python
        if not submission.track and submission.event.cfp.default_track:
            submission.track = submission.event.cfp.default_track
        ```
3.  **Run migrations:**
    *   Run makemigrations:
        ```powershell
        python src/manage.py makemigrations submission
        ```

---

## Story 8: Custom Email Notification when a New Submission matches Specific Tags

### Product Manager Specification
*   **Actor:** Event Organizer (Track Chair).
*   **Action:** Set up alert rules (e.g. tag "Security" triggers email alerts to `security-team@conference.org`).
*   **Value:** Instantly notifies specialized review committees as soon as a relevant proposal is received, speeding up evaluation pipelines.
*   **Functional Requirements:**
    1.  Add an alert configuration list model mapping a Tag to target notification email addresses.
    2.  When a proposal is moved to `SUBMITTED` state, query tags and send alerts dynamically.

### Mid-Level Implementation Plan
1.  **Define Alert Table Model:**
    *   Create a model `TagAlertRule` inside `src/pretalx/submission/models/tag.py`:
        ```python
        class TagAlertRule(models.Model):
            event = models.ForeignKey("event.Event", on_delete=models.CASCADE)
            tag = models.ForeignKey("submission.Tag", on_delete=models.CASCADE)
            email = models.EmailField()
        ```
2.  **Add Post-Save Signal Trigger:**
    *   Open `src/pretalx/submission/domain/submission.py`.
    *   Inside the state transition `set_submission_state` (or post-commit task dispatcher):
        ```python
        def trigger_tag_alerts(submission):
            matching_rules = TagAlertRule.objects.filter(tag__in=submission.tags.all())
            for rule in matching_rules:
                # Dispatch background mail task
                send_mail_task.delay(
                    to=[rule.email],
                    subject=f"New Tag Match: {submission.title}",
                    body=f"A new talk matching {rule.tag.name} has been submitted."
                )
        ```
3.  **Run Migrations:**
    *   Register and execute model migration files.

---

## Story 9: Bulk Update Talk Slot Durations from the Orga Panel

### Product Manager Specification
*   **Actor:** Event Organizer.
*   **Action:** Select multiple submissions in the dashboard table and apply a bulk duration update (e.g., set all selected talks to 45 minutes).
*   **Value:** Reduces administrative friction when re-structuring room schedules or session durations.
*   **Functional Requirements:**
    1.  Add checkbox selections to the Orga talks table list.
    2.  Add a dropdown batch action: *"Set Duration to..."* (input integer minutes).
    3.  Reconcile slot allocations for all updated records.

### Mid-Level Implementation Plan
1.  **Expose Selection Forms in UI:**
    *   Open `src/pretalx/orga/templates/orga/submission/list.html` and wrap list items in standard checkbox HTML inputs `<input type="checkbox" name="submission_code" value="{{ sub.code }}">`.
2.  **Controller Action Handler:**
    *   Modify `/orga/event/{event}/submissions/` POST handler:
        ```python
        class SubmissionBatchActionView(PermissionRequired, View):
            def post(self, request, *args, **kwargs):
                codes = request.POST.getlist("submission_code")
                new_duration = int(request.POST.get("batch_duration"))
                
                with transaction.atomic():
                    for code in codes:
                        sub = Submission.objects.get(code=code, event=request.event)
                        sub.duration = new_duration
                        sub.save()
                        update_talk_slots(sub)
                return redirect("orga:submissions.list")
        ```

---

## Story 10: Enforce "One Proposal Limit per Submitter" per Event

### Product Manager Specification
*   **Actor:** Speaker (Submitter).
*   **Action:** Restrict speakers from submitting more than one talk per event.
*   **Value:** Enforces fairness in highly competitive Call for Proposals and reduces reviewer fatigue.
*   **Functional Requirements:**
    1.  Add an option `one_proposal_per_user` boolean checkbox to the event CfP settings.
    2.  When a speaker attempts to submit a second proposal, block access and render an alert message.

### Mid-Level Implementation Plan
1.  **Add Validation check in CfP Wizard:**
    *   Open `src/pretalx/cfp/views/wizard.py` and modify the first step display or step verification methods.
    *   In the step display override hook:
        ```python
        if self.request.event.cfp.settings.get("one_proposal_per_user"):
            existing_count = Submission.objects.filter(
                event=self.request.event, speakers__in=[self.request.user]
            ).count()
            if existing_count >= 1:
                raise ValidationError(_("You are allowed to submit only one proposal for this event."))
        ```
