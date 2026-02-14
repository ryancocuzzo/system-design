# Real-Time Fraud Decisioning Pipeline

## Context
A global payments platform needs to score card authorization attempts in real time before the issuer timeout window closes. Each decision combines transaction payloads with fast-moving behavioral signals such as device reputation, velocity, and merchant risk posture. The hard part is maintaining low-latency, high-confidence decisions when events arrive out of order and fraud patterns shift quickly.

## Functional Requirements
1. Ingest payment authorization events and streaming risk signals (device, account, merchant) in real time
2. Produce one decision per authorization attempt: `approve`, `review`, or `decline`
3. Support dynamic policy updates (rules and model versions) without stopping traffic
4. Emit decision and outcome events for analyst investigation and post-transaction learning

## Non-Functional Requirements
1. Handle 150,000 authorization events/second at peak, with burst tolerance up to 2x for 5 minutes
2. End-to-end decision latency must be <120ms at p99, with a hard timeout of 200ms
3. Service availability must be 99.99% monthly across two active regions
4. Catch at least 92% of confirmed fraud while keeping false positives below 0.3% for low-risk traffic

## Constraints
1. Upstream behavioral events can arrive up to 2 seconds late and out of order
2. Stream processing components cannot persist raw PII; only tokenized identifiers are allowed
3. If scoring dependencies fail or time out, the system must still return a deterministic fallback decision within 200ms

Time elapsed: 4 mins.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- At-least-once vs exactly-once — affects duplicate handling and correctness of fraud decisions under retries
- Windowing — shapes how recent behavior is aggregated when signals are delayed or out of order

Time elapsed: 6 mins.