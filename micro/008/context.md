# Context — Real-Time Fraud Decisioning Pipeline

## Context setting questions

Q: Is this asking us to design the actual scoring/ranking algorithm? I've never designed one. I'm not really sure how that would work. Are there specific things I can research that can quickly triage my understanding?

No. This exercise is about the system architecture around real-time fraud decisioning, not building a novel ML model from scratch.
Treat the model/rules engine as a black box that outputs a risk score, and focus your design on: ingestion, feature freshness, latency budgets, idempotency/dedup, fallback behavior, and policy/model rollout safety.

Quick triage research (high ROI):
1. **Feature pipeline basics**: "online features vs offline features in fraud systems" - understand how velocity features are precomputed and served fast.
2. **Decision path design**: "synchronous fraud decision service timeout and fallback policy" - learn fail-open/fail-closed tradeoffs. 
3. **Streaming correctness**: "event time windowing with late data and watermarking" - learn how out-of-order events affect risk features.
4. **Idempotency in streams**: "deduplication keys in at-least-once processing" - avoid duplicate events causing inconsistent decisions.

My research from the above tells me:
- We'll need both online feature service (in a redis-like cache) for the prediction component and large, queryable relational data for the offline training of that srevice
- We should be failing-open in the approval flow. Trade-off is risking $ vs customer satisfaction. If the fraud system has an issue, we could create a lot of unhappy customers by failing their transactions irrationally.
- The simplest late data strategy seems to be dead-letter. We'll drop all late notifications. The value is simplicity. This means additional risk of missing important out-of-window events.
- We should should do idempotency key checks in some cache (e.g redis) for stream events

Time elapsed: 41 mins.

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **At-least-once vs exactly-once** — In a fraud decision pipeline, when do at-least-once guarantees become risky, and what mechanisms keep duplicate events from causing inconsistent decisions?

at-least-once gaurantees are risky if you're double-processing. For example, a user might have a transaction just below a high-risk threshold for total spend per 5 transactions (e.g they bought a car). If we process that transaction twice, they are now above the high-risk threshold to where they will be flagged as a fraud candidate. Since we're using streaming, We can de-duplicate (and safeguard this) with idempotency keys checked against a redis store to verify we havent processed that event yet.

2. **Windowing** — How do tumbling, hopping, and sliding windows change fraud feature quality when behavioral events arrive late or out of order?

- tumbling is simplest to operate, but suffers at the edges (they absorb bursts at the edge into the full window aggregate, which loses information). tumbling requires only 1 update for late data (update that timeframe)
- hopping offers smoother edges than tumbling, which gives us more info on patterns at the edge, but updates to late data mean updating several time windows (fanning-out to the number of time windows that contain that time). In terms of fraud feature quality, these windows still do technically have edges, so they're not perfect.
- sliding windows I don't fully understand, but they appear to be the best fraud feature quality but the most operationally expensive.

Total time elasped: 65 mins.

## During Design
[Questions that arise while designing. Note which phase triggered them.]

