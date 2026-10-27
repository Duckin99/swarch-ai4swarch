Antigravity (Gemini 3.8 Flash)

# Architecture Review: ADR-0002 (RabbitMQ for Post-Order Processing)

## 1. Quality-Attribute Scenarios

- **Scenario 1: Checkout Latency & Peak Burst Buffering (Performance)**
  - *Source:* Customer.
  - *Stimulus:* 200 orders/s flash-sale load during 11.11.
  - *Environment:* Production peak; MailFast external quota restricted to 50 emails/s.
  - *Response:* `checkoutservice` completes payment and shipping, emits an `OrderPlaced` event asynchronously, and returns confirmation without blocking on downstream processing.
  - *Response Measure:* `PlaceOrder` p95 latency remains ≤ 500 ms (measured ~305 ms); zero orders are rejected due to downstream throttling.
- **Scenario 2: Downstream Outage Isolation (Availability & Fault Tolerance)**
  - *Source:* External provider (MailFast) or internal consumer (`loyaltyservice`).
  - *Stimulus:* MailFast experiences high latency (p99 8 s) or outage (~0.5% failure rate); `loyaltyservice` pod crashes.
  - *Environment:* Continuous customer order traffic.
  - *Response:* `checkoutservice` successfully places orders; messages safely buffer in RabbitMQ queues (`order.email`, `order.loyalty`); workers retry with exponential backoff and route to `order.dlx` after 5 failures.
  - *Response Measure:* 100% checkout success for valid orders (zero order failures caused by downstream systems); zero dropped events.
- **Scenario 3: Exactly-Once Loyalty Crediting (Data Consistency & Timeliness)**
  - *Source:* System (`checkoutservice` event publishing).
  - *Stimulus:* `OrderPlaced` event delivered multiple times due to network retries.
  - *Environment:* Concurrent consumer worker execution.
  - *Response:* `loyaltyservice` deduplicates message using `order_id` as a unique primary key in PostgreSQL and records loyalty points.
  - *Response Measure:* Points credited within 60 seconds of order; exactly 0 duplicate point allocations.

---

## 2. Findings Table

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | Ambiguous Outbox storage in stateless `checkoutservice` | `checkoutservice` has no local ACID database. Storing outbox events on "local disk" inside ephemeral Kubernetes pods risks losing post-order events during pod evictions or restarts, causing paid orders to miss loyalty points and confirmation emails. |
| **Risk** | Unbounded queue accumulation during MailFast degradation | At 200 orders/s peak with MailFast capped at 50 emails/s, queues accumulate +150 msgs/s (9,000/min). A prolonged MailFast outage will trigger RabbitMQ memory/disk high watermarks, which by default blocks publishers and cascades into blocking `checkoutservice`. |
| **Risk** | Operational burden of clustered RabbitMQ for a 4-developer team | Running stateful RabbitMQ (Quorum Queues, Raft consensus, PVCs, partition recovery) on Kubernetes introduces significant operational overhead for a small 4-developer team without dedicated SRE support. |
| **Non-risk** | Decoupling MailFast from synchronous checkout flow | Removing MailFast (p95 1.5 s, p99 8 s) from the gRPC critical path protects the 500 ms latency SLA and ensures third-party outages cannot abort checkouts. |
| **Non-risk** | Database-level idempotency via `order_id` in `loyaltyservice` | Relying on PostgreSQL unique constraints on `order_id` guarantees exactly-once point allocation despite at-least-once message delivery over RabbitMQ. |
| **Non-risk** | Fanout exchange pub/sub topology for future extensibility | Publishing to `order.v1.events` allows adding SMS, fraud detection, and recommendation workers next year with zero modifications to `checkoutservice`. |
| **Sensitivity point** | MailFast rate-limit ceiling (50 req/s) vs. Peak load (200 orders/s) | Queue length, memory consumption, and email delivery lag are strictly sensitive to this ratio; an 11.11 sale generates a sustained queue backlog that requires hours to drain. |
| **Sensitivity point** | Publisher confirmation timeout in `checkoutservice` | Checkout p95 latency is highly sensitive to the broker publish timeout under network jitter or disk backpressure; a blocking publish directly threatens the 500 ms SLA. |
| **Trade-off point** | Eventual consistency vs. Checkout latency and availability | Decoupling downstream services achieves low latency (≤ 500 ms) and high availability, but sacrifices immediate user feedback on email dispatch and loyalty points. |
| **Trade-off point** | Operational complexity vs. Buffering & architectural decoupling | Introducing a self-hosted RabbitMQ cluster adds infrastructure maintenance, but provides essential peak smoothing (200/s to 50/s) and plug-and-play modifiability. |

---

## 3. Missing or Weak

1. **Missing Outbox persistence architecture:** `checkoutservice` is a stateless Go service in Online Boutique. The ADR parenthetically suggests "or publisher-confirmed channel with local disk fallback", but local container storage is ephemeral in Kubernetes. The dual-write failure mode (payment succeeds via `paymentservice`, but event publish fails before outbox persistence) is unaddressed.
2. **Weak backpressure and publisher-blocking safeguard:** The ADR lacks broker queue overflow rules (`x-max-length`, `x-overflow: drop-head` or reject-publish). If RabbitMQ hits resource limits during peak load, default publisher throttling will directly stall `checkoutservice.PlaceOrder`.
3. **Missing multi-replica rate limiting mechanics:** The ADR specifies that MailFast concurrency is limited to 50 requests/s, but does not define how multiple scaled pods of `emailservice` coordinate this rate limit without centralized throttling (e.g., Redis token bucket or single-pod worker).
4. **Omitted evaluation of Redis Streams:** Online Boutique already uses Redis (`cartservice`). For 200 msgs/s, Redis Streams could provide consumer groups and buffering with zero new infrastructure components.

---

## 4. Verdict

**Accept with changes**

The asynchronous pub/sub model via RabbitMQ is architecturally sound and necessary to satisfy the latency, availability, and modifiability drivers. However, the ADR must address three critical defects before implementation:

1. **Formalize Outbox / Reliable Publish persistence:** Replace the vague "local disk fallback" with a defined persistence strategy (e.g., dedicated PostgreSQL outbox or publisher confirms with an explicit non-blocking in-memory/durable queue with circuit breaker) to eliminate dual-write data loss.
2. **Define broker backpressure & queue overflow policy:** Configure explicit queue limits and non-blocking publisher timeouts in `checkoutservice` so broker saturation during an 11.11 burst never degrades checkout latency or availability.
3. **Specify rate-limiting implementation & operational deployment:** Detail how distributed `emailservice` workers enforce the aggregate 50 req/s MailFast limit, and document the Kubernetes deployment baseline (e.g., RabbitMQ Cluster Operator with 3-node Quorum Queues and storage PVCs).
