# Design: Real-Time Fraud Decisioning Pipeline

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:**

**Core technical challenge:** [Not the product goal — what makes this hard at scale?]

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

## Phase 3: Key Decisions (15 min)

### Decision 1: How should risk features be produced for sub-120ms p99 decision latency?

- **A.** Synchronous feature reads at decision time from OLTP systems plus cache
- **B.** Stream pre-computation into a materialized online feature store
- **C.** Hybrid: pre-computed velocity features plus synchronous lookup for sparse attributes

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

### Decision 2: What decision execution path should the payment authorization call use?

- **A.** Inline scoring in the payment API process with local rule/model runtime
- **B.** Dedicated fraud decision service with bounded-time RPC and explicit fallback policy
- **C.** Queue-first asynchronous scoring with later reconciliation callback

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

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
