# Observability in Production

**Supplementary Module — cloddocs 02**  
**Prerequisite Reading:** `docs/upskill/05-quality-engineering/03-systematic-debugging.md`

---

## The Gap in the Existing Material

The systematic debugging guide teaches you to use `breakpoint()`, print SQL, and inspect Celery logs when you *already know something is wrong*. It assumes you are sitting at a terminal with local access. 

Production systems are never debugged that way.

In production, you learn something is broken because your monitoring tells you — or worse, because a user tweets about it. Observability is the discipline of designing a system so you can answer the question *"What is my system doing right now?"* without needing to SSH into a server.

---

## 1. The Three Pillars of Observability

These three pillars form the standard mental model used across the industry (originated at Twitter, formalized by Charity Majors at Honeycomb):

```
┌─────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY PILLARS                      │
│                                                             │
│  📋 LOGS                📊 METRICS              🔗 TRACES   │
│  "What happened"        "How much/often"       "Where time  │
│  (structured events,    (counters, gauges,      was spent"  │
│   timestamped)          histograms)             (request    │
│                                                  flow map)  │
│  Useful for:            Useful for:            Useful for:  │
│  Debugging specific     Alerting, dashboards,  Diagnosing   │
│  incidents              SLO tracking           latency      │
└─────────────────────────────────────────────────────────────┘
```

You need all three. A system with only logs but no metrics cannot alert you before things break catastrophically. A system with metrics but no traces cannot tell you *which part* of a slow request is the bottleneck. A system with traces but no logs cannot reconstruct what happened to a specific row of data.

---

## 2. Structured Logging — The Right Way

The existing debugging guide shows this:

```python
logger.info(f"Loaded submission: {submission.code}")
```

This is **unstructured logging**. It works for a developer sitting at a terminal. It is nearly useless in production because you cannot query it. You cannot say "show me all slow requests where event_slug = 'pycon2026' and state = 'accepted'".

**Structured logging** emits machine-readable key-value pairs (usually JSON):

```python
import structlog

log = structlog.get_logger()

# Instead of: logger.info(f"Submission state changed: {submission.code}")
log.info(
    "submission_state_changed",
    submission_code=submission.code,
    event_slug=submission.event.slug,
    old_state=old_state,
    new_state=new_state,
    triggered_by_user=user.pk,
    duration_ms=elapsed_ms,
)
```

Output: `{"event": "submission_state_changed", "submission_code": "XJ34K", "event_slug": "pycon2026", "old_state": "submitted", "new_state": "accepted", ...}`

A log aggregation tool (Loki, Datadog, CloudWatch) can now filter and aggregate across millions of these events in seconds.

### What to Log at Each Pretalx Boundary

| Boundary | What to Log | Log Level |
|:--- |:--- |:--- |
| Middleware: `EventPermissionMiddleware` | `event_resolved`, `event_slug`, `user_id`, `path` | DEBUG |
| State transition: `set_submission_state` | `submission_code`, `old_state`, `new_state`, `actor_user_id` | INFO |
| Mail outbox write: `QueuedMail.save()` | `mail_id`, `recipient_email`, `event_slug`, `template_used` | INFO |
| Celery mail dispatch | `mail_id`, `attempt_number`, `smtp_response_code`, `latency_ms` | INFO or WARNING on retry |
| Any uncaught exception | Full traceback + request context | ERROR |
| Scope boundary violation: `ScopeError` | `query_attempted`, `active_scope`, `request_path` | CRITICAL |

---

## 3. Metrics — The Language of SLOs

Metrics are aggregated numerical measurements over time. They tell you not what happened to one request but how the system is behaving overall.

### The Four Golden Signals (from Google SRE)

| Signal | Description | pretalx Example |
|:--- |:--- |:--- |
| **Latency** | How long requests take (p50, p95, p99 percentiles) | `http_request_duration_seconds{endpoint="/cfp/submit"}` |
| **Traffic** | How many requests per second the system is handling | `http_requests_total{event_slug="pycon2026"}` |
| **Errors** | Rate of failed requests (4xx client, 5xx server) | `http_errors_total{status="500"}` |
| **Saturation** | How full your most constrained resource is (DB connections, queue depth) | `celery_queue_depth{queue="default"}` |

### Critical pretalx-Specific Metrics to Instrument

