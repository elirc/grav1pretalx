# Failure Isolation & Resilience Patterns

**Supplementary Module — cloddocs 06**  
**Prerequisite Reading:** `docs/upskill/03-architecture-and-patterns/05-pattern-catalog.md` (all four patterns), `cloddocs/01-cap-theorem-and-consistency.md`

---

## The Gap

The existing pattern catalog correctly describes what can fail (e.g., "If a sub-task fails silently, the state is updated but the side-effect fails, creating a state mismatch"). It describes *that* things fail. It does not teach you how to *architect for failure* — how to design systems that remain partially operational when individual components break, recover gracefully when they come back, and do not cascade a single component failure into a full system outage.

This is sometimes called **defense-in-depth for reliability**, as opposed to defense-in-depth for security.

---

## 1. Failure Modes Specific to pretalx's Architecture

Before prescribing solutions, identify the actual failure modes:

```
Failure Mode Map for pretalx in Production

Component         │ When It Fails                     │ Downstream Impact
──────────────────┼───────────────────────────────────┼──────────────────────────────────
PostgreSQL        │ Disk full, connection pool         │ ALL features fail — the entire
                  │ exhausted, primary DB crash        │ application is down
──────────────────┼───────────────────────────────────┼──────────────────────────────────
Redis             │ Memory exhausted, pod restart,     │ Sessions fail (all users logged
                  │ network partition                  │ out), Celery task queue stops
                  │                                   │ accepting tasks, cache misses
                  │                                   │ flood the database
──────────────────┼───────────────────────────────────┼──────────────────────────────────
Celery Workers    │ Workers crash or hang,             │ Emails not sent, background
                  │ task queue backs up                │ reports not generated — but
                  │                                   │ core submission flow continues
──────────────────┼───────────────────────────────────┼──────────────────────────────────
SMTP / Mail API   │ Mail server rejects connections,   │ Email delivery stops. The
(Resend, SES)     │ rate limit exceeded               │ QueuedMail table grows but no
                  │                                   │ permanent data loss occurs
──────────────────┼───────────────────────────────────┼──────────────────────────────────
Read Replica DB   │ Replication lag, replica crash     │ Public agenda pages may serve
                  │                                   │ stale or unavailable data
──────────────────┼───────────────────────────────────┼──────────────────────────────────
CDN / Edge Cache  │ Cache invalidation failure,        │ Stale schedule pages served to
                  │ origin overloaded                 │ attendees; potential for minutes
                  │                                   │ of incorrect data being visible
```

Notice that **PostgreSQL failure is catastrophic** (total outage) while **Celery failure is degraded** (core features continue, notifications are delayed). This distinction drives every resilience architectural decision.

---

## 2. Circuit Breakers — Stopping the Cascade

A **circuit breaker** is a proxy around an external call that monitors failure rates and "trips open" when failures exceed a threshold, immediately returning a fallback response instead of waiting for a slow or failed dependency.

The name is an analogy from electrical engineering: when too much current flows, the breaker trips to prevent fire damage. When it trips, the rest of the circuit (your application) continues operating.

```
States:

CLOSED (normal)    ────────────────►  OPEN (tripped)
  │                 failure rate       │
  │                 > threshold        │  timeout period elapses
  │                                    │
  │  ◄─────────────────────────────────┘
  │  success rate returns to normal
HALF-OPEN (testing)
```

### Concrete pretalx Application: The SMTP Circuit Breaker

```python
import pybreaker  # Python circuit breaker library

smtp_breaker = pybreaker.CircuitBreaker(
    fail_max=5,          # Trip after 5 consecutive failures
    reset_timeout=60,    # Wait 60 seconds before retrying (HALF-OPEN)
)

@smtp_breaker
def dispatch_queued_mail(mail_id: int):
    mail = QueuedMail.objects.get(pk=mail_id)
    send_via_smtp(mail)  # This call to the SMTP server is protected

# Usage in Celery worker:
def process_mail_queue():
    pending = QueuedMail.objects.filter(sent=False).order_by("created_at")[:100]
    for mail in pending:
        try:
            dispatch_queued_mail(mail.pk)
        except pybreaker.CircuitBreakerError:
            # SMTP is down. Stop processing immediately.
            # All remaining mails stay safely in the QueuedMail table.
            logger.warning("SMTP circuit open — halting mail dispatch")
            break
        except Exception as e:
            logger.error("Mail dispatch failed", mail_id=mail.pk, error=str(e))
```

