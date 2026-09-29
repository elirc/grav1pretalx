# Onboarding Stories 31–42: Mail, Alerts & Operations

This module contains introductory stories on email outboxes, logging mechanisms, warning alerts, and basic validation helpers.

---

## Story 31: Update default email templates with a signature line

### Stakeholder Request
*"We want all system emails to look consistent. Please append a default signature line 'Best regards, The Event Organizers' to our default mail templates."*

### Junior Walkthrough & Explanation
Email templates are stored in **Models** or **settings** as raw text. We need to locate where default templates are instantiated and update the base template text.

### Detailed Implementation Steps
1.  **Locate Email Model Helpers:** Open `src/pretalx/mail/models.py`.
2.  **Edit Default Template Body:**
    *   Find the string used for initial email templates and append:
        ```text
        \n\nBest regards,\nThe Event Organizers
        ```

---

## Story 32: Log an activity entry when a new track is created

### Stakeholder Request
*"To help keep track of configuration changes, please add an Activity Log entry whenever an organizer adds a new Track to the event."*

### Junior Walkthrough & Explanation
pretalx maintains a database-backed activity log. We can record custom events by calling the `log_action` helper on the event model.

### Detailed Implementation Steps
1.  **Locate Track Creation View:** Open `src/pretalx/orga/views/track.py`.
2.  **Call Log Action on Form Valid:**
    ```python
    def form_valid(self, form):
        response = super().form_valid(form)
        self.request.event.log_action(
            "pretalx.track.created",
            person=self.request.user,
            data={"name": form.instance.name}
        )
        return response
    ```

---

## Story 33: Send alert warning when an event start date is set in the past

### Stakeholder Request
*"Occasionally, organizers accidentally select historical years when scheduling events. Please display a warning alert in the settings view if the start date is set in the past."*

### Junior Walkthrough & Explanation
We can compare dates and raise form validation warning alerts inside Event saving view forms.

### Detailed Implementation Steps
1.  **Locate Event Form:** Open `src/pretalx/orga/forms/event.py`.
2.  **Add Check on clean method:**
    ```python
    def clean_date_from(self):
        date_from = self.cleaned_data.get("date_from")
        if date_from and date_from < timezone.now().date():
            # Standard warning logs or non-blocking validation warning
            pass
        return date_from
    ```

---

## Story 34: Log warning when a mail fails delivery due to missing email address

### Stakeholder Request
*"If a queued email has an empty recipient address, we want the system to write a warning log entry rather than crashing with an SMTP error."*

### Junior Walkthrough & Explanation
We wrap potential exceptions in a `try/except` block and write error logs.

### Detailed Implementation Steps
1.  **Locate Mail Delivery Task:** Open `src/pretalx/mail/tasks.py`.
2.  **Inject Try-Except Block:**
    ```python
    # Inside the mail dispatch loop
    if not mail.to:
        logger.warning(f"Skipping delivery of mail {mail.pk}: No recipient address set.")
        mail.state = "failed"
        mail.save()
        return
    ```

---

## Story 35: Add simple fallback subject line when a mail subject is blank

### Stakeholder Request
*"Sometimes bulk drafts are created without a subject. Let's fallback to a generic subject '[Notification] Event Update' instead of sending emails with an empty subject."*

### Junior Walkthrough & Explanation
We use fallback strings inside mail models properties before rendering or saving.

### Detailed Implementation Steps
1.  **Locate Mail Model:** Open `src/pretalx/mail/models.py`.
2.  **Add Fallback logic inside subject getter:**
    ```python
    @property
    def subject_display(self):
        return self.subject or "[Notification] Event Update"
    ```

---

## Story 36: Display warning if a room's capacity is set to zero

### Stakeholder Request
*"If organizers input a room capacity of zero, it breaks schedule calculations. Please display a warning message on the room edit form."*

### Junior Walkthrough & Explanation
We add validation error warnings inside Form classes clean methods.

