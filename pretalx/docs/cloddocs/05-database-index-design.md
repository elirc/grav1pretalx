# Database Index Design

**Supplementary Module — cloddocs 05**  
**Prerequisite Reading:** `docs/upskill/07-interview-prep/02-framework-and-architecture-questions.md` (Q3: N+1 and ORM trade-offs)

---

## The Gap

The existing material correctly identifies the N+1 ORM query problem and mentions that pretalx uses `prefetch_related` and `select_related`. It mentions "compound indexes in `Meta.index_together`" in one sentence. That's where it stops.

Database index design is the single most impactful performance skill for a backend engineer working with relational databases, and it is chronically undertaught. Understanding how to design, read, and debug indexes will have more impact on a production system's performance than almost any application-level optimization.

---

## 1. What an Index Actually Is

An index is a separate, sorted data structure maintained alongside a table, allowing the database to find rows matching a condition without reading the entire table.

Without an index on `submission.event_id`:

```sql
SELECT * FROM submission WHERE event_id = 'pycon2026-uuid';
-- Database reads EVERY row in the submission table, checks each one.
-- If the table has 500,000 rows, it reads 500,000 rows.
-- This is called a Sequential Scan or Full Table Scan.
```

With a B-tree index on `event_id`:

```sql
-- Database traverses the B-tree (O(log n)) to find the matching leaf pages.
-- It reads only the relevant rows.
-- For 500,000 rows, this might read 12-20 rows of index + a handful of data pages.
```

The dominant index type in PostgreSQL (and MySQL, SQLite) is the **B-tree** — a balanced binary tree where leaf nodes contain the indexed values and pointers to the corresponding table rows. Every `CharField`, `IntegerField`, and `ForeignKey` target is a candidate for a B-tree index.

---

## 2. Django Index Declaration — What pretalx Uses

Django declares indexes in the model's `Meta` class:

```python
# src/pretalx/submission/models/submission.py (illustrative)
class Submission(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE)
    state = models.CharField(choices=SubmissionStates.choices)
    code = models.CharField(unique=True)  # unique=True automatically creates an index
    is_featured = models.BooleanField(default=False)
    
    class Meta:
        # Modern Django approach (preferred over deprecated index_together)
        indexes = [
            models.Index(fields=["event", "state"]),     # Composite index
            models.Index(fields=["event", "is_featured"]), 
        ]
```

The `ForeignKey` to `Event` automatically gets a single-column index for JOIN operations. The composite indexes are what make filtered, multi-column queries fast.

---

## 3. Composite Index Column Ordering — The Critical Detail

This is where most engineers go wrong. **Column order in a composite index matters enormously.**

