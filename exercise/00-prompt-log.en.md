**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Kittichon Chaonawig 6731305421
**Agent(s) used:** Antigravity
**Model(s):** Gemini 3.8 Flash
**Plan / tier:** Gemini Pro plan

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Antigravity (Gemini 3.8 Flash) | Follow prompts/1-recover-architecture.md run A | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | Yes | 3 | Accurately extracted gRPC interface definitions, but correctly identified zero runtime dependencies due to lack of caller information in IDL. |
| 2 | 1-B | Antigravity (Gemini 3.8 Flash) | Follow prompts/1-recover-architecture.md run B | `prompts/1-recover-architecture.md`, `demo.proto`, and 13 YAML manifests in `kubernetes-manifests/` | Yes | 4 | Successfully mapped 17 dependencies from env vars; missed `shoppingassistantservice` which lacked a manifest, and included test tooling (`loadgenerator`) as a container. |
| 3 | 2-a | Antigravity (Gemini 3.8 Flash) | Follow prompts/2-write-adr.md run a context: none | `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes | 3 | Generated a sound general pub/sub pattern, but had zero concrete numbers; correctly flagged load, SLAs, and broker choice under Open questions. |
| 4 | 2-b | Antigravity (Gemini 3.8 Flash) | Follow prompts/2-write-adr.md run b context: scenario, adr-0001, checkoutservice | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-...`, `checkoutservice/main.go` | Yes | 5 | Highly specific ADR: tailored directly to RabbitMQ, calculated p95 latency impact (~305 ms), addressed 200/s peak vs 50/s MailFast rate limit, and handled loyalty idempotency. |
| 5 | 3 | Antigravity (Gemini 3.8 Flash) | Follow prompts/3-review-adr.md adr: docs/adr/0002-use-rabbitmq-for-post-order-processing.md | `prompts/3-review-adr.md`, `docs/adr/0002-...`, `docs/scenario.md` | Yes | 5 | Outstanding ATAM review: surfaced deep architectural risks regarding ephemeral outbox storage on K8s, broker publisher-blocking under queue saturation, and missed Redis Streams. |
| 6 | | | | | | | |
| 7 | | | | | | | |
