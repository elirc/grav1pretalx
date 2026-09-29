# Good First Tickets (Junior/Mid level)

This practice module contains real-world styled engineering tickets mapped directly to **pretalx's** source files.

---

## Ticket 1: Prevent Inviting the Submitter as an Additional Speaker

**Difficulty:** Easy  
**Estimated time:** 1.5 hours  
**Skills practiced:** Form input validation, Model queries, Exception boundaries.  
**Story:** As a submitter entering the Call for Proposals, I want the system to block me from entering my own email address in the "additional speaker invites" field so that I do not send duplicate self-invitations.  
**Why this is a good contribution:** Prevents logical loops and clean email draft spam.

### Acceptance Criteria
- [ ] Submitting a proposal with the creator's email address in the `invite_addresses` list throws a validation error.
- [ ] The CFP wizard UI renders a clear warning: *"You cannot invite yourself as an additional speaker."*
- [ ] Fully backed by unit tests verifying the exception boundary.

### Read These Anchors First
*   [`src/pretalx/submission/domain/submission.py#L171-L191`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L171-L191) — Analyzes how `apply_invite_addresses` dispatches speaker invitation tokens.

### Files Likely Touched
*   [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) — Add email matching checks inside `apply_invite_addresses`.

### Implementation Plan
1.  Open `src/pretalx/submission/domain/submission.py`.
2.  Inside `apply_invite_addresses(submission, addresses, *, sender)`, loop over the incoming `addresses` list.
3.  Check if `address.lower() == sender.email.lower()`.
4.  If matched, raise a `SubmissionError(_("You cannot invite yourself as an additional speaker."))`.
5.  Open `src/tests/submission/test_submission_domain.py` and write a test case sending the creator's email address.

**JS Developer Bridge:** Equivalent to checking `req.user.email === req.body.inviteEmail` in an Express invitation endpoint and throwing a `400 Bad Request`.

**What could go wrong (Python Gotcha):** Forgetting to compare using `.lower()` or using the `is` identity operator instead of `==` (identity vs value comparison).

**Suggested Checks:**
*   Run `pytest src/tests/` to verify tests pass.

---

## Ticket 2: Enforce Maximum Proposal Image File Size Limit

**Difficulty:** Easy  
**Estimated time:** 2 hours  
**Skills practiced:** File properties, Forms validation, Config lookups.  
**Story:** As an event organizer, I want the proposal wizard to block image uploads larger than 5MB so that our server storage is not overloaded by un-optimized speaker pictures.  
**Why this is a good contribution:** Enforces performance limits on static storage.

### Acceptance Criteria
- [ ] Proposal image uploads exceeding 5 Megabytes are rejected by the CFP wizard.
- [ ] Renders a clear validation error message: *"The image file is too large (maximum size is 5MB)."*

### Read These Anchors First
*   [`src/pretalx/submission/models/submission.py#L178-L184`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L178-L184) — Identifies the `image` field definition on the `Submission` model.
*   [`src/pretalx/cfp/views/wizard.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/views/wizard.py) — The submission wizard view controller.

### Files Likely Touched
*   [`src/pretalx/cfp/forms/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/cfp/forms/submission.py) — Clean methods on forms fields.

### Implementation Plan
1.  Locate `SubmissionForm` inside the `cfp/forms/` directory.
2.  Implement a `clean_image(self)` validation hook method.
3.  Extract `self.cleaned_data.get('image')`.
4.  If the image exists, check `image.size > 5 * 1024 * 1024`.
5.  If too large, raise `forms.ValidationError(_("The image file is too large (maximum size is 5MB)."))`.

**JS Developer Bridge:** Equivalent to standard Multer file filter middleware checking `file.size` in Express routers.

**What could go wrong (Python Gotcha):** Django form clean hooks must *always* return the cleaned value at the end of the method:
```python
def clean_image(self):
    image = self.cleaned_data.get('image')
    if image and image.size > 5 * 1024 * 1024:
        raise ValidationError(...)
    return image  # ⚠️ Don't forget this, otherwise self.cleaned_data['image'] is set to None!
```

---

## Ticket 3: Prevent Multiple Active Review Phases from Overlapping

**Difficulty:** Medium  
**Estimated time:** 3.5 hours  
**Skills practiced:** Database constraint checking, Date comparisons, Signal overrides.  
**Story:** As an event organizer, I want the review phase settings to validate that no two phases are configured with overlapping timelines so that reviewers do not see duplicate reviews.  
**Why this is a good contribution:** Protects chronological sequence rules.

### Acceptance Criteria
- [ ] Saving an organizer review phase validates that the start/end dates do not overlap with any existing review phases on the same event.
- [ ] Overlap attempts throw a validation error.

### Read These Anchors First
*   [`src/pretalx/submission/domain/review.py#L36-L52`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/review.py#L36-L52) — Exposes `validate_review_phases` logic.

### Files Likely Touched
*   [`src/pretalx/submission/domain/review.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/review.py) — Tweak validation loops.
