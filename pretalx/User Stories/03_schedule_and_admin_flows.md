# Module 3: Schedule & Admin Flows

This module contains User Stories 11 to 15, targeting scheduling tools, administrative logs, and user security limits.

---

## Story 11: Render a "Time Left to Submit" Countdown Widget on CfP Page

### Product Manager Specification
*   **Actor:** Speaker (Submitter).
*   **Action:** Display a dynamic countdown timer widget (days, hours, minutes) on the landing page of the active CfP.
*   **Value:** Creates urgency and prompts speakers to submit early, decreasing overall last-minute load spike margins.
*   **Functional Requirements:**
    1.  Read `cfp.deadline` for the active event.
    2.  Calculate remaining seconds server-side, passing the timezone-aware timestamp to the frontend.
    3.  Implement a client-side JavaScript countdown timer that decreases every second.
    4.  If the deadline is reached, display *"Call for Proposals is closed."*

### Mid-Level Implementation Plan
1.  **Expose Deadline Context:**
    *   Open `src/pretalx/cfp/views/public.py` (or view handlers for CfP main page).
    *   Inject the event's deadline details:
        ```python
        context["deadline_iso"] = self.request.event.cfp.deadline.isoformat() if self.request.event.cfp.deadline else None
        ```
2.  **Add Frontend Timer Widget Markup:**
    *   Open the public CfP landing template `src/pretalx/cfp/templates/cfp/event/index.html`.
    *   Embed the container elements:
        ```html
        {% if deadline_iso %}
            <div id="cfp-countdown" data-deadline="{{ deadline_iso }}" class="alert alert-info">
                Time remaining: <span class="timer-values">--d --h --m --s</span>
            </div>
        {% endif %}
        ```
3.  **Implement JavaScript Countdown Logic:**
    *   Add a client-side JavaScript file under `src/pretalx/static/common/js/cfp-countdown.js`:
        ```javascript
        document.addEventListener("DOMContentLoaded", () => {
            const container = document.getElementById("cfp-countdown");
            if (!container) return;
            const deadline = new Date(container.dataset.deadline).getTime();
            
            const timer = setInterval(() => {
                const now = new Date().getTime();
                const diff = deadline - now;
                if (diff <= 0) {
                    container.innerHTML = "Call for Proposals is closed.";
                    clearInterval(timer);
                    return;
                }
                const days = Math.floor(diff / (1000 * 60 * 60 * 24));
                const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
                const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
                const seconds = Math.floor((diff % (1000 * 60)) / 1000);
                container.querySelector(".timer-values").innerText = `${days}d ${hours}h ${minutes}m ${seconds}s`;
            }, 1000);
        });
        ```

---

## Story 12: Enforce Double-Opt-In for New User Registrations

### Product Manager Specification
*   **Actor:** New Users (Speakers or Organizers).
*   **Action:** When a user registers a new account, save the account in a pending state, send a confirmation email with a verification link, and block system actions until verified.
*   **Value:** Prevents invalid bot signups and ensures users provide valid emails.
*   **Functional Requirements:**
    1.  Introduce an `is_email_verified` boolean field to the User model (default is `False`).
    2.  Upon signup, generate a secure token and queue verification email.
    3.  Redirect unverified users attempting to access dashboard interfaces with warning alerts.

### Mid-Level Implementation Plan
1.  **Add Database Field to Custom User Model:**
    *   Open `src/pretalx/person/models/user.py`.
    *   Add `is_email_verified = models.BooleanField(default=False)`.
2.  **Generate Migration:**
    *   Run migrations:
        ```powershell
        python src/manage.py makemigrations person
        ```
3.  **Conditional Access Middleware:**
    *   Create a middleware `EmailVerificationMiddleware` inside `src/pretalx/common/middleware/`:
        ```python
        class EmailVerificationMiddleware:
            def __init__(self, get_response):
                self.get_response = get_response
            def __call__(self, request):
                if request.user.is_authenticated and not request.user.is_email_verified:
                    # Let them visit logout or confirmation pages, block dashboard access
                    if not request.path.startswith(("/logout", "/confirm")):
                        return redirect("cfp:confirm-email-pending")
                return self.get_response(request)
        ```
