# Design a Global Edge Rate Limiting Service

## Context

Your company operates a global edge network spanning 40+ regions that proxies HTTP traffic for 25 million customer domains. Customers configure rate limiting rules (e.g., "allow 100 requests per minute per IP to `/api/*`") that must be enforced at the edge—before requests reach origin servers. When a client exceeds a rate limit, the edge must reject the request immediately with a `429 Too Many Requests` response. The system also powers an edge firewall that can block IPs, ASNs, or regions outright based on real-time abuse signals.

## Functional Requirements

1. **Rule Configuration**: Allow customers to define rate limiting rules specifying: match criteria (path pattern, HTTP method, headers, source IP/CIDR), threshold (request count), window (fixed or sliding), and action (block, challenge, log-only)
2. **Edge Enforcement**: Evaluate every proxied request against applicable rules at the nearest edge node and return `429` responses without forwarding to origin when limits are exceeded
3. **Counter Tracking**: Maintain accurate request counters per rule per client identifier (IP, API key, or customer-defined header value) across all edge nodes globally
4. **Firewall Rules**: Support immediate, globally-effective block/allow rules (IP, ASN, country) that take effect within seconds of configuration
5. **Rate Limit Analytics**: Provide per-rule request counts, block counts, and top offenders queryable by customers in near-real-time
6. **Rule Prioritization**: Evaluate rules in customer-defined priority order; first matching action wins

## Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Proxied requests | 30 million requests/second globally |
| Edge nodes | 300+ across 40 regions |
| Customer domains | 25 million |
| Active rate limit rules | 200 million |
| Unique client identifiers tracked | 2 billion per hour |
| Rule evaluation latency overhead | p50 < 1ms, p99 < 5ms added to request path |
| Firewall rule propagation | < 10 seconds globally |
| Counter accuracy | Within 5% of true count during normal operation |
| Availability | 99.999% for rule evaluation (fail-open acceptable during edge-local outage) |
| Rule configuration propagation | < 30 seconds from API to all edge nodes |

## Constraints

- Edge nodes have limited memory (64 GB per node) and must serve traffic for all 25 million domains
- Network partitions between regions occur regularly (multiple times per week); edge nodes must continue enforcing limits during partitions
- A single viral endpoint can generate 500,000+ requests/second from millions of distinct IPs
- Customers expect rate limits to work correctly even when traffic for the same rule arrives at dozens of different edge nodes simultaneously
- The system must not become an amplification vector—rejecting traffic must be cheaper than forwarding it
- Some customers require per-API-key rate limiting where the key is extracted from a request header, not just source IP
- Clock skew between edge nodes can be up to 200ms
- Must handle asymmetric attack patterns where an adversary deliberately splits traffic across many edge nodes to evade per-node detection
