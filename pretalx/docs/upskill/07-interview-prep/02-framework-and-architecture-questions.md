# Framework and Architecture Interview Prep

This module contains enterprise backend architectural questions, structured to show how you can use **pretalx's** production implementation to stand out in interviews.

---

## Q1: Walk me through the request/response lifecycle in your web framework.
*   **What the interviewer is really asking:** Do you understand the mechanics of the web framework under the hood, or do you just write view logic?
*   **Where to find it in this repo:** [`src/pretalx/settings.py#L476-L491`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/settings.py#L476-L491) (the registered middleware stack).

### Answer Guide
*   **Junior:** "The request comes in, goes through urls.py router to find the view, the view runs database queries and returns an HTML template response."
*   **Mid-Level:** "In Django, a request enters the middleware pipeline sequentially. It passes through security checking, then static file resolution (Whitenoise), session lookup, authentication (attaching `request.user`), and routing matching inside `urls.py`. Once matched, the view executes, runs models queries, renders templates, and the response travels back up the middleware stack in reverse."
*   **Senior:** "In pretalx, the middleware stack coordinates multi-tenant isolation and security:
    1.  `SecurityMiddleware` secures request protocols.
    2.  `PretalxWhiteNoiseMiddleware` intercepts static asset requests.
    3.  `SessionMiddleware` and `AuthenticationMiddleware` load sessions and authenticate user accounts.
    4.  `MultiDomainMiddleware` resolves customized domain mappings to event contexts.
    5.  `EventPermissionMiddleware` extracts the event slug, binds `request.event`, sets the locale, and opens a `django-scopes` tenant-fence to guarantee that no subsequent database query can query cross-event records.
    6.  `CsrfViewMiddleware` filters out CSRF exploits.
    7.  Finally, the request hits the matched view class, renders the template or returns API JSON, and returns."

---

## Q2: How do you structure a complex web application to enforce boundaries?
*   **What the interviewer is really asking:** Do you know how to organize directories and separation of concerns beyond a basic MVC framework layout?
*   **Where to find it in this repo:** [`src/pretalx/`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/) root directories.

### Answer Guide
*   **Junior:** "I separate files into standard folders like models, views, controllers, and tests."
*   **Mid-Level:** "In pretalx, we separate the codebase into highly focused modular Django sub-applications by concern (`cfp` handles speaker portal, `agenda` handles public schedule, `orga` handles organizer tools, `submission` handles proposals). Each sub-application encapsulates its own views, models, forms, templates, and signals, keeping code highly modular."
*   **Senior:** "Beyond standard sub-application folders, pretalx decouples volatile business domain rules from HTTP adapters by isolating them inside the `domain/` directory (e.g. [`src/pretalx/submission/domain/submission.py`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/domain/submission.py)). The view controller only extracts HTTP inputs and delegates mutations to this domain layer. This allows CLI commands, REST APIs, and HTTP templates to share identical state-transition contracts, reducing code duplication and bug surfaces."

---

## Q3: What are the trade-offs of using an ORM, and how do you prevent performance bottlenecks?
*   **What the interviewer is really asking:** Do you understand the N+1 query problem, lazy loading, and database index patterns?
*   **Where to find it in this repo:** [`src/pretalx/submission/models/submission.py#L46-L53`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/models/submission.py#L46-L53) (uses `prefetch_related` in `SubmissionQuerySet`).

### Answer Guide
*   **Junior:** "ORMs make querying databases easy using standard code, but they can run many queries if you are not careful."
*   **Mid-Level:** "Django ORM uses lazy evaluation: constructing querysets does not hit the database until they are iterated or serialized. The main tradeoff is the N+1 query issue: if you load submissions and loop over speakers without prefetching, Django runs one query to load the submissions, and then N extra queries to fetch each submission's speaker. We solve this by eagerly loading relations using `select_related` for ForeignKeys and `prefetch_related` for ManyToMany fields."
*   **Senior:** "In pretalx, we optimize this directly inside custom model Managers. We define a custom QuerySet subclass `SubmissionQuerySet` with method `with_sorted_speakers` executing prefetching cleanly. When listing proposals, we call this method, reducing database roundtrips from N+1 down to exactly 2. Furthermore, we monitor hot execution paths using Django's Debug Toolbar or automated pytest profiling, and define database-level compound indexes in model `Meta.index_together` for optimized filtering on slugs."

---

## Q4: How do you handle database migrations safely in a production environment?
*   **What the interviewer is really asking:** Do you know how to perform zero-downtime schema changes, split columns, or rollback migrations under load?
*   **Where to find it in this repo:** [`src/pretalx/submission/migrations/`](file:///c:/Users/Owner/Desktop/grav1pretalx/pretalx/src/pretalx/submission/migrations/) folder.

### Answer Guide
*   **Junior:** "I run `makemigrations` and then `migrate` on production."
*   **Mid-Level:** "We use Django's migration engine to generate versioned migration scripts. In production, we run migrations during deployment pipelines. To do this safely, we avoid destructive migrations (like dropping a column immediately). Instead, we split it into multiple steps: deploy code that stops using the column, migrate the database to make it nullable, and only drop it in a subsequent release."
*   **Senior:** "Safe migrations require an strict, multi-step deploy pattern:
    1.  **Additions:** Adding a column is safe. We make sure it is nullable or has a default to prevent locking large tables.
    2.  **Modifications:** For renaming or typing changes, we employ a write-both schema. We add the new column, update application writes to write to both columns (using signals or model save overrides), run a script to backfill historical rows, switch read routes to the new column, and finally delete the old one.
    3.  **Locks:** We avoid DDL locks on large tables by using index building concurrently where applicable, and run migrations within isolated transaction boundaries to permit rollback on failure."