A composite index on `["event", "state"]` can serve:
- ✅ `WHERE event_id = X` (uses the index — leftmost prefix rule)
- ✅ `WHERE event_id = X AND state = 'accepted'` (full composite match)
- ❌ `WHERE state = 'accepted'` (cannot use the index — the leftmost column isn't constrained)

**The Leftmost Prefix Rule:** An index on `(A, B, C)` can efficiently serve queries filtering on `A`, `A+B`, or `A+B+C`. It cannot efficiently serve queries filtering on `B` alone, `C` alone, or `B+C`.

### Practical Impact on pretalx Queries

The organizer submissions list is the most-queried expensive page. A typical query:

```sql
SELECT *
FROM submission
WHERE event_id = 'pycon2026'
  AND state IN ('submitted', 'accepted')
  AND track_id = 'web-track'
ORDER BY created_at DESC;
```

Index design decision:

```python
# Which order?
models.Index(fields=["event", "state", "track"])        # Option A
models.Index(fields=["track", "state", "event"])        # Option B
models.Index(fields=["event", "track", "state"])        # Option C
```

**Option A is correct.** Queries always filter by `event` first (multi-tenant isolation forces this), then typically by `state`, and optionally by `track`. Option A's leftmost prefix matches the query pattern.

Option B is nearly useless — the leftmost column `track` is never filtered alone; it's always paired with an event constraint.

Option C works for `WHERE event AND track`, but not for `WHERE event AND state` without track — a very common filter pattern.

**Rule:** Put the highest-cardinality column that is always present in filters leftmost. In pretalx, `event_id` goes first on almost every composite index.

---

## 4. Index Types Beyond B-tree

PostgreSQL supports several specialized index types that are often the right tool in specific scenarios:

### Partial Indexes

An index built only on rows matching a `WHERE` clause. Much smaller and faster than a full index.

```python
# Index only featured submissions — if is_featured is rarely True,
# this index is tiny and blazing fast for the "featured talks" homepage query.
models.Index(
    fields=["event"],
    condition=models.Q(is_featured=True),
    name="idx_featured_submissions_by_event",
)
```

In raw SQL: `CREATE INDEX idx_featured ON submission (event_id) WHERE is_featured = true;`

**When to use:** A boolean flag, a status column with one dominant value (most submissions are not `accepted`), or a soft-delete `deleted_at IS NULL` pattern.

### GIN (Generalized Inverted Index) — for Arrays and JSON

pretalx stores custom question responses in JSON fields. GIN indexes are required to make JSON field queries efficient:

```python
# For queries like: filter(answers__contains={'question_id': 5})
models.Index(
    fields=["answers"],
    name="idx_submission_answers_gin",
    # Requires explicit creation via RunSQL in a migration:
    # CREATE INDEX ... USING GIN (answers)
)
```

B-tree indexes cannot index inside JSON structure. If you run `filter(data__key=value)` on a `JSONField` without a GIN index, you get a full table scan every time.

### Covering Indexes (INCLUDE columns)

A covering index includes extra columns in the index leaf nodes, meaning the database can satisfy a query entirely from the index without touching the main table at all.

```sql
-- Regular index
CREATE INDEX idx_sub_event_state ON submission (event_id, state);
-- For: SELECT id, title FROM submission WHERE event_id=X AND state='accepted'
-- The DB uses the index to find rows, then fetches title from the heap (2 I/O operations).

-- Covering index
CREATE INDEX idx_sub_event_state_covering ON submission (event_id, state)
    INCLUDE (id, title, code);
-- The DB reads id, title, code directly from the index — never touches the heap.
-- This is dramatically faster for read-heavy list queries.
```

Django supports this via `Index(fields=[...], include=[...])` as of Django 3.2.

---

## 5. Reading EXPLAIN ANALYZE — Your Most Important Debugging Tool

When a query is slow, the first thing you do is run `EXPLAIN ANALYZE` to see what the query planner actually does:

```python
# In the Django shell: python src/manage.py shell
from django.db import connection

queryset = Submission.objects.filter(
    event__slug="pycon2026",
    state="accepted"
).order_by("-created_at")[:50]

# Force the query and get the plan
with connection.cursor() as cursor:
    cursor.execute(
        "EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) " + str(queryset.query)
    )
    for row in cursor.fetchall():
        print(row[0])
```

Key nodes to understand in the output:

| Node Name | Meaning | Good or Bad? |
|:--- |:--- |:--- |
| `Seq Scan` | Full table scan — reads every row | 🔴 Bad on large tables |
| `Index Scan` | Uses an index, but fetches from heap | 🟡 Usually good |
| `Index Only Scan` | Uses a covering index — no heap access | 🟢 Best |
| `Bitmap Heap Scan` | Multiple index ranges merged before heap access | 🟡 Normal for IN/OR queries |
| `Nested Loop` | JOIN via loop — good for small inner sets | 🟡 Depends on row count |
| `Hash Join` | JOIN via hash table — good for large equal sets | 🟢 Usually efficient |
| `Sort` | Explicit sort (ORDER BY with no matching index sort) | 🟡 Can be slow without an index |

The crucial fields in the output:

```
-> Index Scan using idx_sub_event_state on submission  (cost=0.43..8.45 rows=1 width=832)
                                                              ↑           ↑
                                                         Estimated cost  Estimated rows
   (actual time=0.012..0.013 rows=1 loops=1)
                 ↑                ↑
           Actual time ms    Actual rows returned

   Buffers: shared hit=3 read=1
                         ↑          ↑
                   From cache  From disk (expensive!)
```

A large gap between estimated rows and actual rows indicates stale table statistics. Run `ANALYZE submission;` to refresh them.

---

## 6. The Index Trade-off Nobody Talks About

Indexes are not free. Every write (INSERT, UPDATE, DELETE) must also update every index on that table. A table with 15 indexes is fast to read but slow to write.

For pretalx, the submission table has moderate write frequency (submissions arrive during CFP open periods) and very high read frequency (the organizer dashboard is read constantly). This profile justifies more indexes.

The Celery task log table (if it existed as a database-backed log) would be written thousands of times per minute during peak periods. Putting more than one or two indexes on such a table would create write contention. For high-write tables, prefer fewer, more targeted indexes.

**The rule of thumb:** Start with zero indexes beyond the ones Django creates automatically (primary keys, `unique=True` fields, `ForeignKey` targets). Add an index only when you have measured a slow query with `EXPLAIN ANALYZE` and confirmed the index would eliminate the sequential scan. Do not add indexes preemptively "just in case."

---

## Self-Assessment Questions

1. A developer adds this index to the `Review` model: `models.Index(fields=["score", "submission", "user"])`. A common query is `Review.objects.filter(submission=X, user=Y)`. Does this index help that query? Why or why not? What order would be correct?

2. The public speaker directory page loads slowly. `EXPLAIN ANALYZE` shows `Seq Scan on person_speakerprofile (cost=0.00..4240.00 rows=1 width=256) (actual time=1240.32..1240.33 rows=1 loops=1)`. Buffers shows `read=312`. What three specific things does this output tell you, and what is your immediate next diagnostic step?

3. A table has 8 million rows and is written to approximately 2,000 times per second (a high-frequency event log). Someone proposes adding 5 composite indexes to make read queries faster. Explain why this is dangerous and what alternative architectural approach you would propose instead.
