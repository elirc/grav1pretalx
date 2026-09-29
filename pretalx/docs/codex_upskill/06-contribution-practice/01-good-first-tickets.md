# Good First Tickets

This module provides mock junior-to-mid engineering tickets mapping directly to **pretalx** internals.

---

## Ticket 1: Prevent Inviting the Submitter as an Additional Speaker

**Difficulty:** Easy  
**Estimated time:** 1.5 hours  
**Skills practiced:** Form input validation, Model queries, Exception boundaries.  

**Story:** As a submitter entering the Call for Proposals, I want the system to block me from entering my own email address in the "additional speaker invites" field so that I do not send duplicate self-invitations.  

**Why this is a good contribution:** Prevents logical loops and clean email draft spam.

**Acceptance criteria:**
- [ ] Submitting a proposal with the creator's email address in the `invite_addresses` list throws a validation error.
- [ ] The CFP wizard UI renders a clear warning: *"You cannot invite yourself as an additional speaker."*
- [ ] Fully backed by unit tests verifying the exception boundary.

**Read these anchors first:**
- [`src/pretalx/submission/domain/submission.py#L171-L191`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L171-L191) - Analyzes how `apply_invite_addresses` dispatches speaker invitation tokens.

**Files likely touched:**
- [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py) - Add email matching checks inside `apply_invite_addresses`.

**Implementation plan:**
1.  Open `src/pretalx/submission/domain/submission.py`.
2.  Inside `apply_invite_addresses(submission, addresses, *, sender)`, loop over the incoming `addresses` list.
3.  Check if `address.lower() == sender.email.lower()`.
4.  If matched, raise a `SubmissionError(_("You cannot invite yourself as an additional speaker."))`.
5.  Open `src/tests/submission/test_submission_domain.py` and write a test case sending the creator's email address.

**Illustrative fake-code shape:**
```python
# Illustrative fake code: adapt to the repo.
for address in addresses:
    if address.lower() == sender.email.lower():
        raise SubmissionError("Cannot invite self")
```

**What could go wrong:**
Forgetting to compare using `.lower()` or using the `is` identity operator instead of `==` (identity vs value comparison).

**Suggested checks:**
*   Run `pytest src/tests/` to verify tests pass.

**Review questions:**
*   How do you ensure this check does not crash when the sender is AnonymousUser?
