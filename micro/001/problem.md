# Metrics and Logs Ingestion Pipeline

## Context

You are building the ingestion and query backbone for an observability platform similar to Datadog. Thousands of customers continuously emit metrics (CPU, memory, request latency, custom counters) and structured log lines from hundreds of thousands of hosts and containers. The system must accept this firehose in real-time, store it cost-effectively across multiple retention windows, and serve both recent high-resolution queries and long-range aggregate queries with low latency. The business charges per ingested metric series and per GB of logs, so cost visibility and cardinality governance are first-class concerns.

## Before You Design

Answer these before writing any architecture:

1. What is the core technical challenge this system must solve?
   A. queryability (how can we support querying an enormous amount of logs, at tiered "resolution"/fidelity levels)
   B. low-latency submission (for individual or batched metrics)
   C. accurate (probably, with eventual consistency) cost visibility
   D. cardinality tracking and potentially operations based on that.

2. Which of these framings best fits this problem?
   - [X] A storage and retrieval problem
   - [ ] A streaming/pipeline problem
   - [ ] A coordination/consensus problem
   - [ ] A scheduling/state machine problem
   - [ ] A distributed counting/aggregation problem

3. What is the one design decision that, if wrong, makes everything else irrelevant?

Queryability. The user needs to be able to access their logs.

## Sub-system Identification (5 minutes, no details yet)

1. Ingestion engine - users send their metrics here. it processes them
2. query engine - users hit this service to query their metrics
3. Metrics movement engine - moves metrics from "hot" storage to "cold" storage in intervals
4. Storage (hot, cold)

## Functional Requirements

- **Ingestion:** Accept metrics (gauge, counter, histogram, distribution) and structured logs via agents pushing over HTTPS/gRPC, supporting batched and compressed payloads.
- **Tagging:** Every data point carries arbitrary key-value tags (e.g., `host:web-12`, `env:prod`, `service:checkout`). Tags are the primary query dimension.
- **Cardinality tracking:** Detect and alert when a metric's unique tag-combination count (cardinality) exceeds configurable thresholds; optionally drop or aggregate runaway series.
- **Rollups:** Automatically downsample high-resolution data (10s intervals) into coarser resolutions (1m, 5m, 1h) for older retention tiers.
- **Query:** Support last-N-minutes dashboards (p99 latency over the last 15 min grouped by service), historical queries (week-over-week comparison), and full-text log search with tag filters.
- **Retention policies:** Per-customer configurable retention: e.g., 15 days full resolution, 13 months at 1-minute rollup, 3 years at 1-hour rollup. Logs retained 15 days hot, 12 months cold.
- **Multi-tenancy:** Strict data isolation between customers; fair-share resource allocation so one noisy tenant cannot degrade others.

## Non-Functional Requirements

- **Ingestion throughput:** 5 million metric data points per second sustained, 15 million peak. 2 TB/day of raw logs.
- **Ingestion latency:** Data queryable within 10 seconds of receipt for recent queries.
- **Query latency:** < 200ms p95 for last-1-hour dashboard queries; < 5s p95 for 30-day historical queries.
- **Availability:** 99.95% for ingestion path; 99.9% for query path.
- **Storage cost:** Must demonstrate a clear cost model — full-resolution hot storage is expensive; the system should reduce per-GB cost by 5-10x for cold tiers.
- **Durability:** Zero data loss for metrics once acknowledged to the agent.

## Constraints

- Agents are already deployed and push data; you cannot require a pull-based model.
- Tag keys and values are arbitrary strings; you cannot pre-define a schema.
- Some customers emit 500k+ unique metric series; others emit 500. The system must handle both without per-customer provisioning.
- Budget is meaningful — you must justify storage tier choices with rough cost math (e.g., SSDs vs. object storage per TB/month).
- The platform already runs on AWS; assume availability of S3, EC2, EBS, Kinesis, and managed Kafka (MSK), but you may use open-source alternatives where justified.
