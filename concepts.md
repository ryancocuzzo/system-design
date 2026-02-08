# System Design Concept Reference

Practical reference for use during design exercises. Organized for lookup, not study.

---

## Foundations

Mental models that drive every decision.

- Latency vs throughput
- Consistency vs availability (CAP theorem)
- Vertical vs horizontal scaling
- Push vs pull systems
- Sync vs async boundaries
- State vs stateless services
- CPU-bound vs IO-bound workloads
- Tail latency (p95/p99)
- Backpressure
- Idempotency
- Determinism
- Cost vs reliability vs complexity tradeoff triangle
- Bottlenecks (Amdahl's Law thinking)
- Critical path analysis
- Failure domains / blast radius
- SLO-first thinking (design for error budgets, not perfection)

---

## Architecture Patterns

Structural building blocks.

- Monolith (modular)
- Modular monolith
- Microservices
- Service-oriented architecture (SOA)
- Event-driven architecture
- Hexagonal / Ports & Adapters
- CQRS
- Event sourcing
- Saga pattern (orchestration vs choreography)
- Strangler Fig migration
- Backend-for-Frontend (BFF)
- API gateway
- Service mesh
- Sidecar pattern
- Shared-nothing architecture
- Sharding-first design
- Cell-based architecture
- Multi-tenant vs single-tenant

---

## Networking and Transport

- HTTP/1.1 vs HTTP/2 vs HTTP/3
- REST
- gRPC
- WebSockets
- SSE (Server-Sent Events)
- Long polling
- Load balancing (L4 vs L7)
- Reverse proxy
- CDN
- DNS TTL tuning
- Keep-alives
- Connection pooling
- Retries with jitter
- Exponential backoff
- Circuit breakers
- Rate limiting
- Throttling
- Hedged requests
- Request coalescing
- Zero-downtime deploys (blue/green, canary, rolling)

---

## Data Modeling and Storage

### Database types

- Relational (PostgreSQL, MySQL)
- Key-value (Redis, DynamoDB)
- Document (MongoDB)
- Columnar (ClickHouse, Redshift)
- Wide-column (Cassandra, HBase)
- Graph (Neo4j)
- Time-series (InfluxDB, M3DB, VictoriaMetrics)
- Search indexes (Elasticsearch, OpenSearch)

### Concepts

- Index design
- Query plans
- Partition keys
- Hot partitions
- Read vs write amplification
- Normalization vs denormalization
- OLTP vs OLAP
- Transactions and isolation levels
- Locking vs MVCC
- Secondary indexes
- Materialized views
- CDC (change data capture)
- Data locality
- Data gravity
- TTL / compaction
- Schema migrations and backfills
- Versioned schemas
- Data retention policies

---

## Caching

- CDN edge caching
- Reverse proxy cache
- In-memory cache (Redis, Memcached)
- Client-side cache
- Cache-aside
- Write-through
- Write-behind
- Refresh-ahead
- Cache invalidation strategies
- Cache stampede protection
- Negative caching
- TTL tuning
- Consistency implications
- Hot key mitigation

---

## Messaging and Async Systems

- Message queues (SQS, RabbitMQ)
- Pub/Sub
- Streams (Kafka, Kinesis)
- Dead letter queues
- Consumer groups
- At-least-once vs exactly-once
- Idempotent consumers
- Ordering guarantees
- Replay
- Event schemas
- Backpressure
- Batch vs real-time
- Stream processing (Flink, Spark Streaming)
- Windowing
- Compaction topics
- Durable subscriptions
- Outbox pattern
- Inbox pattern
- Saga coordination

---

## Reliability and Resilience

- SLO / SLA / SLI
- Error budgets
- Graceful degradation
- Fail-open vs fail-closed
- Retries vs timeouts vs cancellation
- Circuit breakers
- Bulkheads
- Rate limiting
- Health checks (readiness vs liveness)
- Heartbeats
- Leader election
- Consensus algorithms (Raft, Paxos)
- Replication strategies
- Quorums
- Split-brain handling
- Disaster recovery (RPO/RTO)
- Multi-region failover
- Chaos engineering
- Load shedding
- Brownouts
- Shadow traffic

---

## Security

- Authentication vs authorization
- OAuth / OIDC
- JWT
- mTLS
- Certificate rotation
- Secrets management
- RBAC / ABAC
- Principle of least privilege
- Network segmentation
- WAF
- Rate limiting for abuse
- Audit logs
- Encryption at rest / in transit
- Key management
- Threat modeling

---

## Observability

- Metrics, logs, traces
- Correlation IDs
- Structured logging
- RED metrics (Rate, Errors, Duration)
- USE metrics (Utilization, Saturation, Errors)
- Golden signals
- Percentiles
- Distributed tracing (Jaeger, Zipkin)
- OpenTelemetry
- Alert fatigue avoidance
- Burn rate alerts
- Runbooks
- Postmortems
- Capacity planning
- Load testing / soak testing
- Profiling

---

## Scaling Techniques

- Horizontal autoscaling
- Vertical scaling
- Sharding
- Read replicas
- Write splitting
- Partitioning
- Batch processing
- Async offloading
- Precomputation
- Fan-out / fan-in
- MapReduce
- CDN offload
- Edge compute
- Compression
- Streaming uploads
- Pagination / cursor-based pagination
- Lazy loading
- Data tiering
- Archival storage

---

## Design Process

- Requirements clarification
- Traffic estimation
- Back-of-the-envelope math
- Peak vs average planning
- Identify bottlenecks first
- Design for the common case
- Plan for failure explicitly
- Cost modeling
- Incremental rollout strategy
- Migration plan
- Operational complexity evaluation
- Tech debt accounting
- Simplicity bias
- Build vs buy vs managed
- Make state explicit
- Optimize only after measurement
