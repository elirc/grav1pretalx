# Onboarding Stories 11–20: Basic Views & Forms

This module contains introductory stories on Django Views and Forms to help you learn input validation, clean methods, queryset queries, and template context.

---

## Story 11: Enforce a minimum character limit on the proposal title

### Stakeholder Request
*"We want speakers to write descriptive titles for their talks. Please block submissions with titles shorter than 10 characters."*

### Junior Walkthrough & Explanation
In Django, form validation is handled inside **Form classes** using clean hooks (e.g. `clean_fieldname` or `clean()`). We need to implement a clean hook that checks string length and raises a `ValidationError` if constraints are violated.

### Detailed Implementation Steps
1.  **Locate Form Class:** Open `src/pretalx/cfp/forms/submission.py` (or locate `SubmissionForm`).
2.  **Add clean hook:**
    ```python
    def clean_title(self):
        title = self.cleaned_data.get("title")
        if title and len(title) < 10:
            raise forms.ValidationError(_("Talk title must be at least 10 characters long."))
        return title
    ```

---

## Story 12: Block reviewers from entering negative custom review scores

### Stakeholder Request
*"Reviewers occasionally make typing errors and save negative scores. Please validate that custom review scores are always greater than or equal to 0."*

### Junior Walkthrough & Explanation
We validate input scores inside the organizer scorecard submission form using basic validation logic.

### Detailed Implementation Steps
1.  **Locate Review Form:** Open `src/pretalx/orga/forms/review.py`.
2.  **Add validation inside clean method:**
    ```python
    def clean_score(self):
        score = self.cleaned_data.get("score")
        if score is not None and score < 0:
            raise forms.ValidationError(_("Review score cannot be negative."))
        return score
    ```

---

## Story 13: Expose an option to sort speaker listings alphabetically

### Stakeholder Request
*"Finding speakers in a large directory is hard. Please sort the speaker list page alphabetically by default."*

### Junior Walkthrough & Explanation
Default sorting is controlled by Django QuerySets using the `.order_by("fieldname")` query filter modifier.

### Detailed Implementation Steps
1.  **Locate Speaker View:** Open `src/pretalx/agenda/views/speaker.py`.
2.  **Edit QuerySet Order:**
    *   Locate the query returning speaker profiles:
        ```python
        # Original
        speakers = SpeakerProfile.objects.filter(...)
        # Change to:
        speakers = SpeakerProfile.objects.filter(...).order_by("user__name")
        ```

---

## Story 14: Hide the review tab if no active review phase is set

### Stakeholder Request
*"Reviewers click the 'Reviews' dashboard tab when reviews are closed and see confusing empty tables. Please hide the Reviews navigation tab entirely when there is no active review phase."*

### Junior Walkthrough & Explanation
We can conditionally show HTML navigation elements in templates by wrapping them inside an `{% if %}` template check block.

### Detailed Implementation Steps
1.  **Locate Navigation Layout:** Open `src/pretalx/orga/templates/orga/base.html` (or navigation lists files).
2.  **Add Conditional Check:**
    ```html
    {% if request.event.active_review_phase %}
        <li class="nav-item">
            <a class="nav-link" href="{% url 'orga:reviews.dashboard' event=request.event.slug %}">Reviews</a>
        </li>
    {% endif %}
    ```

---

## Story 15: Enforce a checkbox check for speaker terms of service

### Stakeholder Request
*"Before a proposal can be submitted, co-speakers must check an agreement checkbox verifying they own the rights to the session content."*

### Junior Walkthrough & Explanation
We can enforce check requirements on form elements by adding a checkbox field and setting `required=True` inside our Form class definition.

### Detailed Implementation Steps
1.  **Locate Invitation Form:** Open `src/pretalx/cfp/forms/invitation.py` or wizard step forms.
2.  **Inject Checkbox Field:**
    ```python
    agree_terms = forms.BooleanField(
        required=True,
        label=_("I agree to the Speaker Terms of Service"),
        help_text=_("You must accept our terms to join this proposal.")
    )
    ```

