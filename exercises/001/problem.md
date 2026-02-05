# Design Stripe Webhook Delivery Infrastructure

## Context

Stripe processes billions of dollars in payments and needs to notify merchant applications when events occur (e.g., `payment_intent.succeeded`, `charge.refunded`, `invoice.paid`). You are designing the system that reliably delivers these webhook events to millions of customer-configured HTTPS endpoints, ensuring merchants can trust that they will receive every event needed to keep their systems in sync.

## Functional Requirements

1. **Event Delivery**: Deliver webhook events to customer-configured HTTPS endpoints when events occur in the Stripe platform
2. **Retry Logic**: Automatically retry failed deliveries with configurable backoff strategy
3. **Delivery Guarantees**: Provide at-least-once delivery semantics—events must never be silently lost
4. **Idempotency Support**: Include idempotency keys so customers can safely deduplicate retried events
5. **Failed Event Handling**: Surface persistently failing events to customers for manual inspection and replay
6. **Endpoint Management**: Allow customers to configure multiple endpoints per account with event type filtering
7. **Delivery Status**: Provide customers visibility into delivery attempts, failures, and event history

## Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Event ingestion rate | 500,000 events/second peak |
| Registered endpoints | 10 million |
| Active merchants | 2 million |
| Delivery latency (p50) | < 500ms from event creation to first delivery attempt |
| Delivery latency (p99) | < 5 seconds from event creation to first delivery attempt |
| Event retention | 30 days for replay/inspection |
| Availability | 99.99% for event ingestion; 99.9% for delivery pipeline |
| Retry window | Up to 72 hours for failing endpoints |

## Constraints

- Customer endpoints are external HTTPS URLs with unpredictable latency and availability characteristics
- Some merchant endpoints may be misconfigured, slow, or intentionally malicious
- Webhook payloads average 2KB but can reach 64KB for complex events
- Customers expect cryptographic signature verification to authenticate webhook origin
- Must not allow one customer's failing/slow endpoint to impact delivery to other customers
- System must handle sudden traffic spikes (e.g., Black Friday: 10x normal volume)
- Customers may have compliance requirements for event ordering within certain event types
