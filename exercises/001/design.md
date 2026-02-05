# Design: Stripe Webhook Delivery Infrastructure

One-line summary: We're building a notification system (with a few features) that will notify clients of their stripe events (e.g invoice paid)

NOTE: I'm not seeing any clear sub-systems to break the system down into.

NOTE: Let's talk about what entities we have.

## Entities

1. Events (id, clientId, clientAppId, title, status (OPEN, DELIVERED, ATTEMPTING, FAILED), last_status_at, n_failed_attempts, metadata (varies for different event types))
3. ClientApplications (id, clientId, publicEndpoint, sharedSecret (to validate we are the communicator), eventType (list of strings))
4. WebhookAttempts (id (idempotency id, primary key), eventId, attemptedAtTime, httpResponseCode, failureMessage)

NOTE: We need to get these events to the client applications.

## Components

1. A relational DB, we'll use Postgres. All these entities will be tables in that DB. This design assumes the Events are written directly into the DB by some other internal system.
2. A task queue watching events whose status = OPEN OR (FAILED && n_failed_attempts < 3)  (their "candidate events"). We can extend this to support backoff by adding a lastRetriedAt field to the event table and adding something like `AND lastRetriedAt < now() - (10 seconds * n_failed_attempts)` to increase the timeout as n_failed_attempts scales.
We'll need a bunch of workers. They will pop off batches of candidate events using a "SELECT * FOR UPDATE" and update the events with an ATTEMPTING status. Then, for each of those requests, we'll grab the asociated clientApplication records from the db, filter down to those with a matching event type, for each of those client applications, we'll send an https request to its publicEndpoint using the sharedSecret as our Authorization header. Clients will get that shared secret when they initially create their client application on our system. To handle timeouts, we'll timeout after 1 second. And we won't trigger an error if the request fails.
We'll write the result of that operation to the WebhookAttempts table. To handle idempotency, we'll provide an `Idempotency-Key` header with the webhook attempt id. We'll handle client service errors by writing the specific errors to the webhook attempt failureMessage, or if its an internal error we'll write an internal record note in the failureMessage.
If the webhook attempt failed, we'll still write the WebhookAttemptsRecord and update the event record to increment n_failed_attempts.
3. A cronjob to purge events older than 30 days (DELETE those with attemptedAtTime > now() + 30 days).
4. An API. We'll need to support user access to all their events and webhook delivery attempts.
 - GET /events (filter by client id in the JWT token, assume we'll use JWT auth)
 - GET /webhookAttempts (filter by client id in the JWT token)
We'll also need to support event replay
 - POST /events/:id/replay => Sets the event status to OPEN and n_failed_attempts to 0.

## To address the non-functional requirements

1. To address the event ingestion, 500k new events added per second will fry the DB. We'll need sharding. We can shard based on client ID, but that does risk hotbeds of high-volume clients. I'm not sure of a better strategy that also allows for simple querying of those events using client id as the filter.
2. To address the registered endpoint fanout of 10M endpoints and 2M active merchants, those amounts can both safely live in a postgres table each. They're infrequently changing data as well. We could add indexes as well to help performance (e.g ClientApplications.clientId). We're definitely querying these tables constantly in our webhook attempt workers, so we could potentially add a cache for the registered endpoints per-merchant. And it gets invalidated when that user adjusts their configuration or adds/removes an endpoint.
3. To address delivery latency, we would need a lot of worker pods and concurrency built into each to ensure timeouts are not adding time sequentially. Let's assume each worker will process 2,000 events per 2 seconds (DB interactions + 1 second timeout for each request which are all concurrent). That will average 1,000 events per second. That means we'll need 5,000 workers ensure we're processing all events within 1 second => 10,000 workers to process all events within 0.5 seconds (500ms) on average (which will be our p50 time).
4. We meet our event retention by keeping our events data for 30 days, then purging with a cronjob.
5. A highly available DB should be enough to ensure 4 9s of "event ingestion" availability is met. To meet 3 9s for the delivery pipeline, just addressing failure modes should be enough. The main failure mode is a worker locking a batch of events (marking them as ATTEMPTING) and then experiencing a failure. We can remediate this with a cronjob running every few seconds to mark jobs in ATTEMPTING who have been in ATTEMPTING for more than a few seconds (check last_status_at) as OPEN again. This failure mode would cause a violation of our delivery latency requirement of 500ms.
6. The retry window requirement is supported through retryable events being automatically elegible to be picked up by other workers and replayable for the 30 day lifespan of the event.


Time elapsed: 90 minutes.

---

## Review

**Score: 5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Understanding | 5.5/10 | Core entities identified, but misses per-tenant isolation, cryptographic signatures, event ordering, and abuse protection requirements |
| Architecture Quality | 5/10 | Coherent structure, but polling-based `SELECT FOR UPDATE` cannot handle 500k events/sec; no message queue or streaming layer |
| Tradeoff Reasoning | 4.5/10 | Acknowledges sharding hotspots but doesn't explore alternatives; no discussion of pull vs push, queue-per-tenant vs shared queues |
| Failure Handling | 5.5/10 | Good: identifies stuck-job recovery. Missing: circuit breakers for bad endpoints, tenant isolation from slow endpoints, database unavailability |
| Operational Concerns | 3.5/10 | No monitoring, alerting, metrics, or deployment strategy. Would be difficult to debug in production |
| Communication | 6.5/10 | Readable structure with inline reasoning. Dense in places, no diagrams |

### Feedback

**Critical scale issue**: The design relies on workers polling Postgres with `SELECT * FOR UPDATE`. At 500k events/second, this creates:
- Massive lock contention on the events table
- Database becoming the bottleneck (not the external HTTP calls)
- No backpressure mechanism when downstream is slow

A message queue (Kafka, SQS) between event ingestion and delivery workers would decouple these concerns and handle the throughput requirement.

**Per-tenant isolation is missing**: The constraint "must not allow one customer's failing/slow endpoint to impact delivery to other customers" is not addressed. Currently, a slow endpoint (using your 1s timeout) consumes worker capacity that could serve healthy endpoints. Consider:
- Per-tenant queues or virtual queues with fairness scheduling
- Circuit breakers that fast-fail known-bad endpoints
- Separate pools for retry traffic vs fresh traffic

**Abuse protection not addressed**: What happens when a merchant endpoint is:
- Intentionally slow (resource exhaustion attack)?
- Responding with 500s to trigger infinite retries?
- A redirect loop?

Need rate limiting per endpoint, maximum retry caps, and endpoint health scoring.

**Cryptographic signatures not mentioned**: The problem explicitly requires customers to verify webhook authenticity. The `sharedSecret` in the Authorization header is mentioned but the standard pattern (HMAC signature of payload + timestamp) isn't described.

**Dead-letter queue (DLQ) concept is weak**: Events with status=FAILED after 3 attempts just... sit there. The requirement asks for "surface persistently failing events for manual inspection and replay." Need:
- Clear DLQ concept
- Customer notification when events fail permanently
- Dashboard/API to see failed events specifically

**Operational blindspot**: No discussion of:
- What metrics to track (delivery latency p50/p99, success rate per endpoint, retry rate)
- Alerting thresholds
- How on-call would debug "customer X says they're missing events"
- Deployment strategy for the workers

### Revision Suggestions

1. **Add a message queue layer**: Events → Kafka/SQS → Workers. This handles 500k/sec ingestion, provides durability, and decouples ingestion availability from delivery availability.

2. **Design tenant isolation**: Either partition queues by tenant, or implement weighted fair queuing with per-tenant rate limits. Add circuit breakers that track endpoint health and skip unhealthy endpoints temporarily.

3. **Add explicit DLQ handling**: After N retries over 72 hours, move event to a dead-letter state. Emit a webhook to a customer-configured "failure notification" endpoint. Provide `GET /events?status=dead_letter` in the API.

4. **Describe the signature scheme**: `Stripe-Signature: t=timestamp,v1=hmac_sha256(timestamp.payload, secret)`. Include timestamp to prevent replay attacks.

5. **Add an observability section**: Define key metrics (delivery_latency_seconds histogram, delivery_success_total counter by status code, retry_count histogram). Describe what dashboards operators need.

6. **Address abuse vectors**: Max 1s timeout (good), but also add: max redirects=0, max response body read=1KB, endpoint health scores that trigger circuit breakers, per-endpoint rate limits on retries.

---

## Detailed Critique

### On Structure and Clarity of Thought

**What worked:**
- The `NOTE:` markers show you thinking out loud, which is good in an interview setting
- Starting with entities before components is a reasonable approach
- The NFR section maps requirements to solutions explicitly—this is a good habit

**What didn't work:**

Your opening statement "I'm not seeing any clear sub-systems" is a red flag. This problem *screams* for sub-system decomposition:
- **Ingestion**: accepting events from internal Stripe systems
- **Routing**: determining which endpoints need to receive which events
- **Delivery**: making the HTTP calls
- **Retry management**: scheduling and executing retries with backoff
- **Observability**: tracking delivery status for customers and operators
- **Dead-letter handling**: managing permanently failed events

You collapsed all of these into "workers that do everything." This made your design monolithic and harder to reason about. In an interview, explicitly naming sub-systems shows architectural thinking even if you then choose to combine some for simplicity.

**The entity-first approach hurt you here.** You defined Events, ClientApplications, WebhookAttempts—all reasonable—but then jumped straight to "how do I query these tables?" This locked you into a database-centric mental model. The better question would have been: "What are the data flows and what are the latency/throughput constraints on each?" That would have led you to recognize that event ingestion (500k/sec, must not lose data) and event delivery (external HTTP, unpredictable latency) have fundamentally different characteristics requiring different infrastructure.

**Dense paragraphs obscure your reasoning.** Component #2 is a 200-word wall of text covering: polling logic, worker behavior, endpoint lookup, HTTP calls, timeout handling, idempotency, error handling, and retry logic. An interviewer can't easily verify your thinking or probe specific decisions. Break these apart.

---

### Specific Points You Made—Examined

**1. "SELECT * FOR UPDATE" for the task queue**

You correctly identified you need workers to claim events. But `SELECT FOR UPDATE` at 500k events/sec means:
- Every worker competes for row locks
- Lock wait time becomes your dominant latency
- A single slow transaction blocks others
- Database CPU spent on lock management, not query execution

The math: 500k events/sec with 10k workers = 50 events/worker/sec. If each SELECT FOR UPDATE touches even 10 rows with a 5ms lock hold time, you're at 50% lock contention. This doesn't scale.

**2. "We can extend this to support backoff by adding lastRetriedAt"**

This is the right idea but wrong implementation. Polling for `lastRetriedAt < now() - backoff` means you're constantly scanning rows that aren't ready yet. At scale, you'd be reading millions of rows to find the few thousand that are due for retry.

Better: schedule retries as future-dated messages in a queue, or use a dedicated scheduler (like a time-series database or sorted set) that efficiently answers "what's due now?"

**3. "2,000 events per 2 seconds... 10,000 workers"**

I appreciate the back-of-envelope math—this is good practice. But the model is flawed:
- You assumed 1 second timeout per request, but requests are concurrent within a worker
- You didn't account for the database round-trips (SELECT FOR UPDATE, UPDATE status, INSERT attempt, UPDATE on completion)—these are serial and add latency
- 10,000 workers each hitting Postgres is 10,000 connections; Postgres connection limits are typically 100-500 without pooling, and even with PgBouncer, 10k concurrent transactions is extreme

The actual bottleneck is database operations, not HTTP calls.

**4. "Sharding by client ID risks hotbeds"**

You identified the problem but gave up: "I'm not sure of a better strategy." Alternatives you could have proposed:
- **Consistent hashing on event ID**: spreads load regardless of client size
- **Hybrid**: shard by client ID but rate-limit per-client within each shard
- **Tiered architecture**: large clients get dedicated shards, small clients share

Acknowledging a problem and moving on without a solution is worse than proposing an imperfect solution you can discuss.

**5. "Cache for registered endpoints per-merchant, invalidated on config change"**

Good instinct. But you didn't think through the implications:
- How does cache invalidation propagate to all workers?
- What's the consistency model? (stale cache = events sent to old endpoint)
- Is this actually necessary? 10M endpoints at ~500 bytes each = 5GB. Fits in memory on a single machine, could be a dedicated lookup service.

**6. "A highly available DB should be enough for 4 9s"**

This is hand-waving. "Highly available DB" isn't a solution—it's a requirement. What does HA mean here?
- Primary-replica with automatic failover? (failover takes 10-30 seconds, violates 4 9s)
- Multi-region active-active? (complex, write conflicts)
- Synchronous replication? (latency cost)

4 nines = 52 minutes downtime/year. A single failover event can consume half your budget.

**7. "Cronjob running every few seconds to mark stuck ATTEMPTING jobs as OPEN"**

Good failure mode identification. But:
- A cronjob "every few seconds" running `UPDATE ... WHERE status = 'ATTEMPTING' AND last_status_at < now() - interval` at scale will be expensive
- You're creating a thundering herd: all stuck events become OPEN simultaneously and get grabbed by workers at once
- What if the cronjob itself fails? Single point of failure.

Better: workers set a lease time when claiming events. Other workers can steal events whose lease has expired. Distributed, no coordinator needed.

**8. The sharedSecret as Authorization header**

You mentioned it but didn't design it. Questions you should answer:
- Is the secret the same for all endpoints on an account, or per-endpoint?
- How do customers rotate secrets without downtime?
- What exactly is in the header? Bearer token? HMAC? 

The industry standard (which Stripe actually uses) is `Stripe-Signature: t=timestamp,v1=HMAC-SHA256(timestamp.payload)`. The timestamp prevents replay attacks; the HMAC proves Stripe sent it.

---

## What the Ideal System Looks Like

```
═══════════════════════════════════════════════════════════════════════════════
                              MAIN DATA FLOW (pipeline)
═══════════════════════════════════════════════════════════════════════════════

┌──────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Internal   │─────▶│      Kafka      │─────▶│  Router Service │
│   Stripe     │      │  (by tenant_id) │      │   (stateless)   │
│   Services   │      └─────────────────┘      └────────┬────────┘
└──────────────┘       500k events/sec                  │
                                                        │ writes delivery tasks
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Delivery Queue (SQS/Kafka, partitioned by endpoint_id)                     │
│  - Tenant isolation: slow endpoint only backs up its own partition          │
│  - Visibility timeout = worker lease (auto-recovery on worker crash)        │
└─────────────────────────────────────────────────────────────────────────────┘
                          │                               │
                    ┌─────┴─────┐                   ┌─────┴─────┐
                    ▼           ▼                   ▼           ▼
              ┌──────────┐ ┌──────────┐       ┌──────────┐ ┌──────────┐
              │ Worker 1 │ │ Worker 2 │  ...  │ Worker N │ │ Worker M │
              └────┬─────┘ └────┬─────┘       └────┬─────┘ └────┬─────┘
                   │            │                  │            │
                   └────────────┴────────┬─────────┴────────────┘
                                         │
              ┌──────────────────────────┬┴─────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
        ┌──────────┐              ┌────────────┐             ┌─────────────┐
        │ SUCCESS  │              │  FAILURE   │             │ EXHAUSTED   │
        │ ack msg  │              │ schedule   │             │ (N retries) │
        └────┬─────┘              │  retry     │             └──────┬──────┘
             │                    └─────┬──────┘                    │
             │                          │                           │
             │                          ▼                           ▼
             │                 ┌─────────────────┐         ┌─────────────────┐
             │                 │ Retry Scheduler │         │  Dead Letter    │
             │                 │ (Redis ZSET by  │         │  Queue          │
             │                 │  next_retry_at) │         │  + Customer     │
             │                 └────────┬────────┘         │    Notification │
             │                          │                  └─────────────────┘
             │                          │ when due
             │                          ▼
             │                 ┌─────────────────┐
             │                 │ Re-enqueue to   │
             │                 │ Delivery Queue  │─────────────────┐
             │                 └─────────────────┘                 │
             │                                                     │
             ▼                                                     │
      ┌─────────────────────────────────────────┐                  │
      │           Delivery Log                  │◀─────────────────┘
      │  (append-only, every attempt recorded)  │
      └─────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════
                         SUPPORTING SERVICES (not in pipeline)
═══════════════════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Endpoint Service                                                       │
  │  - Source of truth: Postgres (10M endpoints)                            │
  │  - Redis cache with pub/sub invalidation                                │
  │  - Tracks endpoint health scores                                        │
  │                                                                         │
  │  Queried by:                                                            │
  │    • Router Service ── "what endpoints for tenant X, event type Y?"     │
  │    • Workers ── "what's the health score? should I circuit-break?"      │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Metrics + Alerting                                                     │
  │  - delivery_latency_seconds (histogram, by endpoint tier)               │
  │  - delivery_status_total (counter, by status_code)                      │
  │  - queue_depth (gauge, by partition)                                    │
  │  - endpoint_health_score (gauge, by endpoint_id)                        │
  │  - Alerts: queue depth > threshold, success rate < 99%, p99 > 5s        │
  │                                                                         │
  │  Fed by: Workers, Delivery Queue metrics, Endpoint Service              │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Customer Dashboard API                                                 │
  │  - GET /webhooks/endpoints ── list configured endpoints                 │
  │  - GET /webhooks/events ── list events with delivery status             │
  │  - GET /webhooks/events/{id}/attempts ── delivery attempt history       │
  │  - POST /webhooks/events/{id}/retry ── manual replay                    │
  │  - GET /webhooks/endpoints/{id}/health ── success rate, latency         │
  │                                                                         │
  │  Reads from: Endpoint Service, Delivery Log, DLQ                        │
  │  Writes to: Delivery Queue (for manual retries)                         │
  └─────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════
                              DELIVERY WORKER DETAIL
═══════════════════════════════════════════════════════════════════════════════

  Worker behavior on each message:
  1. Pull message from queue (visibility timeout starts)
  2. Query Endpoint Service for health score
  3. If health < threshold → fast-fail (circuit breaker), nack with delay
  4. Make HTTPS call to endpoint:
     - Header: Stripe-Signature: t={ts},v1={hmac(ts.body, secret)}
     - Header: Idempotency-Key: {event_id}_{endpoint_id}_{attempt}
     - Timeout: 5s connect, 30s read
  5. On 2xx → ack message, write success to Delivery Log
  6. On error → nack with backoff, increment attempt, write to Delivery Log
  7. Update Endpoint Service with result (for health scoring)
```

---

## Key Differences: Your Design vs. Ideal

| Aspect | Your Design | Ideal Design | Why It Matters |
|--------|-------------|--------------|----------------|
| **Event ingestion** | Direct to Postgres | Kafka | Kafka handles 500k/sec writes trivially; Postgres doesn't. Kafka provides durability + ordering + decoupling. |
| **Work distribution** | Poll DB with SELECT FOR UPDATE | Message queue with partitions | Queue provides built-in work distribution, backpressure, and lease semantics without lock contention. |
| **Tenant isolation** | None—shared worker pool | Partition by endpoint_id | A slow endpoint only backs up its own partition; other tenants unaffected. This is the single biggest architectural difference. |
| **Retry scheduling** | Poll DB for eligible retries | Sorted set by next_retry_time | O(log N) to find due retries vs. full table scan. Retries scheduled precisely, not on next poll cycle. |
| **Backoff implementation** | Query filter on lastRetriedAt | Delayed message visibility / scheduled job | Queue natively supports "don't show this message for X seconds." No polling. |
| **Dead letter handling** | FAILED status, manual query | Explicit DLQ + customer notification | Customer knows immediately when events fail permanently; clear operational boundary. |
| **Stuck job recovery** | Cronjob scans ATTEMPTING | Queue visibility timeout / lease expiry | Distributed, no coordinator. Message reappears automatically if worker dies. |
| **Sub-system separation** | Monolithic workers do everything | Separate Router, Delivery, Retry, Observability | Each component scales independently; easier to reason about, test, and debug. |
| **Observability** | API to query events | Dedicated metrics, health scores, dashboards | Operators can see queue depth, latency percentiles, per-endpoint health in real-time. |
| **Signature scheme** | sharedSecret in Authorization header | HMAC-SHA256 with timestamp | Timestamp prevents replay attacks; HMAC is standard and well-understood. |

---

## The Core Mental Model Shift

Your design treats this as a **database problem**: "I have events in a table, I need to process them."

The ideal design treats this as a **streaming problem**: "I have a firehose of events that need to flow through a pipeline to millions of destinations with different characteristics."

The streaming mental model naturally leads to:
- Kafka for high-throughput, durable ingestion
- Partitioning for tenant isolation
- Queues with visibility timeouts instead of database locks
- Backpressure propagation instead of unbounded polling

This is the fundamental reframe that would have changed your architecture.