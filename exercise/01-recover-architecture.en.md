**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI Evidence | Confirmed? ✅ / ❌ / ❓ | Findings |
|---|---|:---:|---|
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | Verified in `src/checkoutservice/main.go`: maps env at L116, dials gRPC at L123, and `PlaceOrder` calls `chargeCard()` at L252. |
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | Verified in `src/checkoutservice/main.go`: dials gRPC at L120, calls `getUserCart()` at L290, and calls `emptyUserCart()` at L263 after payment succeeds. |
| `frontend` → `checkoutservice` | `frontend.yaml` (`CHECKOUT_SERVICE_ADDR`) | ✅ | Verified in `src/frontend/main.go` (L136, L146) and `src/frontend/handlers.go` (L354): `placeOrderHandler` directly invokes `PlaceOrder` gRPC endpoint. |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR`) | ✅ | Verified in `src/cartservice/src/cartstore/RedisCartStore.cs` (L24-50): C# .NET service using `IDistributedCache` to persist and retrieve cart items by `userId`. |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | Verified in `src/recommendationservice/recommendation_server.py`: reads env at L131 and calls `product_catalog_stub.ListProducts()` at L73 to recommend products. |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): …
1. `shoppingassistantservice` is missing from the diagram despite being declared in `frontend.yaml` and actively called via HTTP POST in `frontend/handlers.go` (L464), because it lacked a deployment manifest.
2. `loadgenerator` is placed inside the System_Boundary as if it were an internal business service, whereas it is actually a client-side synthetic load test tool (Locust).
3. Other persistence mechanisms are invisible: e.g., `productcatalogservice` reads a bundled products.json file, which isn't reflected, giving the impression that no other service stores data.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

    A `.proto` file is strictly an interface specification that defines what methods and payloads a callee service exposes. It contains no caller-side logic, no entry points (web UI/gateway), and no database connections. Recovering the actual runtime call graph requires deployment manifests (*_ADDR env vars) or caller source code.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

    Assuming `paymentservice` and `shippingservice` connect to real external third-party providers (e.g., Stripe, Visa, FedEx). The diagram suggests real payment/shipping integrations, whereas the actual codebase uses dummy mock functions that simply generate fake IDs in-memory.