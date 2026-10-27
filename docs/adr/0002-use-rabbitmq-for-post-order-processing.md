# ADR-0002: Use RabbitMQ with Transactional Outbox for asynchronous post-order processing

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Lead Architect, Online Boutique Core Engineering Team

## Context and problem

In Online Boutique, `checkoutservice.PlaceOrder` (Go) coordinates order placement synchronously over gRPC: reserving cart items, calculating currency conversions, charging the card via `paymentservice`, shipping via `shippingservice`, emptying the cart, and sending confirmation emails via `emailservice`. The current p95 latency of `PlaceOrder` is approximately 300 ms.

Requirement R-12 introduces three new post-order actions, with more planned next year (SMS, fraud detection, recommendation updates):
1. Dispatch order confirmation emails via **MailFast**, replacing mock email logic.
2. Award **loyalty points** via a new `loyaltyservice`.
3. Record an **"order placed" analytics event** for marketing.

These requirements introduce critical performance and reliability conflicts:
- **Latency SLA:** `PlaceOrder` p95 must remain ≤ 500 ms. MailFast has a p95 latency of 1.5 s (p99 of 8 s). Synchronous invocation would immediately violate our SLA.
- **Throughput & Quotas:** Peak load reaches 200 orders/s during 11.11, whereas MailFast enforces a strict quota of 50 emails/s. Direct calls would trigger massive 429 throttling.
- **Availability:** Checkout must never fail due to downstream email, loyalty, or analytics degradation (MailFast has an expected ~0.5% failure rate).
- **Consistency:** Loyalty points must be awarded within 1 minute, strictly exactly once.
- **Team & Environment:** 4 developers strong in Go/gRPC with RabbitMQ experience; deployed on self-hosted Kubernetes with a preference for open-source software.

## Decision drivers

1. **Latency & Performance:** Keep `PlaceOrder` p95 ≤ 500 ms under 200 orders/s peak load by decoupling external/downstream calls.
2. **Availability & Fault Isolation:** Prevent MailFast, loyalty, or analytics outages from failing customer checkouts.
3. **Throughput Regulation:** Buffer 200 orders/s bursts and throttle MailFast dispatches to ≤ 50 emails/s.
4. **Reliability & Exactly-Once Semantics:** Guarantee point crediting within 60 seconds with strict deduplication.
5. **Modifiability:** Enable plug-and-play addition of future post-order consumers without modifying `checkoutservice`.
6. **Operational Fit:** Self-hostable on Kubernetes matching team capabilities (Go, gRPC, RabbitMQ).

## Considered options

1. **Synchronous orchestration with background in-memory goroutines in `checkoutservice`**
2. **Asynchronous event pub/sub via self-hosted RabbitMQ (Quorum Queues) with Transactional Outbox**
3. **Distributed streaming via Apache Kafka**

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option 1: In-memory goroutines** | No new infrastructure; zero additional operational cost. | Tasks lost on pod crash/restart (violating loyalty SLA); cannot throttle 200/s bursts to 50/s MailFast quota; tight code coupling for future actions. |
| **Option 2: RabbitMQ + Outbox (Chosen)** | Sub-5ms publish overhead; durable buffering absorbs 200/s peaks; fanout exchanges support new consumers with zero checkout changes; aligns with team RabbitMQ skills; outbox prevents message loss. | Requires managing stateful RabbitMQ cluster on Kubernetes; requires idempotent consumers. |
| **Option 3: Apache Kafka** | High retention and massive throughput. | Excessive operational overhead (JVM, cluster storage); steep learning curve; unnecessary for 200 orders/s scale. |

## Decision

We adopt **Option 2: Self-hosted RabbitMQ on Kubernetes with a Transactional Outbox pattern** for post-order processing.

1. **Checkout & Publishing:**
   - In `checkoutservice.PlaceOrder`, after successful payment and cart clearing, serialize an `OrderPlaced` event using **Protocol Buffers** (`order_event.proto`).
   - Write the event to a local transactional outbox store (or publisher-confirmed channel with local disk fallback).
   - Publish to the RabbitMQ fanout exchange `order.v1.events`.
   - `PlaceOrder` returns immediately to the customer (~305 ms p95, preserving the ≤ 500 ms SLA).

2. **Consumer Topology & Rate Limiting:**
   - **`emailservice` (MailFast worker):** Binds queue `order.email` to `order.v1.events`. Worker concurrency is rate-limited to 50 requests/s to strictly respect MailFast limits. Failures are retried with exponential backoff up to 5 times before routing to a Dead Letter Exchange (`order.dlx`).
   - **`loyaltyservice`:** Binds queue `order.loyalty`. Persists to PostgreSQL. Implements an idempotent consumer using `order_id` as a unique primary key constraint to ensure points are credited exactly once within 60 seconds.
   - **`analytics-worker`:** Binds queue `order.analytics` to stream events to marketing storage.

3. **Future Extensibility:**
   - Future actions (SMS, fraud scoring, recommendations) will bind independent queues to `order.v1.events` without code changes to `checkoutservice`.

## Consequences

- **Positive:**
  - `PlaceOrder` p95 remains at ~305 ms, well below the 500 ms ceiling (performance).
  - Outages or latency spikes in MailFast or downstream consumers cannot block or fail checkout (availability).
  - RabbitMQ queues buffer peak bursts (200 orders/s) and smoothly dispatch to MailFast at ≤ 50/s (scalability).
  - New marketing actions can be deployed independently without modifying `checkoutservice` (modifiability).
  - Zero cloud vendor lock-in; matches team skills (operability, cost).

- **Negative:**
  - Introduces a stateful RabbitMQ cluster to monitor and operate on Kubernetes (operability).
  - Post-order actions become eventually consistent (consistency).

- **Risks and mitigations:**
  - *Risk:* RabbitMQ unavailable during checkout.  
    *Mitigation:* Local outbox fallback ensures events are drained once the broker recovers; orders are never lost after payment.
  - *Risk:* MailFast persistent failure triggers queue backlog.  
    *Mitigation:* Messages exceeding 5 retries move to `order.email.dlq` with Prometheus alert thresholds; retained for 48 hours for manual replay.
  - *Risk:* Duplicate message delivery from network retries.  
    *Mitigation:* `loyaltyservice` enforces unique index on `order_id` in PostgreSQL.

## Validation

- **Locust Load Test:** Simulate 200 orders/s peak load; verify `PlaceOrder` p95 stays ≤ 500 ms.
- **Quota Validation:** Verify outbound HTTP requests to MailFast never exceed 50 req/s during queue bursts.
- **Fault Injection:** Terminate `emailservice` and RabbitMQ pods during active ordering; ensure zero customer checkout failures.
- **Idempotency Test:** Inject duplicate `OrderPlaced` messages to `loyaltyservice`; verify no duplicate points are credited.