4.  **Confirm Verification Handler View:**
    *   Write a verify token view that extracts user and checks token matching. If matched, sets `is_email_verified = True` and saves the user record.

---

## Story 13: Allow Reviewers to Flag a Submission for CoC (Code of Conduct) Review

### Product Manager Specification
*   **Actor:** Event Reviewer.
*   **Action:** Introduce a "Flag for Code of Conduct Review" checkbox on the reviewer scoring scorecard interface.
*   **Value:** Enables reviewers to quickly alert organizers about proposals that contain offensive content, hate speech, or plagiarized materials.
*   **Functional Requirements:**
    1.  Add `is_coc_flagged` boolean flag on the `Submission` database model.
    2.  Add a CoC warning card and checkbox to the review scoring forms.
    3.  If checked, set the flag and instantly alert track chairs.

### Mid-Level Implementation Plan
1.  **Modify Submission Model Schema:**
    *   Add a boolean flag `is_coc_flagged = models.BooleanField(default=False)` inside `Submission` models file.
2.  **Expose Checkbox inside Review Form:**
    *   Open `src/pretalx/orga/forms/review.py` (or scorecard forms classes).
    *   Append `is_coc_flagged` to form fields:
        ```python
        is_coc_flagged = forms.BooleanField(
            required=False,
            label=_("Flag for Code of Conduct Review"),
            help_text=_("Check this box if the proposal content violates safety policies.")
        )
        ```
3.  **Model Syncing on Review Save:**
    *   Inside the View processing review score submissions:
        ```python
        if form.cleaned_data.get("is_coc_flagged"):
            submission = review.submission
            submission.is_coc_flagged = True
            submission.save()
            # Send immediate alert email
            notify_orga_coc_breach(submission)
        ```

---

## Story 14: Log Administrative Actions when Deleting a Submission

### Product Manager Specification
*   **Actor:** System Administrator / Event Organizer.
*   **Action:** When a submission is deleted, write a structured log entry detailing who did it, when, and the title/authors of the deleted talk.
*   **Value:** Protects repository records against accidental or unauthorized data deletions.
*   **Functional Requirements:**
    1.  Hook into submission deletions.
    2.  Write entries to the `ActivityLog` table before database rows are deleted.

### Mid-Level Implementation Plan
1.  **Intercept Model Deletions:**
    *   Open `src/pretalx/submission/models/submission.py`.
    *   Implement an override on `.delete(*args, **kwargs)` method:
        ```python
        def delete(self, *args, **kwargs):
            # Write system log
            from pretalx.common.models.log import ActivityLog
            ActivityLog.objects.create(
                event=self.event,
                person=kwargs.get("user"), # Pass user doing deletion
                action_type="pretalx.submission.deleted",
                content=f"Deleted submission {self.code}: '{self.title}'"
            )
            super().delete(*args, **kwargs)
        ```

---

## Story 15: Support Custom File Type Validation for Submission Resources

### Product Manager Specification
*   **Actor:** Speaker (Submitter) & Event Organizer.
*   **Action:** Restrict submission resource uploads (e.g. session handouts, slide decks) to specific permitted file extensions configured in the Event settings (e.g., allow `.pdf`, `.zip`, reject `.exe`, `.dmg`).
*   **Value:** Safeguards review committees from downloading malicious executable binaries.
*   **Functional Requirements:**
    1.  Configure allowed file formats (e.g. `pdf,zip`) in Event settings.
    2.  Clean resource file uploads against lists.

### Mid-Level Implementation Plan
1.  **Form Cleaning Hook:**
    *   Locate submission resources form (or `SubmissionForm` field loaders).
    *   Introduce validator on file uploads:
        ```python
        def validate_resource_file(file_obj, allowed_extensions):
            ext = os.path.splitext(file_obj.name)[1].lower()
            if ext not in allowed_extensions:
                raise ValidationError(f"File format {ext} is not allowed.")
        ```
