# Module 5: Security & Operations

This module contains User Stories 21 to 25, targeting custom redirects, bulk actions, drafts, rate limiting, and directories.

---

## Story 21: Support custom redirection URLs after CfP completion

### Product Manager Specification
*   **Actor:** Event Organizer & Speakers.
*   **Action:** Add a settings field `custom_redirect_url` in the CfP settings page. When a speaker successfully finishes their proposal wizard submission, redirect them to this custom URL (e.g. a logistics survey or hotel booking page) instead of the standard success dashboard.
*   **Value:** Allows organizers to guide speakers toward booking accommodations or filling out demographic surveys immediately upon proposal finalization.
*   **Functional Requirements:**
    1.  Expose `custom_redirect_url` text input (URL validation enforced) on the CfP settings panel.
    2.  Read and redirect to this URL on successful submit completion.

### Mid-Level Implementation Plan
1.  **Expose Settings Input Form:**
    *   Open `src/pretalx/orga/forms/cfp.py` and register field:
        ```python
        custom_redirect_url = forms.URLField(
            required=False,
            label=_("Custom Redirect URL"),
            help_text=_("Redirect submitters here after proposal completion.")
        )
        ```
2.  **Redirect View Handler:**
    *   Open `src/pretalx/cfp/views/wizard.py`.
    *   Inside the completion handler method `done()`:
        ```python
        custom_url = self.request.event.cfp.settings.get("custom_redirect_url")
        if custom_url:
            return redirect(custom_url)
        return redirect("cfp:event.index")
        ```

---

## Story 22: Allow organizers to bulk-requeue failed email drafts

### Product Manager Specification
*   **Actor:** Event Organizer.
*   **Action:** Provide a checkbox list in the "Outbox failed mails" page. Select multiple entries and click *"Re-queue selected emails"* to move them back into the active pending queue.
*   **Value:** Saves organizers from manually editing and re-sending individual email records when bulk deliveries fail due to SMTP timeouts.
*   **Functional Requirements:**
    1.  Add batch select actions to the failed outbox table template.
    2.  Modify the mail view's POST handler to support state updates.

### Mid-Level Implementation Plan
1.  **Controller Action Implementation:**
    *   Open `src/pretalx/orga/views/mail.py` (or mail detail views files).
    *   Create batch requeue view:
        ```python
        class RequeueFailedMailsView(PermissionRequired, View):
            def post(self, request, *args, **kwargs):
                mail_ids = request.POST.getlist("mail_id")
                # Enforce event scoping
                with scope(event=self.request.event):
                    QueuedMail.objects.filter(
                        pk__in=mail_ids, state="failed"
                    ).update(state="pending")
                return redirect("orga:mails.failed")
        ```

---

## Story 23: Allow reviewers to save drafts of reviews before submitting

### Product Manager Specification
*   **Actor:** Event Reviewer.
*   **Action:** Add a "Save as Draft" button alongside the "Submit Review" button in the scorecard form. Draft reviews should not affect a talk's aggregate score or be visible to other reviewers until published.
*   **Value:** Allows reviewers to record notes, take breaks, and finalize scorecards at a later time.
*   **Functional Requirements:**
    1.  Add `is_draft` boolean to the `Review` model (default: `False`).
    2.  If `is_draft = True`, exclude the score from all aggregate ranking calculations.

### Mid-Level Implementation Plan
1.  **Add Database Field:**
    *   Open `src/pretalx/submission/models/review.py`.
    *   Add `is_draft = models.BooleanField(default=False)`.
2.  **Update Aggregate Score View logic:**
    *   Ensure review aggregation loops explicitly check draft boundaries:
        ```python
        # Filter draft reviews from score averages
        scores = review.submission.reviews.filter(is_draft=False)
        ```
3.  **Run Migrations:**
    *   Execute model migration commands.

---

## Story 24: Enforce rate limiting on the CfP submission endpoint

### Product Manager Specification
*   **Actor:** System Operator.
*   **Action:** Enforce a maximum threshold of 5 proposal submissions per IP address every hour.
*   **Value:** Protects resource queues and email SMTP systems against denial-of-service or automated spam scripts.
*   **Functional Requirements:**
    1.  Add a rate-limiting middleware using django's cache backend or django-ratelimit.
    2.  If the rate limit is exceeded, return `HTTP 429 Too Many Requests`.

### Mid-Level Implementation Plan
1.  **Implement Rate Limiting view decorator:**
    *   Open `src/pretalx/cfp/views/wizard.py`.
    *   Import and wrap `SubmitWizard` view class with django-ratelimit decorators:
        ```python
        from django_ratelimit.decorators import ratelimit

        @method_decorator(ratelimit(key='ip', rate='5/h', method='POST', block=True), name='dispatch')
        class SubmitWizard(SessionWizardView):
            ...
        ```

---

## Story 25: Display a public speaker directory with biographies search

### Product Manager Specification
*   **Actor:** Public Site Visitor.
*   **Action:** Enable a "Public Speakers List" tab in the public agenda page. Include an input search field to filter speakers by name or biography keyword.
*   **Value:** Increases event engagement and allows visitors to browse all confirmed speakers easily.
*   **Functional Requirements:**
    1.  Add search text inputs.
    2.  Execute database queries searching speaker profiles.

### Mid-Level Implementation Plan
1.  **Public Speaker View Controller:**
    *   Create `SpeakerDirectoryView` in `src/pretalx/agenda/views/speaker.py`:
        ```python
        class SpeakerDirectoryView(View):
            def get(self, request, *args, **kwargs):
                query = request.GET.get("q", "")
                speakers = SpeakerProfile.objects.filter(
                    user__name__icontains=query,
                    submission__state="confirmed" # Only show confirmed speakers
                ).distinct()
                return render(request, "agenda/speakers.html", {"speakers": speakers, "q": query})
        ```
