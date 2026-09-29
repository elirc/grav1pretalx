# Systematic Debugging in pretalx

This guide outlines a strict, scientific debugging methodology to resolve bugs within the **pretalx** system, comparing Python and JavaScript diagnostic tools.

---

## Scenario: A speaker gets an HTTP 500 error when clicking "Accept Invitation"

**Reproduction:**
1.  Log in as a standard user.
2.  Navigate to the speaker invitation link sent via email.
3.  Click the "Accept Invitation" button.
4.  Observe the page crash with a generic `HTTP 500 Internal Server Error`.

**First question:** Is the bug in [layer A: CFP view layer] or [layer B: domain signal handlers]?

**Narrowing path:**
1.  Check the browser Developer Tools Network Tab. Observe if the POST request to the accept endpoint fails with a 500 status.
2.  If yes, inspect the local console terminal for Python Tracebacks. Locate the specific file and line that raised the exception.
3.  If the traceback points to a database scoping exception, the issue resides in the Multi-Tenant scope manager.
4.  If the traceback points to `SMTPException` or email rendering, the bug is in the background outbox template dispatcher.

**Useful probes:**
*   Log/breakpoint/query/tool:
    Insert `breakpoint()` directly inside the view handler:
    ```python
    # Inside cfp/views/wizard.py
    def accept_invitation(request, code):
        breakpoint()  # Drops into active pdb shell to inspect request context
        ...
    ```

**Likely root causes:**
*   Forgetting to wrap database operations inside the event's scoped manager:
    ```python
    # Buggy Code
    submission = Submission.objects.get(code=code) # Raises ScopeError!
    ```
    *Correction:*
    ```python
    with scope(event=request.event):
        submission = Submission.objects.get(code=code)
    ```

**Regression test to add:**
*   Add a test case in `tests/cfp/views/test_invitation.py` that mocks an invitation accept POST request for a submission that belongs to a different event, verifying that the client receives a 404/403 rather than a 500 crash.

**Senior lesson:** Always ensure database queries inside anonymous or transient flows explicitly open active scoping transaction fences, preventing runtime tenant boundary leaks.
