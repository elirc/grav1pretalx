# JavaScript ↔ Python Rosetta Stone

This quick-reference guide maps core JS/TS conventions, operators, and tooling patterns directly to Python/Django equivalents. Grounded in the design choices of **pretalx**, this stone serves as your instant lookup card.

---

## 🗺️ Core Syntax & Runtime Comparison

| Category | JavaScript/TypeScript | Python (Django) | Critical Gotchas & Bridge Insights |
| :--- | :--- | :--- | :--- |
| **Package Manager** | `npm` / `yarn` / `pnpm` | `pip` / `poetry` / `uv` | Python isolates packages via virtual environments (`venv`) rather than a local `node_modules` folder. |
| **Lock File** | `package-lock.json` | `requirements.txt` / `poetry.lock` | Standard `pip` has no native auto-locking mechanism; modern setups use `uv lock` or `poetry.lock`. |
| **Entry Point** | `index.js` / `main.ts` | `__init__.py` / `__main__.py` | In Python, code inside `__init__.py` runs immediately whenever the package or any sub-module is imported. |
| **Null Value** | `null` / `undefined` | `None` | Python has no distinct equivalent to `undefined`. A missing object key or variable raises a `KeyError` or `NameError`. |
| **Equality** | `===` (Strict) / `==` (Loose) | `==` (Value) / `is` (Identity) | **Never use `is` for value equality.** In Python, `is` checks reference identity (memory address); `==` checks value equivalence. |
| **Truthiness** | `0, "", null, undefined, NaN` are falsy. | `0, "", None, [], {}, set()` are falsy. | **Critical Difference:** In JavaScript, empty arrays `[]` and objects `{}` are **truthy**. In Python, empty lists `[]`, dicts `{}`, and sets `set()` are **falsy**. |
| **String Format** | Template literal `` `${var}` `` | f-string `f"{var}"` | Modern Python f-strings are functionally identical to JS backtick template literals. |
| **Spread Operator** | Unpack: `...arr` / Collect: `...rest` | Unpack: `*arr`, `**dict` / Collect: `*args`, `**kwargs` | In Python, single asterisk `*` unpacks sequences (lists/tuples); double asterisk `**` unpacks mappings (dicts). |
| **Destructuring** | `const { title, state } = proposal;` | `title, state = (proposal.title, proposal.state)` | Python natively unpacks sequences (tuples/lists) but does not have JS-style object key destructuring. |
| **Lambda** | `(x) => x + 1` | `lambda x: x + 1` | Python lambda expressions are restricted to a single expression. They cannot contain statements or multi-line blocks. |
| **Async Operations** | `async/await` + `Promise.all()` | `async/await` + `asyncio.gather()` | **GIL Bound:** Python's event loop is opt-in (`asyncio`). Default Django routes run synchronously unless configured using ASGI. |
| **Testing Framework** | Jest / Vitest | `pytest` / `unittest` | `pytest` uses functional fixtures and parameterization. It is much more idiomatic than Django's standard JUnit-like `TestCase`. |
| **Linting & Format** | ESLint + Prettier | `ruff` (combines Flake8 + Black) | `ruff` compiles to Rust and is incredibly fast, instantly cleaning imports and checking complexity. |
| **Type System** | TypeScript (Structural) | `mypy` / `pyright` (Nominal / Duck) | Python type annotations are fully optional and ignored at runtime. Type checkers run strictly in pre-commit/CI environments. |
| **ORM** | Prisma / Drizzle / TypeORM | Django ORM / SQLAlchemy | **Django ORM Lazy Queries:** Queries like `Submission.objects.filter()` *do not execute SQL* until the QuerySet is iterated. |
| **Router Registry** | Express Router / File-based (Next.js) | `urls.py` pattern routing | Router bindings in Django require explicit registration in a nested tree of `urls.py` config modules. |
| **Middleware Chain** | `app.use((req, res, next) => {})` | Nested Classes: `__call__(self, request)` | In Django, middleware is modeled as a stack of nested classes wrapping callables (e.g. `EventPermissionMiddleware`). |
| **UI Templating** | JSX / TSX / Handlebars | Django Templates / Jinja2 | Django templates are explicitly designed to be *logic-less*. You cannot execute arbitrary Python code inside templates. |
| **REPL Shell** | Node REPL / Chrome Console | `python manage.py shell` | The Django shell loads up fully booted framework contexts, pre-loading all models and database connections. |

---

## ⚠️ The 3 Python "Gotchas" That Trip Up JS Developers

### Gotcha 1: Empty Container Truthiness
In JavaScript, a check like `if (items)` will run the block even if `items` is an empty array `[]`.
```javascript
// JS/TS
const items = [];
if (items) {
  console.log("This will print! Empty array is truthy in JS.");
}
```
In Python, empty structures (`[]`, `{}`, `set()`, `""`) are evaluated as **falsy**:
```python
# Python
items = []
if items:
    print("This will NOT print in Python!")
```

### Gotcha 2: Mutable Default Parameters
In Python, default arguments are evaluated **once at function definition time**, not at each call. Setting a mutable container (like a list or dict) as a default creates a single shared object across all calls.
```python
# ⚠️ THE GOTCHA
def add_speaker(speakers=[]):  # All calls without arguments share the same list!
    speakers.append("Alice")
    return speakers

print(add_speaker()) # ['Alice']
print(add_speaker()) # ['Alice', 'Alice'] -- Oops! Contaminated.
```
**The pretalx Pattern:** Always use immutable tuples `()` or defaults of `None` to prevent contamination.
```python
# pretalx implementation (submission/domain/submission.py)
def create_submission(*, speakers=(), **kwargs):
    # Immutable default tuple prevents mutations!
    ...
```

### Gotcha 3: The `is` vs `==` Operators
*   `==` checks for **value equality** (similar to `==` or `===` in JS depending on type).
*   `is` checks for **identity equality** (whether they reference the exact same memory address, like checking pointer equality).
```python
# Python
list_a = [1, 2, 3]
list_b = [1, 2, 3]

print(list_a == list_b)  # True (same values)
print(list_a is list_b)  # False (different locations in memory)

x = None
print(x is None)  # True (idiomatic Python: always use 'is' when checking against singletons like None)
```