```python
# Example: Prometheus-style metric instrumentation in Django middleware
from prometheus_client import Counter, Histogram

SUBMISSION_STATE_CHANGES = Counter(
    "pretalx_submission_state_changes_total",
    "Total submission state transitions",
    labelnames=["event_slug", "from_state", "to_state"],
)

MAIL_DISPATCH_LATENCY = Histogram(
    "pretalx_mail_dispatch_seconds",
    "Time taken to dispatch queued mail",
    buckets=[0.1, 0.5, 1.0, 5.0, 30.0, 60.0],
)

CELERY_QUEUE_DEPTH = Gauge(
    "pretalx_celery_queue_depth",
    "Current number of tasks in each queue",
    labelnames=["queue_name"],
)
```

### Alerting Strategy for a Production Event Platform

The dangerous periods for pretalx are:
1. **CFP deadline day** — massive submission traffic spike
2. **Acceptance announcement day** — massive outbound email volume
3. **Conference day itself** — public schedule read traffic spike

Pre-set alerts for each:

| Alert | Condition | Severity |
|:--- |:--- |:--- |
| Celery queue backing up | `queue_depth > 500 for 5min` | WARNING |
| Mail dispatch failure rate | `mail_failures / mail_dispatches > 5%` | CRITICAL |
| DB connection pool exhaustion | `pg_connections_active / pg_connections_max > 90%` | CRITICAL |
| Public schedule response time | `p99 latency > 2s for 3min` | WARNING |
| ScopeError rate elevated | `scope_errors_total rate > 1/min` | CRITICAL (data leak risk) |

---

## 4. Distributed Tracing — Following a Request Across Boundaries

A trace represents the complete lifecycle of a single request as it flows through multiple components. Each component adds a **span** — a timed segment of work.

For pretalx, a submission acceptance request generates a trace like this:

```
Trace: POST /orga/pycon2026/submissions/XJ34K/accept/  [Total: 145ms]
│
├── Span: EventPermissionMiddleware.resolve_event           [2ms]
│     └── Tag: event_slug=pycon2026
│
├── Span: SubmissionAcceptView.post                         [130ms]
│     ├── Span: submission.accept() → domain service        [125ms]
│     │     ├── Span: DB: UPDATE submission SET state       [8ms]
│     │     │     └── Tag: db.statement, db.rows_affected=1
│     │     ├── Span: DB: UPDATE talk_slots                  [12ms]
│     │     ├── Span: DB: INSERT queuedmail                  [5ms]
│     │     └── Span: transaction.atomic commit              [100ms] ← SLOW
│     └── Span: messages.success()                          [0ms]
│
└── Span: Response serialized and returned                  [1ms]
```

This trace immediately reveals that the database commit is taking 100ms — the transaction is doing too much work inside the atomic block, or the database is under index pressure. No amount of log searching would surface this as clearly.

**Tools:** OpenTelemetry (vendor-neutral instrumentation), Jaeger (open-source trace collector), Datadog APM, or Honeycomb.

---

## 5. The Production Incident Checklist

When something breaks in production and you have no terminal access, work through these layers in order:

1. **Check error rate metric.** Is the p99 latency spiking? Is the 5xx rate elevated? Narrow to which endpoints.
2. **Check Celery queue depth metric.** If it's climbing, workers are not keeping up or have crashed.
3. **Search structured logs for ERROR level entries** in the last 10 minutes. Filter to the affected endpoint or event slug.
4. **Pull the trace** for a specific failing request ID to pinpoint which span is slow or failing.
5. **Check database connection metrics.** Pool exhaustion causes cascading timeouts that look like application bugs.
6. **Check Redis availability.** If Redis is down, session middleware fails silently, Celery cannot accept tasks, and cache misses flood the database.

---

## 6. JS/TS Equivalent Tooling

| Python / Django | JavaScript / Node.js |
|:--- |:--- |
| `structlog` | `pino`, `winston` (structured loggers) |
| `prometheus_client` | `prom-client` |
| `opentelemetry-sdk` | `@opentelemetry/sdk-node` |
| Django Debug Toolbar (dev) | `clinic.js`, `0x` flame graphs |
| Celery Flower dashboard | BullMQ Board |
| Sentry (error tracking) | Sentry (same library, JS SDK) |

---

## Self-Assessment Questions

1. A user reports that their submission disappeared after they clicked "Submit". Your metrics show the HTTP 200 rate is normal. Your logs show a `transaction.atomic` block was entered but no `QueuedMail INSERT` appears. What does this tell you about where the failure occurred?

2. You are approaching a CFP deadline and the Celery queue depth metric starts climbing from 50 to 800 over 10 minutes. List three distinct root causes this metric could indicate, and explain how you would distinguish between them using the other two observability pillars.

3. Why is `logger.info(f"Submission {s.code} accepted by {user}")` inferior to structured logging for production incident investigation? Be specific about what queries become impossible.
