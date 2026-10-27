Antigravity (Gemini 3.8 Flash)

# ADR-0002: Use asynchronous event-driven publish/subscribe for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Architecture Team

## Context and problem

After an order is placed, Online Boutique must execute three distinct actions:
1. Send an order confirmation email via MailFast, a new external email provider replacing the mock `emailservice` logic.
2. Award loyalty points through a new `loyaltyservice` (to be built).
3. Record an "order placed" analytics event for the marketing team.

Furthermore, the marketing team expects to introduce additional post-order actions next year, specifically SMS notifications, fraud scoring, and a recommendations update.

We need to decide on an architectural pattern to coordinate these post-order actions so that adding future capabilities is straightforward and order processing remains fast and resilient, without making assumptions about infrastructure or performance targets that are not yet specified.

## Decision drivers

- **Modifiability / Extensibility:** The architecture must allow adding new post-order actions (SMS, fraud scoring, recommendations update) with minimal friction and without modifying existing core checkout logic.
- **Availability / Fault Isolation:** Failures or delays in downstream or external services (e.g., MailFast outages, analytics downtime) must not cause the customer checkout to fail.
- **Performance / Latency:** The end-user checkout experience must remain responsive and not block on external API calls or background processing.
- **Reliability:** All required post-order actions must eventually be executed reliably once an order is confirmed.
- **Operational Complexity:** The design should keep system operations and maintenance manageable.

## Considered options

1. **Synchronous orchestration via checkout service:** The checkout service sequentially or concurrently calls `emailservice` (integrating MailFast), `loyaltyservice`, and the analytics ingestion endpoint via synchronous RPC/HTTP requests before completing the checkout response.
2. **Asynchronous event-driven publish/subscribe (Pub/Sub):** Upon placing an order, the checkout service publishes an `OrderPlaced` event to an event/message broker. Downstream consumers (`emailservice` / MailFast adapter, `loyaltyservice`, and an analytics subscriber) independently subscribe to and process the event asynchronously.
3. **Asynchronous task queue with worker services:** The checkout service enqueues discrete background tasks into a central task queue (e.g., "send-email", "award-loyalty", "record-analytics"), which are picked up by specialized background worker processes.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option 1: Synchronous orchestration** | - Simple to implement initially with no new infrastructure (message broker or queue) required.<br>- Immediate consistency: caller immediately knows if each action succeeded.<br>- Easy to debug and trace using standard synchronous call stacks. | - Tight coupling: `checkoutservice` must know about every post-order service and change whenever actions are added (e.g., SMS, fraud, recommendations).<br>- Poor availability: Downtime or slowdown in MailFast or downstream services risks delaying or failing user checkout.<br>- Increased checkout latency: User waits for all downstream calls to finish. |
| **Option 2: Asynchronous event-driven (Pub/Sub)** | - Loose coupling: `checkoutservice` only publishes `OrderPlaced`; new subscribers (SMS, fraud, recommendations) can be added without modifying checkout.<br>- High availability: Failure or latency in MailFast or loyalty processing has zero impact on checkout success.<br>- Low user latency: Checkout returns immediately after persisting order and publishing event.<br>- Independent scalability and rate regulation for external services like MailFast. | - Introduces additional infrastructure (message broker) to manage and monitor.<br>- Eventual consistency: Actions occur asynchronously, requiring distributed tracing and monitoring.<br>- Requires handling of duplicate delivery (idempotency) and dead-letter queues. |
| **Option 3: Asynchronous task queue** | - Decouples checkout latency from task execution.<br>- Provides built-in retry and rate limiting per task type.<br>- Easier failure management per individual task. | - `checkoutservice` remains coupled to the specific list of tasks to generate, requiring code changes whenever new actions (e.g., SMS, fraud scoring) are added.<br>- Requires managing queue infrastructure and task dispatching logic. |

## Decision

We choose **Option 2: Asynchronous event-driven publish/subscribe (Pub/Sub)**.

Once an order is placed, the checkout service publishes an `OrderPlaced` event containing relevant order details to a message broker. Independent consumer services subscribe to this event to perform their respective duties:
- An email worker / service consumes the event and dispatches the confirmation email via MailFast.
- The new `loyaltyservice` consumes the event to calculate and award loyalty points.
- An analytics consumer records the event for the marketing team.

