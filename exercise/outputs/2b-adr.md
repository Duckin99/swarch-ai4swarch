Antigravity (Gemini 3.8 Flash)

# ADR-0002: Use RabbitMQ for asynchronous post-order processing

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Online Boutique Core Engineering Team

## Context and problem

In Online Boutique, `checkoutservice.PlaceOrder` (Go) currently coordinates the order placement process synchronously. It prepares cart items, converts currencies, charges the customer via `paymentservice`, ships items via `shippingservice`, empties the user cart via `cartservice`, and calls `emailservice.SendOrderConfirmation` synchronously over gRPC before returning a response to the user. At present, the p95 latency of `PlaceOrder` is approximately 300 ms.

A new business requirement (R-12) introduces several post-order actions:
1. Send order confirmation emails via **MailFast**, a third-party email provider replacing the mock `emailservice`.
2. Award **loyalty points** via a new `loyaltyservice` (to be developed).
3. Record an **"order placed" analytics event** for the marketing team.
4. Support **future post-order actions** expected next year (SMS notifications, fraud scoring, recommendations update).

These requirements introduce severe performance, availability, and throughput challenges:
- **Latency constraint:** The p95 latency of `PlaceOrder` must stay ≤ 500 ms. However, MailFast has a p95 latency of 1.5 s and a p99 of 8 s. Calling MailFast synchronously would immediately violate the 500 ms SLA.
- **Throughput & Rate Limiting:** Peak traffic during the 11.11 sale reaches 200 orders/s. MailFast imposes a strict rate limit of 50 emails/s. Direct synchronous forwarding would exceed MailFast's quota by 4x, causing 75% of calls to be throttled or rejected.
- **Availability:** Orders must never fail if email, loyalty, or analytics services fail or degrade (MailFast has an expected ~0.5% failure rate).
- **Loyalty timing & correctness:** Loyalty points must be awarded within 1 minute of the order, strictly exactly once (no double points).
- **Team & Infrastructure constraints:** The team consists of 4 developers strong in Go and gRPC, with 2 having prior experience with RabbitMQ. The application runs on Kubernetes with a strong preference for open-source, self-hosted components over cloud-managed services.

We need an architectural mechanism to handle post-order processing that isolates the critical checkout path, handles traffic buffering, guarantees event delivery, and supports modular extensibility.

## Decision drivers

1. **Performance and Latency:** p95 latency of `PlaceOrder` must remain ≤ 500 ms at 200 orders/s peak load, isolating the user checkout path from slow downstream operations (MailFast p95 = 1.5 s, p99 = 8 s).
2. **Availability and Fault Isolation:** Order completion must succeed independently of downstream availability or transient failures in email, loyalty, or analytics.
3. **Scalability and Rate Limiting:** The system must absorb peak bursts of 200 orders/s and throttle requests to MailFast to stay within its 50 emails/s limit.
4. **Reliability and Consistency:** Loyalty points must be awarded within 1 minute and strictly exactly once, even in the event of retries or transient crashes.
5. **Modifiability and Extensibility:** Decouple `checkoutservice` so future marketing post-order actions (SMS, fraud scoring, recommendations) can be added without modifying checkout logic.
6. **Simplicity and Team Expertise:** The architecture must run on self-hosted Kubernetes and match the team's skillset (Go, gRPC, and RabbitMQ experience).

## Considered options

1. Synchronous gRPC chaining with background in-memory goroutines in `checkoutservice`
2. Asynchronous event publish-subscribe via self-hosted RabbitMQ on Kubernetes
3. Distributed event streaming via Apache Kafka

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| In-memory goroutines in `checkoutservice` | No new infrastructure components to deploy or operate; leverages existing Go/gRPC codebase; minimal implementation effort. | In-memory tasks are lost if a pod restarts or crashes, violating the loyalty point guarantee; does not buffer or queue peak bursts (200 orders/s) to respect MailFast's 50/s rate limit; adding future actions requires modifying `checkoutservice`, increasing coupling. |
| RabbitMQ (self-hosted in-cluster) | Sub-5ms publish latency preserves `PlaceOrder` p95 well under 500 ms; durable queues buffer peak 200 orders/s load and throttle MailFast dispatch to ≤ 50 emails/s; fanout/topic exchanges allow adding future subscribers (SMS, fraud, recommendations) without touching `checkoutservice`; matches team skills (2 devs know RabbitMQ); open-source and self-hostable on Kubernetes. | Introduces another stateful component to operate and monitor on Kubernetes; at-least-once message delivery requires `loyaltyservice` to implement deduplication for exactly-once processing; eventual consistency for post-order tasks. |
| Apache Kafka | High throughput and persistent log retention; replayability; robust pub/sub scaling. | Significant operational overhead (JVM memory footprint, ZooKeeper/KRaft cluster management); steep learning curve for a 4-person team with zero Kafka experience; unnecessary complexity for 200 orders/s traffic. |

## Decision

