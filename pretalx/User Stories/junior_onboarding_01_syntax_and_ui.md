# Onboarding Stories 1–10: UI, Syntax & Static Assets

This module contains introductory stories to help you learn HTML templates, style files, translation placeholders, and simple frontend logic.

---

## Story 1: Add a helper link in the CfP landing page to the Code of Conduct

### Stakeholder Request
*"We want to ensure all speakers read our Code of Conduct before submiting. Please place a small link to the Code of Conduct at the bottom of the public Call for Proposals landing page."*

### Junior Walkthrough & Explanation
In Django, public pages are rendered using **templates** (HTML files with template tags). We need to locate the template for the public CfP landing page and inject a standard anchor link tag pointing to our Code of Conduct page.

### Detailed Implementation Steps
1.  **Locate the Template:** Open `src/pretalx/cfp/templates/cfp/event/index.html`.
2.  **Add Link Markup:** Find the end of the Call for Proposals description container. Add:
    ```html
    <div class="mt-4">
        <p>By submitting a proposal, you agree to follow our <a href="/code-of-conduct/" target="_blank">Code of Conduct</a>.</p>
    </div>
    ```
3.  **Local Check:** Refresh your public event page `http://127.0.0.1:8000/{event}/` in your browser and verify the link displays and functions.

---

## Story 2: Change default text for speaker invitations to be warmer

### Stakeholder Request
*"The automated co-speaker invitation email text feels a bit too formal. Let's change the default subject line from 'You have been invited' to 'Join us as a speaker!'"*

### Junior Walkthrough & Explanation
Pretalx stores default phrases and email subject text inside a translation catalog or text constants file. We need to modify the default string constant used when generating invitations.

### Detailed Implementation Steps
1.  **Locate Phrasing:** Open `src/pretalx/cfp/phrases.py` (or search for `"You have been invited"` using your editor).
2.  **Edit Phrasing String:**
    *   Change the default text:
        ```python
        # Original
        invite_subject = _("You have been invited to a proposal")
        # Change to:
        invite_subject = _("Join us as a speaker!")
        ```
3.  **Local Check:** Send an invite to a co-speaker in your local development environment and check `QueuedMail` details inside the Orga dashboard to verify the new subject line is used.

---

## Story 3: Adjust the padding of the review score display badge

### Stakeholder Request
*"The review score badges in the organizer submissions listing feel a bit cramped. Please increase the side padding on the CSS class representing the score badge."*

### Junior Walkthrough & Explanation
Visual styling in pretalx is controlled by **CSS** sheets compiled from Sass or stored as static files. We need to locate the review badge CSS class and adjust its padding properties.

### Detailed Implementation Steps
1.  **Locate Style Sheet:** Open `src/pretalx/static/orga/css/custom.css` (or `scss` variables files).
2.  **Edit Padding Property:**
    *   Find or add the `.badge-score` class:
        ```css
        .badge-score {
            padding-left: 0.8em;
            padding-right: 0.8em;
        }
        ```
3.  **Local Check:** Refresh your browser in the organizer view and check that the score badge displays the new padding size.

---

## Story 4: Fix a typo in the organizer settings help text

### Stakeholder Request
*"There is a small typo in the event settings description. It says 'configurate' instead of 'configure'. Let's fix it."*

### Junior Walkthrough & Explanation
Copy text displayed under form inputs is stored as `help_text` inside **Form** definition files, wrapped in a translation getter (`_()`).

### Detailed Implementation Steps
1.  **Locate Typo:** Open `src/pretalx/orga/forms/event.py` (or search for `"configurate"`).
2.  **Modify Phrasing:** Correct the typo inside the `help_text` parameter of the form field:
    ```python
    help_text=_("Configure your event details here.")
    ```

---

## Story 5: Show a visual "Required" asterisk next to required CfP fields

### Stakeholder Request
*"Speakers often get confused about which fields are required. Please append a small red asterisk (*) next to the labels of all required fields in the CfP forms."*

