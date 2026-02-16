# Design: Real-Time Fraud Decisioning Pipeline

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We are designing a system that accepts events and classifies them based on a classifier. We are designing the data stores and processes the classifier will use as well.

**Core technical challenge:** Constructing a pipeline of events to both a "hot" and "cold" path for the classifier (online vs offline) features.

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [X] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

- Online feature store - this is what the classifier invokes to make a decision
- Offline feature store - this is what will be eventually used for training of the classifier and other analytics
- Event ingestion engine - where events are processed, and routes to the proper places
- front api - the user hits this to get their classification

Time elapsed: 74 mins

## Phase 3: Key Decisions (15 min)

### Decision 1: How should risk features be produced for sub-120ms p99 decision latency?

- **A.** Synchronous feature reads at decision time from OLTP systems plus cache
- **B.** Stream pre-computation into a materialized online feature store
- **C.** Hybrid: pre-computed velocity features plus synchronous lookup for sparse attributes

**Your choice:** C
**Why:** Option A would be DB-heavy and subject to a thundering herd upon cache invalidation. Option B allows the creation of a low-latency storage solution (an online feature store) that the inference system needs. Option C adds a layer for the infrequently-changing features.
**How it works:**
1. An event comes in.
2. we store the event in a persistent db
3. via CDC or a producer, we send off a kafka message with the event 
4. there is a stream processor job waiting to pick up messages, one job per feature we care about (and potentially many workers per job).
5. when a stream processor job picks up its message, it'll write back our online feature store (e.g redis, key could be `account_id:123:features`)  with the aggregated feature info (e.g `{purchases_5m: 2, last_tx_ts: <time>}`)
6. when the inference component needs the data, it's quickly accessible via a lookup (e.g `account_id:123:features`).

Time elapsed: 111 mins

### Decision 2: What decision execution path should the payment authorization call use?

- **A.** Inline scoring in the payment API process with local rule/model runtime
- **B.** Dedicated fraud decision service with bounded-time RPC and explicit fallback policy
- **C.** Queue-first asynchronous scoring with later reconciliation callback

**Your choice:** B
**Why:** Inline scoring in the payment api process couples the scoring and the api. That means they deploy together and fail together. That's not good. The benefit of the option is lower latnecy. The dedicated fraud inference service is a good option because it de-couples the api and detection, but will be higher-latency than inline because we're making an RPC call. Option C is not deterministic within the 200ms, so we can't accept that option.
**How it works:**
We will make an RPC call with the transaction event info to the fraud inference service, store the result in a relational DB for analytics, and return the result.
It seems like the better path is to fail-open on this fraud detection service (if it's down or the call times out after some reasonable TTL) and allow all transactions while it's down. We can have a reconiliation mechanism (a one-off worker job) to back-assess transactions that occurred while the fraud detection service was down.

= 129 mins

### Decision 3: Where should short-lived risk state and deduplication state live?

- **A.** In-memory key-value cluster with TTL-based risk state and idempotency keys
- **B.** Stream processor managed state with changelog-backed recovery
- **C.** Wide-column store keyed by account/device with time-bucketed rows

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

---

Time elapsed: ___