**Without the circuit breaker:** Each Celery task tries to connect to SMTP, waits 30 seconds for the connection timeout, fails, and retries. During an SMTP outage, your Celery worker pool is entirely consumed waiting for TCP timeouts — leaving no capacity for other background tasks.

**With the circuit breaker:** After 5 failures, the breaker trips. Subsequent calls fail immediately (no timeout wait). Worker capacity is preserved for other tasks. Mail stays in the outbox until SMTP recovers.

---

## 3. Retry Strategies — Exponential Backoff with Jitter

When a transient failure occurs (a brief network hiccup, a momentary database overload), retrying is the right strategy. But naive retries cause thundering herds.

**The thundering herd problem:** 500 Celery workers all fail at the same moment because the database restarted. They all retry after exactly 5 seconds. The database comes back up and immediately receives 500 simultaneous queries — crashing it again.

**Exponential backoff with jitter** solves this by:
1. Waiting exponentially longer between attempts: 1s, 2s, 4s, 8s, 16s...
2. Adding random jitter to desynchronize retries across workers.

```python
# Celery task retry configuration — pretalx uses this pattern
@shared_task(
    bind=True,
    max_retries=5,
    default_retry_delay=60,      # Base retry delay
)
def send_queued_mail(self, mail_id: int):
    try:
        mail = QueuedMail.objects.get(pk=mail_id)
        send_via_smtp(mail)
    except SMTPException as exc:
        # Exponential backoff with jitter
        delay = (2 ** self.request.retries) * 60  # 60s, 120s, 240s, 480s, 960s
        jitter = random.uniform(0, delay * 0.2)   # ±20% randomization
        raise self.retry(exc=exc, countdown=delay + jitter)
    except QueuedMail.DoesNotExist:
        # Permanent failure — do not retry
        logger.error("Mail not found", mail_id=mail_id)
```

**Key parameters to tune:**

| Parameter | Too Low | Too High | Goldilocks |
|:--- |:--- |:--- |:--- |
| `max_retries` | Gives up too early on transient failures | Never stops on permanent failures | 3–7 for external I/O calls |
| Base delay | Thundering herd, overwhelms recovering service | Users wait too long for notifications | 30–120s for email |
| Jitter range | Still synchronizes retries | Adds too much unpredictability | 10–25% of base delay |

---

## 4. Dead-Letter Queues — What Happens to Permanently Failed Tasks

After exhausting all retries, where does the task go?

Without a dead-letter queue (DLQ), it disappears. You have no record that it failed, no way to investigate why, and no way to manually replay it after fixing the underlying problem.

```python
# In Celery configuration (src/pretalx/celery_app.py conceptual extension):
app.conf.task_routes = {
    "pretalx.mail.tasks.send_queued_mail": {
        "queue": "default",
        "dead_letter_exchange": "pretalx.dlq",  # Failed tasks route here
    }
}
```

After all retries are exhausted, the task message goes to the dead-letter queue. A monitoring alert fires when DLQ depth grows above zero. A developer can then:

1. Investigate the failure (inspect the task arguments and error).
2. Fix the underlying code or configuration issue.
3. Replay the failed tasks from the DLQ using a management command.

Without this, **permanent failures silently disappear** — the speaker is never notified, and nobody knows.

---

## 5. Bulkheads — Preventing Resource Exhaustion Cascades

A **bulkhead** is an isolation boundary that prevents a failure in one part of the system from consuming all resources and starving other parts. The name comes from ship compartments: a hull breach in one compartment doesn't sink the entire ship.

In pretalx's Celery architecture, the straightforward bulkhead is **separate queues with dedicated worker pools**:

