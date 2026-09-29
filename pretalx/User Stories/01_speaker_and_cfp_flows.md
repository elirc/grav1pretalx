# Module 1: Speaker & CfP Flows

This module contains User Stories 1 to 5, targeting proposal submission boundaries and speaker interaction lifecycles.

---

## Story 1: Restrict Speaker Abstract Length by Word Count (Instead of Character Count)

### Product Manager Specification
*   **Actor:** Speaker (Submitter) & Event Organizer.
*   **Action:** Enable a configuration switch in the CfP settings panel to enforce word count validation on the submission `abstract` field.
*   **Value:** Standardizes submission abstract sizes by paragraph count rather than alphabetical character length, which accommodates multi-lingual glyph differences.
*   **Functional Requirements:**
    1.  Add a dropdown selector `"count_length_in"` under `Orga CfP settings` allowing the options: `"Characters"` vs `"Words"`.
    2.  When set to `"Words"`, form validations on the proposal wizard must count words separated by standard spaces and punctuation using a regex boundary matching `\b\w+\b`.
    3.  Help text below the abstract inputs must update dynamically: e.g. *"Please write between 50 and 200 words. You wrote X words."*
    4.  Verify word limit configurations on save.

### Mid-Level Implementation Plan
1.  **Locate Configuration Fields:**
    *   Open [`src/pretalx/cfp/forms.py#L77-L100`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/forms.py#L77-L100). The `RequestRequire` mixin already reads `self.event.cfp.settings["count_length_in"]`.
2.  **Expose Settings in Orga Panel Form:**
    *   Open `src/pretalx/orga/forms/cfp.py` (or search for CfPSettingsForm using `grep_search`).
    *   Add a choice field `count_length_in` under CfP settings:
        ```python
        count_length_in = forms.ChoiceField(
            choices=(("chars", _("Characters")), ("words", _("Words"))),
            label=_("Count text limits in"),
            initial="chars"
        )
        ```
3.  **Form Validation Logic:**
    *   The `validate_field_length` method inside `cfp/forms.py` utilizes:
        ```python
        length = len(re.findall(WORD_REGEX, value))
        ```
    *   Ensure the JavaScript dynamic feedback updates live in `src/pretalx/static/common/js/forms/character-limit.js`. Read the current selection and calculate count based on whitespace split:
        ```javascript
        // Illustrative fake code: JS character-limit.js update
        const countWords = (text) => text.trim().split(/\s+/).filter(Boolean).length;
        ```
4.  **Create Schema Migration:**
    *   If `count_length_in` settings are stored in `cfp.settings` (JSONField), no SQL DDL migration is required. If a database field is added to `Cfp` model, execute:
        ```powershell
        python src/manage.py makemigrations
        ```
5.  **Write Tests:**
    *   Open `src/tests/cfp/test_cfp_wizard.py` and write a test case setting `count_length_in = "words"`. Submit a proposal abstract with 10 words when `min_length = 20` and assert that the validator raises a `ValidationError` containing `"You wrote 10 words."`.

---

## Story 2: Auto-expire Pending Speaker Invitations after X Days

### Product Manager Specification
*   **Actor:** Event Organizer & Speaker Invitees.
*   **Action:** Configure a maximum valid timeframe (e.g. 7 days) for co-speaker invitations. If the invitee does not redeem the access link within that period, the token expires automatically.
*   **Value:** Prevents outstanding unclaimed speaker roles from blocking slot allocations.
*   **Functional Requirements:**
    1.  Add an integer field `invitation_expiry_days` to the Event settings page (default is `None`, meaning infinite validity).
    2.  When a speaker invitation is issued, calculate and store `valid_until` timestamp.
    3.  If a user visits the accept token URL after `valid_until`, render an error page: *"This invitation link has expired."*

### Mid-Level Implementation Plan
1.  **Update Database Model:**
    *   Open [`src/pretalx/submission/models/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py).
    *   Locate the `SpeakerInvite` class (or search using `grep_search`).
    *   Add a DateTimeField `valid_until = models.DateTimeField(null=True, blank=True)`.
2.  **Generate Migration:**
    *   Run migrations:
        ```powershell
        python src/manage.py makemigrations submission
        python src/manage.py migrate
        ```
3.  **Calculate Expiry on Invitation Creation:**
    *   Locate the method dispatching invites inside `domain/submission.py`.
    *   When saving a new `SpeakerInvite` record:
        ```python
        expiry_days = submission.event.settings.get("invitation_expiry_days")
        if expiry_days:
            invite.valid_until = timezone.now() + timedelta(days=int(expiry_days))
        ```
4.  **Enforce Validation in the Redemption View:**
    *   Open the speaker redemption view (e.g. `src/pretalx/cfp/views/wizard.py` or `src/pretalx/cfp/views/invitation.py`).
    *   Inside the token verify method:
        ```python
        if invite.valid_until and timezone.now() > invite.valid_until:
            messages.error(request, _("This invitation link has expired."))
            return redirect("cfp:event.index")
        ```
5.  **Write Tests:**
    *   Create a test case in `src/tests/cfp/` setting `valid_until` to 2 days in the past. Attempt to POST to the accept URL and assert that access is blocked and returns a redirect with an error message.

---

## Story 3: Export Review Scores as CSV for Custom Excel Analysis

### Product Manager Specification
*   **Actor:** Event Organizer (Reviewer Board Chair).
*   **Action:** Add an "Export Reviews to CSV" button in the Reviewer Dashboard of the Orga panel.
*   **Value:** Allows organizers to run complex mathematical rankings, pivot tables, and statistical reviews in Excel.
*   **Functional Requirements:**
    1.  Add a button labelled `"Export CSV"` on `/orga/event/{event}/reviews/`.
    2.  The resulting CSV file must contain columns: `Submission ID`, `Title`, `Track`, `Reviewer Email`, `Reviewer Name`, `Score`, and `Text Review`.
    3.  Generate the CSV dynamically in-memory without holding locks on SQLite threads.

### Mid-Level Implementation Plan
1.  **Add Route in Orga URLs:**
    *   Open `src/pretalx/orga/urls.py` and register the endpoint:
        ```python
        path("reviews/export/", views.ReviewExportView.as_view(), name="reviews.export"),
        ```
2.  **Implement View Controller:**
    *   Create `ReviewExportView(PermissionRequiredView)` inside `src/pretalx/orga/views/review.py`:
        ```python
        import csv
        from django.http import StreamingHttpResponse
        from pretalx.common.views.mixins import PermissionRequired

        class ReviewExportView(PermissionRequired, View):
            permission_required = "orga.view_reviews"

            def get(self, request, *args, **kwargs):
                response = HttpResponse(content_type="text/csv")
                response["Content-Disposition"] = 'attachment; filename="reviews.csv"'
                writer = csv.writer(response)
                writer.writerow(["Submission ID", "Title", "Reviewer Email", "Score", "Text"])
                
                # Enforce multi-tenant scoping boundary
                with scope(event=self.request.event):
                    reviews = Review.objects.select_related("submission", "user").filter(submission__event=self.request.event)
                    for r in reviews:
                        writer.writerow([r.submission.code, r.submission.title, r.user.email, r.score, r.text])
                return response
        ```
3.  **Register Button in HTML Template:**
    *   Open `src/pretalx/orga/templates/orga/review/dashboard.html` and inject the link markup pointing to `{% url 'orga:reviews.export' event=request.event.slug %}`.
4.  **Write Tests:**
    *   Verify view responses using a logged-in orga client in `src/tests/orga/test_review_views.py`. Assert response headers match `content_type="text/csv"`.

---

## Story 4: Allow Speakers to Request Session Room Layout Changes

### Product Manager Specification
*   **Actor:** Speaker.
*   **Action:** Submit a requested room layout selection (e.g. "Theatre", "Classroom", "Workshop Tables") when entering the CfP wizard or editing their accepted proposal.
*   **Value:** Helps venue planners prepare tables and seating layouts dynamically based on talk requirements.
*   **Functional Requirements:**
    1.  Expose a new field `room_layout_preference` on the `Submission` model (choices: `"theatre"`, `"classroom"`, `"workshop"`).
    2.  Render this option as a dropdown selection in the speaker's CFP submission steps.
    3.  Expose the preferred layout inside the Orga dashboard talk detail sidebar.

### Mid-Level Implementation Plan
1.  **Edit Submission Model:**
    *   Open `src/pretalx/submission/models/submission.py`.
    *   Add:
        ```python
        LAYOUT_CHOICES = (
            ("theatre", _("Theatre Seating")),
            ("classroom", _("Classroom Desks")),
            ("workshop", _("Workshop Round Tables")),
        )
        room_layout_preference = models.CharField(
            max_length=20, choices=LAYOUT_CHOICES, default="theatre", blank=True, null=True
        )
        ```
2.  **Add Form Selectors:**
    *   Open `src/pretalx/cfp/forms/submission.py` or inject in `SubmissionForm`.
    *   Include `room_layout_preference` inside the `Meta.fields` list.
3.  **Display in Organizer Template:**
    *   Open `src/pretalx/orga/templates/orga/submission/detail.html` and add a layout display card showing the active preference.
4.  **Run Migrations:**
    *   Execute:
        ```powershell
        python src/manage.py makemigrations submission
        python src/manage.py migrate
        ```

---

## Story 5: Enforce Password Complexity Validator for Organizer Accounts

### Product Manager Specification
*   **Actor:** System Administrator & Event Organizers.
*   **Action:** Enforce strict password complexity checks (minimum 12 characters, at least 1 digit, 1 special character, and 1 uppercase letter) specifically for users with organizer panel access, leaving standard speaker accounts on default validation.
*   **Value:** Protects administrative access bounds against brute-force attempts.
*   **Functional Requirements:**
    1.  Introduce a custom Django password validator module.
    2.  Hook the validator into registration and reset pipelines specifically when registering or upgrading to organizer roles.
    3.  Display a warning detailing the password requirement guidelines on the registration page.

### Mid-Level Implementation Plan
1.  **Write Password Validator Class:**
    *   Create a new file `src/pretalx/common/validators/password.py`:
        ```python
        import re
        from django.core.exceptions import ValidationError
        from django.utils.translation import gettext_lazy as _

        class ComplexityValidator:
            def validate(self, password, user=None):
                if len(password) < 12:
                    raise ValidationError(_("Password must be at least 12 characters."))
                if not re.search(r"[A-Z]", password):
                    raise ValidationError(_("Password must contain at least one uppercase letter."))
                if not re.search(r"\d", password):
                    raise ValidationError(_("Password must contain at least one number."))
            
            def get_help_text(self):
                return _("Must be 12+ characters with an uppercase letter and a number.")
        ```
2.  **Register Custom Validator in Django Settings:**
    *   Open `src/pretalx/settings.py` and append your validator to `AUTH_PASSWORD_VALIDATORS` configuration arrays.
3.  **Conditional Form Assertion:**
    *   Open `src/pretalx/orga/forms/auth.py` and enforce password complexity clean validations directly on password registration fields.
