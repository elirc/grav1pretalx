# Language and Runtime Interview Prep

This module contains direct, high-impact interview questions comparing Python and JavaScript/TypeScript runtime behaviors, complete with junior, mid-level, and senior-graded answers grounded in the architecture of **pretalx**.

---

## 🐍 Python Language Questions (with JS/TS Comparisons)

### Q1: Explain Python's GIL (Global Interpreter Lock). What does it mean for web applications?
*   **What the interviewer is testing:** Knowledge of multi-threading, concurrency models, CPU-bound vs I/O-bound bottlenecks, and parallel scheduling.
*   **Where to find it in this repo:** [`src/pretalx/celery_app.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/celery_app.py) (uses multiple background process workers to bypass GIL limitations).

#### Answer Guide
*   **Junior:** "The GIL is a lock that makes sure only one Python thread runs at a time. It prevents multi-threaded Python programs from taking full advantage of multi-core processors."
*   **Mid-Level:** "The GIL ensures memory safety in CPython by synchronizing access to objects. For I/O-bound web apps (like fetching a database row in Django), the thread releases the GIL while waiting for the database, so it's not a major bottleneck. However, CPU-bound tasks (like rendering PDF schedule reports in `reportlab`) block other threads."
*   **Senior:** "Since Django routes run synchronously in standard WSGI setups, we bypass GIL issues by scaling out *horizontally* using process-based workers (like Gunicorn/Uwsgi) instead of threads. For intensive async workflows (like scheduling transactional event mailers), we delegate tasks to Celery workers running on distinct OS processes, completely avoiding GIL contention."
*   **JS/TS Bridge:** JavaScript has no GIL because the V8 engine runs strictly on a single-threaded event loop. Node.js handles I/O-bound work asynchronously via libuv, and scales CPU-bound work via `worker_threads` or cluster processes.

---

### Q2: What happens when you use a mutable default argument in a Python function?
*   **What the interviewer is testing:** Understanding of object instantiation lifecycles, memory references, and code safety.
*   **Where to find it in this repo:** [`src/pretalx/submission/domain/submission.py#L36-L45`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L36-L45) (explicitly uses `speakers=()` instead of `speakers=[]`).

#### Answer Guide
*   **Junior:** "Mutable defaults like lists `[]` are evaluated once when the function is defined, so every time you call the function without arguments, it keeps adding elements to the exact same list."
*   **Mid-Level:** "In Python, default arguments are evaluated only once at definition time. A list is a mutable object, so using `def add_speaker(speakers=[])` behaves as a shared static singleton. If one call modifies `speakers`, all future calls receive the contaminated list. The fix is to use an immutable tuple `speakers=()` or a default of `None` and instantiate a new list inside the method."
*   **Senior:** "In pretalx, domain services enforce compile-time safety by defaulting sequence arguments to empty tuples `speakers=()`. Tuples are immutable, so any attempt to append to them raises an error immediately, guaranteeing that call boundaries remain clean and side-effect free."
*   **JS/TS Bridge:** JavaScript has no equivalent gotcha: JS default parameter expressions are re-evaluated **each time** a function is invoked.

---

### Q3: Explain list comprehensions vs generator expressions. When do you use each?
*   **What the interviewer is testing:** Memory efficiency, lazy evaluation, big-O overheads, and streaming datasets.
*   **Where to find it in this repo:** [`src/pretalx/submission/domain/submission.py#L560-L602`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py#L560-L602) (uses `_collect_content_fields` with generator yields to build email templates lazily).

#### Answer Guide
*   **Junior:** "List comprehensions use brackets `[x for x in list]` and create a list. Generator expressions use parentheses `(x for x in list)` and are lazy."
*   **Mid-Level:** "List comprehensions immediately construct the entire list in memory (eager evaluation). Generator expressions return an iterator that yields elements one at a time on demand (lazy evaluation). We use generators to save memory when looping over massive records or streaming database rows."
*   **Senior:** "In pretalx, building placeholders for transactional emails uses generator sequences `_collect_content_fields`. Since we only need to yield label-value strings sequentially, using a generator prevents constructing a massive intermediate list in RAM, minimizing allocations."
*   **JS/TS Bridge:** List comprehensions resemble eager `.map().filter()` method chains. Generator expressions are equivalent to ES6 generator functions (`function*`) yielding values lazily.

---

## 🪟 JS/TS Language Questions (with Python Comparisons)

### Q4: Explain the event loop, microtask queue, and macrotask queue.
*   **What the interviewer is testing:** Async execution models, execution ordering, and non-blocking I/O runtime systems.

#### Answer Guide
*   **Junior:** "The event loop is what allows JavaScript to run async code. Macrotasks are things like `setTimeout`, and microtasks are Promises."
*   **Mid-Level:** "JavaScript runs on a single thread. Positional synchronous code executes immediately on the call stack. When async work completes, callbacks are queued. The event loop checks the stack; if empty, it drains the *microtask queue* (Promises, `process.nextTick`) completely before pulling the next single item from the *macrotask queue* (`setTimeout`, network event callbacks)."
*   **Senior:** "In enterprise JS CRUD apps, misunderstanding microtask starvation can block the event loop. If recursive promise resolutions flood the microtask queue, macrotasks like incoming HTTP requests are stalled indefinitely. We offload heavy CPU cycles to cluster modules or background processes."
*   **Python Bridge:** Python is synchronous by default. Async concurrency is opt-in via `import asyncio`, which runs an event loop structurally similar to JS but explicit rather than implicit.

---

### Q5: What is a discriminated union in TypeScript, and how would you model it in Python?
*   **What the interviewer is testing:** Strong type safety, structural typing, and pattern matching in domain domains.

#### Answer Guide
*   **Junior:** "A discriminated union is a type of object that has a common string field like `type` that TypeScript uses to narrow down the properties you can access."
*   **Mid-Level:** "Discriminated unions combine type unions with a literal field. For instance, `type Action = { type: 'accept' } | { type: 'reject', reason: string }`. TypeScript guards refine the type within block scope when checking the value. In Python, we use the `typing.Literal` and `Union` types, or python 3.10+ `match` case pattern matching."
*   **Senior:** "Discriminated unions are excellent for state machine transitions. In pretalx, we model submission states via `SubmissionStates` enum choices. In modern typed Python, we can narrow down states cleanly using duck typing, nominal structural protocols (`typing.Protocol`), or class-based inheritance using pattern matching."
*   **TS Code:**
    ```typescript
    type StateChange = { type: "accept" } | { type: "reject"; reason: string };
    function handle(action: StateChange) {
      if (action.type === "reject") console.log(action.reason);
    }
    ```
*   **Python Equivalent:**
    ```python
    from typing import Literal, Union, TypedDict

    class AcceptDict(TypedDict):
        type: Literal["accept"]

    class RejectDict(TypedDict):
        type: Literal["reject"]
        reason: str

    StateChange = Union[AcceptDict, RejectDict]
    ```