```python
# Without bulkheads: One queue, one pool
# If bulk acceptance emails (500 tasks) flood the queue,
# critical tasks (password resets, invitation emails) wait in line.

# With bulkheads: Separate queues with dedicated workers
CELERY_TASK_ROUTES = {
    "pretalx.mail.tasks.send_invitation":    {"queue": "high-priority"},
    "pretalx.mail.tasks.send_reset":          {"queue": "high-priority"},
    "pretalx.mail.tasks.send_acceptance":     {"queue": "default"},
    "pretalx.reports.tasks.generate_export":  {"queue": "bulk"},
}

# Worker process 1 (2 workers): consumes ONLY "high-priority"
# Worker process 2 (4 workers): consumes ONLY "default"  
# Worker process 3 (2 workers): consumes ONLY "bulk"
```

If the bulk export job hangs all bulk workers, high-priority invitation emails are completely unaffected.

In the database layer, the equivalent bulkhead is connection pool partitioning: a separate `DATABASE_ROUTERS` configuration in Django that routes read queries to read replicas and write queries to the primary — ensuring that a surge of public read traffic does not consume all write connections.

---

## 6. Graceful Degradation — The Hierarchy of Features

Not all features matter equally. Under load or partial failure, a well-architected system continues serving its most important functionality while shedding less critical features.

For pretalx, the degradation hierarchy from most to least critical:

```
Tier 1 — Must Never Fail:
  ✅ Speakers can submit proposals (before CFP deadline)
  ✅ Organizers can accept/reject submissions
  
Tier 2 — Degraded is Acceptable:
  ⚠️  Email notifications delayed (QueuedMail outbox holds them safely)
  ⚠️  Schedule export/CSV downloads delayed
  ⚠️  PDF badge generation delayed

Tier 3 — Can Fail Silently with Error Message:
  ❌  Search and filter on organizer dashboard
  ❌  Public speaker directory sorted view
  ❌  Schedule editor drag-and-drop (read-only fallback acceptable)
```

To implement this, expensive Tier 3 operations can be wrapped in try/except with fallback behavior:

```python
def organizer_submission_list(request):
    try:
        queryset = get_filtered_submissions(request)  # Complex filter + search
    except Exception:
        # Cache or database issue — serve unfiltered list
        queryset = Submission.objects.filter(event=request.event)
        messages.warning(request, "Search and filters are temporarily unavailable.")
    return render(request, "orga/submissions/list.html", {"submissions": queryset})
```

---

## 7. The SLO Triangle — Choosing Your Reliability Target

A **Service Level Objective (SLO)** is the target reliability you commit to. For pretalx:

| Feature | Target SLO | Justification |
|:--- |:--- |:--- |
| CFP submission endpoint | 99.9% availability | A 1-hour outage during deadline day could cause speaker data loss |
| Email delivery | 99% within 5 minutes | 1% of delayed emails is tolerable; none are lost (outbox) |
| Public schedule pages | 99.5% availability, p99 < 2s | Conference attendees need this during the event |
| Organizer dashboard | 99% availability | Used only by organizers, outages are annoying but not critical |
| Schedule PDF export | 95% availability | Batch operation; failure is retryable |

The SLO directly drives infrastructure investment: a 99.9% availability target for CFP submissions means the database holding submissions must have automatic failover (streaming replication + pg_auto_failover or RDS Multi-AZ), while the PDF export service can run on a single preemptible instance.

---

## Self-Assessment Questions

1. During a major conference acceptance announcement, the SMTP provider returns HTTP 429 (Too Many Requests) after 200 emails are dispatched. Celery retries them. The next retry wave hits the rate limit again immediately. Trace through the circuit breaker + exponential backoff + DLQ pipeline and describe the exact sequence of events that should occur to handle this gracefully.

2. A single Celery worker gets stuck processing a malformed PDF export task that never terminates (infinite loop). Without bulkheads, what is the risk? How does the queue partitioning strategy prevent this from affecting email delivery?

3. Design the graceful degradation behavior for the following scenario: the PostgreSQL read replica falls behind the primary by 30 minutes due to a replication issue. Which features in pretalx can continue operating from the replica (with stale data), which must be routed to the primary, and which should be temporarily disabled? Justify each decision.
