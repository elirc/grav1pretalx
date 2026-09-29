# Python for JS/TS Developers

This module bridges core Python concepts directly to your existing JavaScript/TypeScript mental models, using real production code from **pretalx** as the primary source of examples.

---

## 1. Syntax Bridge: Indentation, Identity, and Truthiness

### Indentation vs Braces
In JavaScript, braces `{}` delineate blocks; indentation is strictly style. In Python, indentation *is* block scoping. 
*   **Why it trips up JS devs:** Missing a tab or mixing spaces and tabs throws a syntax error.
*   **Real Code:** Look at `get_duration` inside [`src/pretalx/submission/models/submission.py#L375-L379`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L375-L379):
    ```python
    def get_duration(self) -> int:
        if self.duration is None:  # Indented 4 spaces
            return self.submission_type.default_duration  # Indented 8 spaces
        return self.duration  # Indented 4 spaces
    ```

### `self` vs `this`
*   In JS, `this` is dynamically bound at runtime based on *how* a function is called, often leading to binding bugs.
*   In Python, `self` is explicitly declared as the first argument of any instance method. It represents the instance of the class.
```javascript
// JS/TS
class Submission {
  constructor(title) { this.title = title; }
  getTitle() { return this.title; } // Implicit 'this'
}
```
```python
# Python/Django
class Submission(models.Model):
    def get_email_locale(self, fallback=None):
        # 'self' is passed explicitly as the first argument!
        if self.content_locale in self.event.locales:
            return self.content_locale
```

### `None` vs `null` & `undefined`
*   JS has both `null` (explicit empty pointer) and `undefined` (variable has not been initialized).
*   Python has **only** `None`. If you try to access a non-existent property on a class, Python immediately raises an `AttributeError`.

### Truthy/Falsy & Identity Checks
*   **Truthy/Falsy Gotcha:** In JS, an empty array `[]` is truthy. In Python, empty collections `[]`, `{}`, `set()` are falsy!
*   **Identity Check (`is` vs `==`):** In JS, you use `===` to prevent implicit conversions. In Python, `==` is safe and calls a class's `__eq__` operator comparison. The `is` operator checks memory address equivalence (reference identity).

### 💡 5-Minute Drill
Rewrite the following JS function into Python, respecting truthiness and explicit `self` reference:
```javascript
function validateProposal(proposal) {
  if (proposal.speakers === null || proposal.speakers.length === 0) {
    return false;
  }
  return true;
}
```
**Answer:**
```python
def validate_proposal(self, proposal) -> bool:
    # Empty lists evaluate as falsy!
    if not proposal.speakers:
        return False
    return True
```

---

## 2. Functions, Closures, & Decorators

### Default Arguments (The Mutable default trap)
*   **JS/TS:** Default parameters are evaluated each time the function is called.
*   **Python:** Default parameters are evaluated **only once** when the function is defined. If you use a mutable default like `def add_speaker(speakers=[])`, every call shares the exact same list instance!
*   **Real Code:** Look at `create_submission` inside [`src/pretalx/submission/domain/submission.py#L36-L45`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L36-L45):
    ```python
    def create_submission(
        *,
        submission,
        user,
        orga=False,
        speakers=(),  # Immutable tuple prevents shared-state mutations!
        tags=(),
        invite_addresses=(),
        send_initial_mails=False,
    ):
    ```

### Rest/Spread (`*args` and `**kwargs`)
*   JS uses the rest/spread operator `...`.
*   Python uses `*args` to collect positional arguments as a tuple, and `**kwargs` to collect keyword arguments as a dictionary.
```javascript
// JS/TS
function logAction(action, ...details) {
  console.log(action, details);
}
```
```python
# Python
def log_action(self, action, data=None, **kwargs):
    # kwargs collects extra keyword parameters into a dictionary
    return super().log_action(action=action, data=data, **kwargs)
```

### Decorators vs Higher-Order Functions (HOFs)
In JavaScript, you wrap functions manually or use Express middleware array pipelines:
```javascript
const authenticatedRoute = (req, res, next) => { ... }
app.get('/dashboard', authenticatedRoute, getDashboard);
```
In Python, `@decorator` syntax is a clean compiler hook that passes the underlying function through a wrapper:
*   **Real Code:** Let's look at predicate rules inside [`src/pretalx/submission/rules.py#L11-L17`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/rules.py#L11-L17):
    ```python
    @rules.predicate
    def reviewer_can_create_tags(user, obj):
        event = obj.event
        return bool(
            event.active_review_phase
            and event.active_review_phase.can_tag_submissions == "create_tags"
        )
    ```

