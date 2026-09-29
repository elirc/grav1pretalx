# Module 4: Notifications & API Flows

This module contains User Stories 16 to 20, targeting reviewers limits, talk slot statuses, PDF generation, REST API enhancements, and visual schedule clashed warnings.

---

## Story 16: Configure a Maximum Number of Reviewers per Submission

### Product Manager Specification
*   **Actor:** Event Organizer (Reviewer Board Chair).
*   **Action:** Add a settings parameter `max_reviewers_per_talk` (default is `None`, unlimited). If a submission already has `max_reviewers_per_talk` reviews saved, hide it from other reviewers' lists.
*   **Value:** Distributes review allocation fairly, ensuring all talks get reviewed instead of a few popular talks receiving all scoring cards.
*   **Functional Requirements:**
    1.  Add configuration option to review settings.
    2.  Filter dashboard queries to hide talks that have reached the review limit.

### Mid-Level Implementation Plan
1.  **Add Configuration Property:**
    *   Expose `max_reviewers_per_talk` inside Cfp settings or Event details configuration forms.
2.  **Filter Reviewer Dashboard Query:**
    *   Open `src/pretalx/orga/views/review.py` (or dashboard data loaders).
    *   Add condition filtering:
        ```python
        limit = self.request.event.settings.get("max_reviewers_per_talk")
        if limit:
            # Filter out talks where count of reviews >= limit
            queryset = queryset.annotate(review_count=Count('reviews')).filter(review_count__lt=int(limit))
        ```

---

## Story 17: Support "Tentative" Status for Talk Slot Scheduling

### Product Manager Specification
*   **Actor:** Event Organizer.
*   **Action:** Allow talk slots in the visual schedule editor to be set to a "Tentative" status, rendering them in a distinct yellow outline in the agenda layout.
*   **Value:** Helps schedulers block slots for speakers whose attendance is unconfirmed, without publishing them as fully committed schedule entries.
*   **Functional Requirements:**
    1.  Add `"tentative"` option to slot layout choices.
    2.  Update CSS render styling to show yellow background outlines.

### Mid-Level Implementation Plan
1.  **Edit Schedule Model Choice Field:**
    *   Open `src/pretalx/schedule/models/slot.py` (or `TalkSlot` models file).
    *   Append `"tentative"` status to choice lists.
2.  **Style Widget Layout CSS:**
    *   Add styling under Svelte/Vite CSS classes or visual editor components:
        ```css
        /* Illustrative css stylesheet addition */
        .slot-tentative {
            border: 2px dashed #f1c40f;
            background-color: #fef9e7;
        }
        ```

---

## Story 18: Auto-generate speaker badges as PDF downloads

### Product Manager Specification
*   **Actor:** Speaker & Event Organizer.
*   **Action:** Add a "Download My Badge" button to the speaker's dashboard after their talk is fully accepted.
*   **Value:** Allows speakers to download and print their conference entry badges, reducing check-in delays at the registration desks.
*   **Functional Requirements:**
    1.  Introduce a PDF renderer view using reportlab or standard Django templating.
    2.  Populate badge details: speaker's name, talk title, and track metadata.

### Mid-Level Implementation Plan
1.  **Write PDF Generator Helper:**
    *   Create a view `DownloadSpeakerBadgeView` inside `src/pretalx/cfp/views/profile.py`:
        ```python
        from reportlab.pdfgen import canvas
        from django.http import HttpResponse

        class DownloadSpeakerBadgeView(PermissionRequired, View):
            def get(self, request, *args, **kwargs):
                response = HttpResponse(content_type='application/pdf')
                response['Content-Disposition'] = 'attachment; filename="badge.pdf"'
                
                # Draw PDF Badge
                p = canvas.Canvas(response)
                p.drawString(100, 750, f"Speaker: {request.user.name}")
                p.drawString(100, 730, f"Role: Speaker")
                p.showPage()
                p.save()
                return response
        ```

---

## Story 19: Filter Submissions by Tag in the REST API Endpoint

### Product Manager Specification
*   **Actor:** API Integrator / Developer.
*   **Action:** Add filter parameter `?tag=Slug` to `/api/events/{event}/submissions/` endpoint.
*   **Value:** Allows custom widgets (like websites displaying only specific workshop lists) to load filtered lists instead of fetching and filtering full JSON arrays client-side.
*   **Functional Requirements:**
    1.  Extend REST API View QuerySet filters to check and filter tag slugs.

### Mid-Level Implementation Plan
1.  **Modify Serializer view Queryset:**
    *   Open `src/pretalx/api/views/submission.py`.
    *   Override `get_queryset()`:
        ```python
        def get_queryset(self):
            qs = super().get_queryset()
            tag_slug = self.request.query_params.get("tag")
            if tag_slug:
                qs = qs.filter(tags__slug=tag_slug)
            return qs
        ```

---

## Story 20: Display a "Warning: Schedule Clash" Alert in visual editor

### Product Manager Specification
*   **Actor:** Event Organizer (Scheduler).
*   **Action:** In the visual drag-and-drop schedule planner, if an organizer schedules the same speaker to two rooms at overlapping times, display a warning banner: *"Speaker Clash: [Name] is scheduled twice."*
*   **Value:** Eliminates schedule planning errors.

### Mid-Level Implementation Plan
1.  **Enforce Validation Loop in Schedule Reconciliation:**
    *   Locate schedule saving view (or `TalkSlot` change signals).
    *   Run checks:
        ```python
        # Check speaker schedule overlaps
        overlapping = TalkSlot.objects.filter(
            submission__speakers__in=slot.submission.speakers.all(),
            start__lt=slot.end,
            end__gt=slot.start
        ).exclude(pk=slot.pk)
        if overlapping.exists():
            # Send warning in API response payload
            return JsonResponse({"warning": "Speaker Clash detected!"})
        ```