---

## Story 16: Add a "Reset Filter" button on the organizer submissions list

### Stakeholder Request
*"When searching and filtering talks in the dashboard, it is tedious to clear filters manually. Let's add a simple 'Clear Filters' button."*

### Junior Walkthrough & Explanation
We can clear parameters from URLs by adding an anchor tag redirecting to the base view path.

### Detailed Implementation Steps
1.  **Locate Filter Form Markup:** Open `src/pretalx/orga/templates/orga/submission/list.html`.
2.  **Inject Reset Link:**
    ```html
    <a href="{% url 'orga:submissions.list' event=request.event.slug %}" class="btn btn-secondary">
        Clear Filters
    </a>
    ```

---

## Story 17: Add a placeholder to the track description text area

### Stakeholder Request
*"Organizers often skip writing track descriptions because the input is empty. Let's provide a helpful placeholder like 'e.g. For topics focusing on web performance and scaling.'"*

### Junior Walkthrough & Explanation
We configure placeholders for long TextAreas by passing the attribute inside a custom `forms.Textarea` widget definition.

### Detailed Implementation Steps
1.  **Locate Track Form:** Open `src/pretalx/orga/forms/track.py`.
2.  **Configure Widget Placeholder:**
    ```python
    description = forms.CharField(
        widget=forms.Textarea(attrs={"placeholder": _("e.g. For topics focusing on web performance and scaling.")}),
        required=False
    )
    ```

---

## Story 18: Log warning message when an organizer updates the CfP dates

### Stakeholder Request
*"When Call for Proposal deadlines are changed, other track chairs get surprised. Let's write a warning alert message in the dashboard when CfP dates are updated."*

### Junior Walkthrough & Explanation
Django handles simple transient user notifications using the **messages framework** (`messages.warning()`).

### Detailed Implementation Steps
1.  **Locate CfP View Controller:** Open `src/pretalx/orga/views/cfp.py`.
2.  **Add Warning Message:**
    *   Inside the form valid save routine:
        ```python
        from django.contrib import messages
        messages.warning(request, _("Warning: Call for Proposals dates have been updated. Ensure you notify track chairs!"))
        ```

---

## Story 19: Add simple character length counter display for the talk abstract

### Stakeholder Request
*"Speakers want to know how close they are to the abstract character limit. Please show a basic remaining character count indicator."*

### Junior Walkthrough & Explanation
We can implement this instantly by attaching a simple vanilla JavaScript text input event listener on the abstract textarea.

### Detailed Implementation Steps
1.  **Locate Template:** Open `src/pretalx/cfp/templates/cfp/submission/abstract.html`.
2.  **Embed Markup & Script:**
    ```html
    <div class="character-counter">
        Characters remaining: <span id="char-count">1000</span>
    </div>
    <script>
        document.addEventListener("DOMContentLoaded", () => {
            const textarea = document.querySelector("#id_abstract");
            const counter = document.querySelector("#char-count");
            if (textarea && counter) {
                textarea.addEventListener("input", () => {
                    counter.innerText = 1000 - textarea.value.length;
                });
            }
        });
    </script>
    ```

---

## Story 20: Enforce lowercase-only formatting on new event slugs

### Stakeholder Request
*"Organizers sometimes create event slugs with mixed casing (e.g. PyCon2026), which looks inconsistent in URLs. Please enforce lowercase-only formatting for event slugs."*

### Junior Walkthrough & Explanation
Form inputs can be programmatically converted inside Django forms clean hooks before saving to the database.

### Detailed Implementation Steps
1.  **Locate Event Form:** Open `src/pretalx/orga/forms/event.py`.
2.  **Convert input to lowercase:**
    ```python
    def clean_slug(self):
        slug = self.cleaned_data.get("slug")
        return slug.lower() if slug else slug
    ```