---

## 3. Data Structures: Comprehensions vs Methods

### Comprehensions vs `.map()/.filter()`
In JS, you transform arrays using method chains:
```javascript
const speakerNames = speakers.map(s => s.name);
const activeSpeakers = speakers.filter(s => s.isActive);
```
In Python, the standard idiom is a **comprehension**:
*   **Real Code:** Look at `_collect_content_fields` inside [`src/pretalx/submission/domain/submission.py#L393-L397`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L393-L397):
    ```python
    lines = [line for r in self.resources.all() if (line := r.as_markdown)]
    ```
    *(Bridge Note: The `[x for x in list if condition]` acts as both a map and a filter in one pass!).*

---

## 4. Classes, Properties, & Dunder Methods

### Instance Initialization
*   JS uses the `constructor` keyword.
*   Python uses the special double-underscore ("dunder") method `__init__`.

### `@property` vs Getters
In JS, you define getters directly inside the class:
```javascript
class Submission {
  get imageUrl() { return this.image ? this.image.url : ""; }
}
```
In Python, you decorate an instance method with `@property` or `@cached_property` (cached so that multiple accesses don't repeat calculations):
*   **Real Code:** Look at `image_url` and `slot` inside [`src/pretalx/submission/models/submission.py#L301-L304`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L301-L304):
    ```python
    @property
    def image_url(self):
        return self.image.url if self.image else ""

    @cached_property
    def slot(self):
        ...
    ```

### Dunder Methods vs Symbol Protocols
In JS, custom string representations require defining `toString()`. In Python, you implement the `__str__` dunder method:
```python
    def __str__(self):
        if self.pk:
            return f"Submission(event={self.event.slug}, code={self.code}, title={self.title}, state={self.state})"
```

---

## 5. Modules, Packages, and Imports

### ES Module Hoisting vs Circular Imports
*   **JS/TS ES Modules:** Imports are statically analyzed and hoisted. Circular imports are resolved using placeholders, usually executing without crash.
*   **Python:** Imports are executed **procedurally at runtime**. If `models.py` imports `views.py`, and `views.py` imports `models.py` at the top level of the file, Python immediately crashes with a `CircularDependencyError` or `ImportError`.
*   **The pretalx Pattern:** Move sub-module imports inside specific methods or functions ("leaf imports") to defer import resolution until execution time.
*   **Real Code:** Examine `confirm` method inside [`src/pretalx/submission/models/submission.py#L401-L406`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L401-L406):
    ```python
    def confirm(self, person=None, orga: bool = False):
        from pretalx.submission.domain.submission import (  # Inline local import prevents circular crash!
            set_submission_state,
        )
        set_submission_state(self, SubmissionStates.CONFIRMED, person=person, orga=orga)
    ```

---

## 6. Error Handling

### Exception Hierarchies vs Generic Errors
In JS, developers often throw raw generic strings or generic Errors:
```javascript
throw new Error("Invalid state");
```
In Python, idiomatic code uses distinct exception classes:
*   **Real Code:** Examine state signal veto checks in [`src/pretalx/submission/domain/submission.py#L71-L80`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L71-L80):
    ```python
        exceptions = [r[1] for r in responses if isinstance(r[1], SubmissionError)]
        if exceptions:
            raise exceptions[0]
    ```

---

## 🧪 5-Minute Refactoring Kata

Take this Express/Prisma JS controller, refactor it into clean, idiomatic Python using leaf imports, list comprehensions, and robust custom exception handling.

```javascript
// JS/TS Express Controller
async function listFeaturedTalks(req, res) {
  try {
    const talks = await prisma.submission.findMany({
      where: { eventId: req.event.id, isFeatured: true }
    });
    const names = talks.map(t => t.title.toUpperCase());
    res.json({ talks: names });
  } catch (err) {
    res.status(500).json({ error: "Failed to load talks" });
  }
}
```

**Refactored Python/Django Version:**
```python
from django.http import JsonResponse
from pretalx.common.exceptions import SubmissionError

def list_featured_talks(request):
    try:
        # Django-scopes automatically isolates queries to request.event!
        # Django QuerySets are lazy: SQL only executes during list comprehension evaluation.
        talks = request.event.submissions.filter(is_featured=True)
        names = [talk.title.upper() for talk in talks]
        return JsonResponse({"talks": names})
    except Exception as err:
        raise SubmissionError("Failed to load talks") from err
```