### Detailed Implementation Steps
1.  **Locate Room Form:** Open `src/pretalx/orga/forms/room.py`.
2.  **Clean Capacity field:**
    ```python
    def clean_capacity(self):
        capacity = self.cleaned_data.get("capacity")
        if capacity is not None and capacity <= 0:
            raise forms.ValidationError(_("Room capacity must be greater than zero."))
        return capacity
    ```

---

## Story 37: Add activity log entry when a talk is withdrawn

### Stakeholder Request
*"When speakers withdraw their proposals, organizers need to know. Please write an Activity Log entry when a proposal's state is changed to 'withdrawn'."*

### Junior Walkthrough & Explanation
We can hook into state transitions inside domain services.

### Detailed Implementation Steps
1.  **Locate State Service:** Open `src/pretalx/submission/domain/submission.py`.
2.  **Inject Log Call inside state setter:**
    *   Inside the transition matching `SubmissionStates.WITHDRAWN`:
        ```python
        submission.event.log_action(
            "pretalx.submission.withdrawn",
            person=user,
            data={"submission": submission.code}
        )
        ```

---

## Story 38: Hide rooms without scheduled talks from the public agenda page

### Stakeholder Request
*"Empty rooms on the schedule agenda page look confusing. Please hide any rooms that have zero talks scheduled on the public timetable view."*

### Junior Walkthrough & Explanation
We can filter room lists inside agenda views before passing them to the rendering templates.

### Detailed Implementation Steps
1.  **Locate Schedule View:** Open `src/pretalx/agenda/views/schedule.py`.
2.  **Filter Room QuerySet:**
    ```python
    # Filter rooms that have talk slot associations
    rooms = rooms.filter(talks__isnull=False).distinct()
    ```

---

## Story 39: Add simple success banner message when a profile is saved

### Stakeholder Request
*"Speakers often save their bio edit forms and wonder if it succeeded. Please show a green success notification badge when speaker details are updated."*

### Junior Walkthrough & Explanation
We invoke the messages helper `messages.success()` inside the form success handler.

### Detailed Implementation Steps
1.  **Locate Profile View:** Open `src/pretalx/cfp/views/profile.py`.
2.  **Add Success Message:**
    ```python
    def form_valid(self, form):
        response = super().form_valid(form)
        messages.success(self.request, _("Your profile has been saved successfully!"))
        return response
    ```

---

## Story 40: Log action details when an invite is deleted

### Stakeholder Request
*"If co-speaker invites are cancelled, we need a record. Write a log entry when an invitation token is deleted."*

### Junior Walkthrough & Explanation
We add logging inside the view processing invitation cancellations.

### Detailed Implementation Steps
1.  **Locate Cancel View:** Open `src/pretalx/cfp/views/invitation.py`.
2.  **Add Log Action:**
    ```python
    self.request.event.log_action(
        "pretalx.invite.deleted",
        person=self.request.user,
        data={"email": invite.email}
    )
    ```

---

## Story 41: Show error alert when speaker biography is empty

### Stakeholder Request
*"We need complete biographies to print badges. If a speaker leaves their biography blank on details pages, display a warning card."*

### Junior Walkthrough & Explanation
We write template level conditional checks.

### Detailed Implementation Steps
1.  **Locate Template:** Open `src/pretalx/orga/templates/orga/speaker/detail.html`.
2.  **Embed warning card:**
    ```html
    {% if not speaker.biography %}
        <div class="alert alert-danger">
            <strong>Warning:</strong> Biography is empty. Please complete it to print badges.
        </div>
    {% endif %}
    ```

---

## Story 42: Generate warning alert when a review phase ends before it starts

### Stakeholder Request
*"Organizers sometimes switch start and end dates on review phases by mistake. Validate that the end date is always after the start date."*

### Junior Walkthrough & Explanation
We write dual-field validations inside our Form class clean hook.

### Detailed Implementation Steps
1.  **Locate Review Form:** Open `src/pretalx/orga/forms/review.py`.
2.  **Validate dates inside clean():**
    ```python
    def clean(self):
        cleaned_data = super().clean()
        start = cleaned_data.get("start")
        end = cleaned_data.get("end")
        if start and end and end < start:
            raise forms.ValidationError(_("Review phase end date cannot be before its start date."))
        return cleaned_data
    ```
