# API Design Trade-offs: REST vs GraphQL vs gRPC

**Supplementary Module — cloddocs 04**  
**Prerequisite Reading:** `docs/upskill/07-interview-prep/02-framework-and-architecture-questions.md`

---

## The Gap

The existing documentation mentions that pretalx has "REST endpoints and serializers" under `src/pretalx/api/`. The interview prep file discusses the request lifecycle and middleware. Neither document asks or answers the question that will actually come up in a senior engineering interview:

**"Why did you build a REST API? Have you considered GraphQL or gRPC? What would change if you switched?"**

This module teaches you to reason through API design paradigms systematically, using pretalx's real API surface as the anchor.

---

## 1. The pretalx REST API — What It Actually Provides

Look at the actual API surface:

```
src/pretalx/api/
├── views/
│   ├── submission.py      # GET, POST, PATCH /api/events/{slug}/submissions/
│   ├── schedule.py        # GET, PATCH /api/events/{slug}/schedules/
│   ├── speaker.py         # GET /api/events/{slug}/speakers/
│   ├── review.py          # GET, POST /api/events/{slug}/reviews/
│   └── event.py           # GET /api/events/
├── serializers/
│   └── ...                # DRF serializers defining field shapes
└── permissions.py         # Token-based auth, event-scoped access
```

Key characteristics:
- **Authentication:** Token-based (API key per event, not per user).
- **Versioning:** None explicit — URL-based (`/api/` prefix).
- **Pagination:** Standard DRF cursor and page-based pagination.
- **Primary consumers:** External conference management tools, mobile apps, data export scripts.

---

## 2. REST — Principles and Pretalx's Adherence

REST (Representational State Transfer) defines six constraints. In practice, the three that matter most for API design:

### Resource-Oriented URLs
Each URL identifies a noun (resource), not a verb (action):

```
✅ REST:    GET    /api/events/pycon2026/submissions/XJ34K/
✅ REST:    PATCH  /api/events/pycon2026/submissions/XJ34K/

❌ Not REST: POST   /api/acceptSubmission?code=XJ34K
❌ Not REST: GET    /api/getSubmissionsByTrack?track=web
```

pretalx's API follows this convention correctly. However, state transitions (accept, reject) are inherently verbs, which creates an inevitable tension in REST design. The common resolution is to treat the state *itself* as the resource:

```
PATCH /api/events/pycon2026/submissions/XJ34K/
Body: { "state": "accepted" }
```

This is what pretalx does. The serializer validates allowed state transitions on the server.

### Statelessness
Every request must contain all information needed to process it. The server stores no session state between requests. pretalx's token auth model satisfies this — the API key in the Authorization header identifies the caller without server-side session lookup.

### Uniform Interface (HTTP Verbs Have Specific Semantics)

| Verb | Semantics | pretalx Use |
|:--- |:--- |:--- |
| `GET` | Read, safe, idempotent | Fetch submissions, schedules |
| `POST` | Create, not idempotent | Create new submission |
| `PUT` | Full replace, idempotent | Replace entire submission record |
| `PATCH` | Partial update, not necessarily idempotent | Update specific fields |
| `DELETE` | Remove, idempotent | Delete a review |

The common interview gotcha: `PATCH` is idempotent only if the underlying operation is (e.g. `{ "state": "accepted" }` is idempotent; `{ "view_count": view_count + 1 }` is not).

### REST's Core Weakness: Over-fetching and Under-fetching

The public schedule page needs to display a list of talks with their rooms, start times, and speaker names — but not abstracts, reviews, or organizer notes.

A REST endpoint for submissions returns the full submission object. The frontend discards 80% of the payload. This is **over-fetching**.

To build a speaker profile page that shows their submissions and their reviews, a REST client must make:
1. `GET /api/speakers/{id}/` — speaker data
2. `GET /api/submissions/?speaker={id}` — speaker's submissions
3. `GET /api/reviews/?submission={code}` for each submission — N extra requests

This is **under-fetching** (also called the N+1 API problem — not to be confused with the N+1 ORM query problem, though they share the same root cause).

---

## 3. GraphQL — The Right Tool When Clients Vary Widely

GraphQL is a query language for APIs where clients specify exactly what data they need in their query.

```graphql
# A mobile app making one network request instead of three
query SpeakerProfile($speakerId: ID!) {
  speaker(id: $speakerId) {
    name
    biography
    submissions {
      code
      title
      state
      room {
        name
      }
    }
  }
}
```

The server returns exactly this shape. No over-fetching. No extra round-trips.

### When GraphQL is the Right Choice for pretalx

GraphQL would be a strong choice for pretalx if:
- **Multiple, diverse clients** (mobile app, public website, partner integration tools) need the same data in different shapes.
- **Rapid frontend iteration** is happening — frontend teams change their data requirements frequently without waiting for backend API changes.
- **The public schedule** must be highly bandwidth-efficient for attendees on conference Wi-Fi.

### When GraphQL is the Wrong Choice for pretalx