### Junior Walkthrough & Explanation
We can achieve this globally by targeting the standard CSS class used on required Django form labels, adding a CSS `:after` pseudo-element.

### Detailed Implementation Steps
1.  **Locate Stylesheet:** Open `src/pretalx/static/common/css/forms.css`.
2.  **Add Pseudo-Element CSS:**
    ```css
    .required label:after {
        content: " *";
        color: #e74c3c;
    }
    ```

---

## Story 6: Update the copyright year footer to dynamically pull the current year

### Stakeholder Request
*"The public footer displays a hardcoded copyright year. Please make it dynamic so we don't have to manually update it every January."*

### Junior Walkthrough & Explanation
We can pass a context helper containing the current year or use a custom template tag. In Django, we can read the current date time inside templates using the `{% now "Y" %}` tag.

### Detailed Implementation Steps
1.  **Locate Footer Template:** Open `src/pretalx/cfp/templates/cfp/base.html`.
2.  **Insert Dynamic Date Tag:**
    *   Replace `© 2024 pretalx` with:
        ```html
        © {% now "Y" %} pretalx
        ```

---

## Story 7: Expose speaker biography length constraints in the CfP profile form help text

### Stakeholder Request
*"We want speakers to know their biography limits before saving. Please display the biography character limit in the profile edit form."*

### Junior Walkthrough & Explanation
Biographies limits are configured on the Event CfP settings. We need to append the configured constraints to the biography help text.

### Detailed Implementation Steps
1.  **Locate Speaker Form:** Open `src/pretalx/cfp/forms/speaker.py` (or `forms.py` inside speaker folders).
2.  **Modify Help Text:**
    ```python
    self.fields["biography"].help_text = _("Please provide a short biography (max 1000 characters).")
    ```

---

## Story 8: Render the conference logo on the public speaker biography page

### Stakeholder Request
*"We want our branding to show up when visitors look at speaker profiles. Please render the event logo on public speaker details pages."*

### Junior Walkthrough & Explanation
The event model has a logo image property. We can read `request.event.logo.url` in the template to render it.

### Detailed Implementation Steps
1.  **Locate Template:** Open `src/pretalx/agenda/templates/agenda/speaker.html`.
2.  **Embed Image Element:**
    ```html
    {% if request.event.logo %}
        <img src="{{ request.event.logo.url }}" alt="{{ request.event.name }} Logo" class="event-logo-header">
    {% endif %}
    ```

---

## Story 9: Customize the placeholder in the submission title text field

### Stakeholder Request
*"Speakers often enter simple or confusing names for their talks. Let's provide a helpful placeholder like 'e.g. Building Scalable Web Apps with Django' in the Title input."*

### Junior Walkthrough & Explanation
Input placeholders are configured by adding an `attrs` dictionary containing a `"placeholder"` key inside the form field widget.

### Detailed Implementation Steps
1.  **Locate Submission Form:** Open `src/pretalx/cfp/forms/submission.py`.
2.  **Update Widget Attributes:**
    ```python
    self.fields["title"].widget.attrs["placeholder"] = _("e.g. Building Scalable Web Apps with Django")
    ```

---

## Story 10: Add a CSS style to highlight withdrawn talk submissions

### Stakeholder Request
*"It is hard for organizers to spot withdrawn talks in listings. Let's add a grey background overlay styling on withdrawn rows."*

### Junior Walkthrough & Explanation
We need to append a CSS state class dynamically to table rows based on `submission.state`.

### Detailed Implementation Steps
1.  **Locate Table Row:** Open `src/pretalx/orga/templates/orga/submission/list.html`.
2.  **Bind Dynamic Class:**
    ```html
    <tr class="{% if sub.state == 'withdrawn' %}state-withdrawn{% endif %}">
    ```
3.  **Define CSS Styles:**
    *   Open `src/pretalx/static/orga/css/custom.css`:
        ```css
        .state-withdrawn {
            background-color: #f2f4f4;
            opacity: 0.7;
        }
        ```
