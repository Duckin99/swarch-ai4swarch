**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criteria | Run a | Run b |
|---|---|---|
| **Uses scenario numbers?** | **None**; strictly flagged missing latency, load, and SLA numbers under Open questions. | **Fully utilized:** cites p95 ≤ 500 ms, 200 orders/s peak, MailFast 1.5s p95 / 8s p99 latency, and 50 emails/s quota limit. |
| **Considered options** | 1. Synchronous orchestration<br>2. Generic Async Pub/Sub<br>3. Async task queue with workers | 1. In-memory Go goroutines in `checkoutservice`<br>2. Self-hosted RabbitMQ on Kubernetes<br>3. Apache Kafka |
| **Handles loyalty "exactly once" & MailFast rate limits?** | High-level conceptual recommendations only (suggests idempotency keys and generic DLQs). | **Concrete solution:** buffers 200/s peak in durable queues drained at ≤ 50 emails/s to MailFast; enforces exactly-once loyalty points via database `order_id` unique constraints within 60s. |
| **Hallucinated / invented facts?** | **None**; followed strict instructions not to invent numbers. | **None**; strictly adhered to constraints and team skills from scenario. |   

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | Change from AI draft | Rationale |
|:---:|---|---|
| 1 | **Standardized on Protobuf serialization for `OrderPlaced` events** | The AI draft left message format generic; Protobuf ensures schema evolution and type-safety consistent with our gRPC stack. |
| 2 | **Elevated Transactional Outbox Pattern to core decision** | Prevents message loss in edge cases where RabbitMQ is temporarily unreachable immediately after payment succeeds. |
| 3 | **Resolved Open Questions on persistence and DLQ policy** | Specified PostgreSQL for `loyaltyservice` to ensure ACID unique constraints on `order_id`, and added a 48h DLQ alert policy. |