Use **RabbitMQ** (Option 2) deployed as an in-cluster, self-hosted message broker on Kubernetes for asynchronous post-order event processing.

1. **Checkout flow:** In `checkoutservice.PlaceOrder`, after successful payment, shipping, and cart clearing, publish an `OrderPlaced` event to a RabbitMQ fanout exchange (`order.events`) using publisher confirms. The synchronous gRPC response returns immediately to the client. Publishing to RabbitMQ adds only ~2–5 ms latency, keeping `PlaceOrder` p95 at ~305 ms (well below the 500 ms SLA).
2. **Post-order consumers & queues:**
   - **`emailservice` (MailFast worker):** Binds a durable queue (`order.email`) to the exchange. Uses a worker pool configured with rate limiting (or prefetch/concurrency limits) to ensure outgoing calls to MailFast never exceed 50 emails/s. During the 11.11 peak (200 orders/s), the queue acts as a buffer. Transient errors (~0.5% failure rate) are retried with exponential backoff and routed to a dead-letter exchange (DLX) upon exhausting retries.
   - **`loyaltyservice`:** Binds a durable queue (`order.loyalty`). Consumes messages promptly to ensure points are credited within 1 minute of the order. To achieve the required **exactly-once** guarantee, `loyaltyservice` implements an idempotent consumer pattern by tracking processed `order_id` values with a unique constraint in its database.
   - **`analytics-worker`:** Binds a durable queue (`order.analytics`) to record the marketing event.
3. **Future extensibility:** When marketing introduces SMS, fraud scoring, or recommendation updates next year, each service will declare its own queue bound to the `order.events` exchange without requiring any code changes or redeployments of `checkoutservice`.

## Consequences

- **Positive:** `checkoutservice.PlaceOrder` p95 latency remains at approximately 305 ms, well within the 500 ms target, by removing MailFast's 1.5 s p95 response time from the critical request path (quality attribute: performance).
- **Positive:** Order placement continues to succeed even if MailFast, `loyaltyservice`, or analytics is unavailable or slow (quality attribute: availability).
- **Positive:** RabbitMQ queues absorb 11.11 peak traffic bursts of 200 orders/s and throttle dispatch to MailFast at ≤ 50 emails/s, preventing quota rejections (quality attribute: scalability).
- **Positive:** Decoupled pub/sub design allows adding new post-order handlers (SMS, fraud scoring, recommendations) by binding new queues without touching `checkoutservice` (quality attribute: modifiability).
- **Positive:** Uses self-hosted open-source software on Kubernetes and directly leverages the team's existing RabbitMQ experience without proprietary cloud lock-in (quality attribute: operability / cost).
- **Negative:** Requires deploying, clustering, and monitoring a stateful RabbitMQ cluster on Kubernetes (quality attribute: operability).
- **Negative:** Downstream actions become eventually consistent rather than immediate upon checkout response (quality attribute: consistency).
- **Risks and mitigations:**
  - *Risk:* RabbitMQ cluster failure could cause message publication failures in `checkoutservice`.  
    *Mitigation:* Deploy RabbitMQ in high-availability mode with Quorum Queues across Kubernetes nodes. If publishing fails, `checkoutservice` can log an alert or store to a local outbox/fallback buffer, ensuring user orders are not rolled back after payment.
  - *Risk:* At-least-once message delivery could cause duplicate point awards on consumer retries.  
    *Mitigation:* `loyaltyservice` must enforce idempotency using `order_id` as a unique primary key / deduplication record in its transactional datastore.
  - *Risk:* During peak load (200 orders/s), buffering against MailFast's 50/s rate limit will temporarily delay confirmation email delivery.  
    *Mitigation:* At 200 orders/s arrival and 50/s drain, the email backlog grows by 150 msgs/s during burst periods and drains once traffic eases. Monitor queue lag with Prometheus/Alertmanager and verify that burst latency stays within acceptable business limits.

## Validation

- **Load and latency testing:** Execute a Locust load test at 200 orders/s peak load and measure `PlaceOrder` p95 latency to verify it remains ≤ 500 ms (expected ~305 ms).
- **Rate-limit compliance test:** Verify via monitoring metrics that outgoing request rates from `emailservice` to MailFast remain capped at ≤ 50 emails/s under maximum queue backlog.
- **Fault-injection / Availability testing:** Simulate MailFast outages and terminate `loyaltyservice` during active checkouts; verify that `checkoutservice.PlaceOrder` achieves 100% success rate without errors.
- **Loyalty SLA & Idempotency verification:** Validate under peak load that `order.loyalty` messages are consumed and points credited within 60 seconds of order creation; redeliver duplicate messages to verify no duplicate points are awarded.

## Open questions

- What database technology will the new `loyaltyservice` use for persisting user loyalty point balances and the deduplication table (`order_id`)?
- What is the maximum acceptable delay for order confirmation emails during sustained peak bursts when queue buffering occurs?
- What alerting policy and operational procedure should handle poisoned messages in the dead-letter queue (DLQ) after MailFast retries are exhausted?
