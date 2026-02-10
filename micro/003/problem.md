# Multi-Tenant Search-as-a-Service

## Context

You are building a managed search platform that provides Elasticsearch/OpenSearch-backed full-text search as a service. Tenants range from small apps with tens of thousands of documents to large e-commerce platforms with hundreds of millions of products. Each tenant defines custom field mappings and text analyzers, and expects documents to become searchable shortly after ingestion. The platform runs shared Elasticsearch clusters — tenants do not get dedicated infrastructure.

## Functional Requirements

- **Index configuration:** Tenants create search indices with custom field mappings (text, keyword, numeric, nested) and text analyzers (language-specific stemming, synonym lists, custom tokenizers).
- **Document ingestion:** Tenants push documents via API. Documents must be searchable within 5 seconds of acknowledgment under normal load.
- **Search API:** Tenants issue full-text queries with filters, faceted aggregations, and relevance scoring against their own data only.
- **Schema evolution:** Tenants can modify field mappings and analyzers on existing indices. Changes apply to all existing documents without search downtime.

## Non-Functional Requirements

- **Scale:** 8,000 tenants. Individual tenants range from 50K to 500M documents (median: 2M). Top 3% of tenants hold 70% of total documents.
- **Indexing throughput:** 40K documents/second aggregate. Individual tenants can burst to 8K docs/sec during bulk imports.
- **Search latency:** < 50ms p95 for typical queries; < 500ms p95 for complex aggregation queries across large tenants.
- **Noisy neighbor isolation:** No single tenant operation — bulk import, expensive query, or reindex — may degrade search latency for other tenants by more than 20%.

## Constraints

- Shared Elasticsearch/OpenSearch clusters. A single node should hold no more than 1,000 shards; optimal shard size is 10–50 GB.
- Adding cluster capacity (new data nodes) takes 5–10 minutes. The system must absorb burst load with existing capacity.
- Tenants expect zero search downtime during schema evolution — queries must return results while reindexing is in progress.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- **Data Modeling and Storage** — how Elasticsearch shard sizing, index design, and routing strategy affect query parallelism, cluster stability, and tenant density on shared clusters
- **Reliability and Resilience** — how bulkhead patterns and resource quotas prevent one tenant's operations from degrading search performance for others on shared infrastructure

Time elapsed: 6 mins.