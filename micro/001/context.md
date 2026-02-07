# Context Questions — Metrics and Logs Ingestion Pipeline

Questions asked and answered before or during the design.
Separate from the design document to keep architectural reasoning clean.

## Before Design

### Q: What does metrics cardinality governance mean? My guess would be that it means we're monitoring the number of unique metrics?

Close, but the nuance matters. It's not the number of unique *metric names* — it's the number of unique **time series**, which is the product of a metric name and its distinct tag-combination set.

A single metric name like `http.request.latency` fans out into one time series per unique tag combination:

- `http.request.latency{host:web-1, env:prod, endpoint:/api/users}` — series 1
- `http.request.latency{host:web-2, env:prod, endpoint:/api/orders}` — series 2
- ...

If you have 100 hosts × 3 environments × 50 endpoints, that one metric name produces **15,000 series**. Manageable. But if someone adds a tag like `request_id` (unique per request at 1,000 RPS), that same metric explodes to **millions of new series per hour**. This is a **cardinality explosion** — the single most dangerous operational event in a metrics platform.

**Governance** means more than monitoring. It's the full control loop:

1. **Tracking** — counting unique series per metric per customer in near-real-time (often via probabilistic structures like HyperLogLog since exact counting at this scale is expensive).
2. **Alerting** — notifying customers when they approach or exceed their cardinality budget.
3. **Enforcing** — actively intervening when cardinality spirals: dropping new series beyond a threshold, stripping high-cardinality tags, or aggregating runaway series into a summary bucket.
4. **Billing visibility** — since customers are charged per series, governance feeds directly into cost attribution.

Why this is a first-class design concern: an ungoverned cardinality explosion from a single customer can saturate indexing, blow out memory in the query engine, and degrade the platform for *every* tenant. The ingestion pipeline must detect and react to this within seconds, not after the damage is done.

## During Design
[Ask and answer questions that arise while designing. Note which phase triggered the question.]