GraphQL would be problematic because:
- **Authorization complexity.** pretalx's multi-tenant scoping is carefully enforced at the ORM layer. In GraphQL, if a resolver fetches a submission without the scope middleware active, you get a data leak. Every resolver must independently enforce tenant boundaries, and the complexity multiplies with every field added to the schema.
- **N+1 in resolvers.** The same N+1 problem exists, now inside resolvers instead of views. Every `submission { speakers { name } }` query triggers a separate database call per submission unless you implement **DataLoader** (a batching library). Django has no mature DataLoader equivalent.
- **Introspection risk.** GraphQL's built-in schema introspection lets clients discover all available types and fields. For pretalx's organizer-only fields (like internal review notes), this requires careful field-level authorization — complexity the existing DRF serializers handle more naturally.

**Verdict for pretalx:** REST is the correct choice for pretalx's current use case (external tools, data exports, programmatic access). If pretalx ever built a high-traffic native mobile conference app, GraphQL for the public-facing read API would be worth revisiting.

---

## 4. gRPC — The Right Tool for Internal Service Communication

gRPC is a Remote Procedure Call framework using Protocol Buffers (binary serialization) and HTTP/2. It is designed for service-to-service communication, not browser-to-server APIs.

```protobuf
// submission.proto
service SubmissionService {
  rpc AcceptSubmission (AcceptRequest) returns (AcceptResponse);
  rpc GetSubmission (GetSubmissionRequest) returns (SubmissionResponse);
}

message AcceptRequest {
  string submission_code = 1;
  string actor_user_id = 2;
}
```

| Property | REST | GraphQL | gRPC |
|:--- |:--- |:--- |:--- |
| **Protocol** | HTTP/1.1 | HTTP/1.1 or HTTP/2 | HTTP/2 (required) |
| **Payload format** | JSON (text) | JSON (text) | Protocol Buffers (binary) |
| **Type safety** | None (runtime) | Schema (runtime checked) | Strong (compile-time) |
| **Browser support** | Native | Native | Requires gRPC-Web proxy |
| **Streaming** | Long-polling / SSE | Subscriptions | Bidirectional streaming native |
| **Primary use** | External APIs | Flexible client-driven APIs | Internal microservices |
| **Latency** | Moderate | Moderate | Very low (binary + HTTP/2) |

**When would pretalx use gRPC?** If the architecture was split into microservices — e.g., a separate "Mail Service" that pretalx communicates with to queue and dispatch emails. The internal service-to-service call from the main pretalx process to the Mail Service would be an ideal gRPC use case. JSON serialization overhead and HTTP/1.1 handshake overhead matter when you're making thousands of internal calls per second.

**It would not replace the public REST API**, because gRPC requires either native gRPC clients or a translation proxy — neither of which external conference tool integrators want to deal with.

---

## 5. Webhooks — The Fourth Paradigm

None of the existing documentation mentions webhooks, which is a gap. pretalx has clear natural webhook triggers:

- Submission state changed (accepted, rejected, withdrawn)
- New submission received
- Schedule published
- Review phase opened or closed

A webhook is a user-configured HTTP callback that pretalx calls when an event occurs. The consumer registers a URL; pretalx `POST`s a JSON payload to it when the event fires.

```python
# Conceptual webhook dispatch (not currently in pretalx)
def notify_webhooks(event: str, payload: dict, event_slug: str):
    hooks = WebhookRegistration.objects.filter(
        event__slug=event_slug,
        triggers__contains=event,
    )
    for hook in hooks:
        # Queue a Celery task for each webhook (don't block the request)
        dispatch_webhook.delay(hook.url, hook.secret, payload)
```

The webhook delivery itself should be:
- **Signed** with an HMAC secret to prove origin authenticity.
- **Retried** with exponential backoff on failure.
- **Logged** to a delivery log so the developer can inspect what was sent and what response was received.

Webhooks are the correct pattern for "push" notifications to external integrators — they do not have to poll the REST API repeatedly to detect state changes.

---

## 6. Choosing the Right Paradigm: Decision Tree

```
Does the client need to specify its own data shape?
├── YES → Consider GraphQL (if authorization can be enforced per-resolver)
└── NO  → Continue...

Is this internal service-to-service communication?
├── YES → Use gRPC (low latency, strong types, bidirectional streaming)
└── NO  → Continue...

Does the external system need to react to events asynchronously?
├── YES → Use Webhooks (push model, registered callbacks)
└── NO  → Continue...

Is the client browser-based or external integrator needing a simple, stable interface?
└── YES → Use REST (standard HTTP verbs, JSON, stateless, widely understood)
```

---

## Self-Assessment Questions

1. The pretalx public schedule endpoint `GET /api/events/pycon2026/schedule/` currently returns the full schedule including organizer-only fields like internal notes. A public mobile app only needs room name, talk title, start time, and speaker name. Describe two approaches to solving this within REST, and explain the trade-off between them.

2. A large conference organization company wants to integrate pretalx into their internal tooling. They want to be notified in real time when a submission's state changes, without polling the API every 5 seconds. What API pattern would you recommend, and what security concerns must you address in the implementation?

3. A colleague proposes migrating the pretalx API to GraphQL entirely. You need to present the three strongest arguments against this change, each grounded in a specific technical property of the pretalx system.