This option best satisfies our primary decision drivers:
- **Modifiability:** It adheres to the Open-Closed Principle. When the marketing team adds SMS, fraud scoring, and recommendations update next year, new consumers can simply subscribe to `OrderPlaced` without any modifications to `checkoutservice`.
- **Availability and Performance:** Isolating external calls (MailFast) and downstream processing prevents third-party latency or failures from impacting checkout completion for the customer.

## Consequences

- **Positive:**
  - **Modifiability:** New post-order capabilities (SMS, fraud scoring, recommendations update) can be added as independent event subscribers without modifying or redeploying `checkoutservice` (quality attribute: modifiability).
  - **Availability:** Outages or degradation in MailFast, `loyaltyservice`, or analytics will not impede customer checkout completions (quality attribute: availability).
  - **Performance:** Checkout request-response time to the customer remains minimal, as it does not wait for downstream network I/O or third-party email API calls (quality attribute: performance).
  - **Scalability:** Downstream event processing can be scaled independently or throttled to match third-party rate limits (e.g., MailFast quotas) without backpressure on checkout (quality attribute: scalability).

- **Negative:**
  - **Operational Complexity / Cost:** Requires introducing, deploying, monitoring, and operating a message broker in the system (quality attribute: operability, cost).
  - **Observability / Testability:** Asynchronous processing increases debugging difficulty and requires distributed tracing across asynchronous boundaries (quality attribute: testability, observability).
  - **Reliability (Idempotency requirement):** Consumers must implement idempotent event processing to protect against duplicate event delivery (e.g., preventing duplicate emails or loyalty points) (quality attribute: reliability).

- **Risks and mitigations:**
  - **Risk:** Message loss if the broker fails or if an event is published before/after order transaction commits.
    - *Mitigation:* Use persistent message storage, broker clustering, and consider an Outbox pattern for publishing if transactional guarantees are required.
  - **Risk:** MailFast external API outage, network error, or rate limiting causes email delivery failures.
    - *Mitigation:* Implement automated retries with exponential backoff and dead-letter queues (DLQ) for failed email notifications.
  - **Risk:** Duplicate message delivery causing duplicate loyalty points or multiple emails.
    - *Mitigation:* Require consumers to use the unique `order_id` as an idempotency key before awarding points or sending emails.

## Validation

- **Functional validation:** Execute end-to-end integration tests verifying that publishing an `OrderPlaced` event results in MailFast dispatching the confirmation email, `loyaltyservice` recording the points, and the analytics event being logged.
- **Latency benchmark:** Run a load test on checkout verifying that p99 checkout response times are decoupled from MailFast response times and downstream service latencies.
- **Resilience testing:** Simulate an outage or latency spike on MailFast and `loyaltyservice` to ensure user checkout transactions complete successfully without failure.
- **Extensibility review:** When adding the upcoming post-order actions next year (e.g., SMS, fraud scoring), verify that they can be deployed as new event subscribers with zero code changes to `checkoutservice`.

## Open questions

Because operational metrics, team constraints, and detailed non-functional requirements were not provided in the specification, the following questions remain open:
1. **Load and throughput:** What is the expected order volume (average and peak orders per second), and what throughput must the message broker handle?
2. **Latency targets:** What are the acceptable latency thresholds for post-order actions (e.g., must confirmation emails arrive within seconds, and can loyalty points be credited within minutes)?
3. **MailFast specifications:** What are MailFast's rate limits, authentication model, retry policies, and SLA guarantees?
4. **Team skills and infrastructure preferences:** What message broker technologies (e.g., RabbitMQ, Apache Kafka, NATS, GCP Pub/Sub, AWS SNS/SQS) align with the team's operational expertise and hosting environment?
5. **Consistency requirements for loyalty points:** Does the customer expect their updated loyalty balance immediately upon viewing the post-checkout page, or is eventual consistency acceptable?
6. **Fraud scoring workflow:** Does the upcoming fraud scoring action require synchronous pre-authorization/rejection before order fulfillment, or is it purely asynchronous post-order auditing?
