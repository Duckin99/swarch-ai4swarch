**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Ashira Aungsumal | defends the ADR |
| **AI analyst** | Kittichon Chaonawig | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Peeravas Piboolvorakul | role card P (business side) |
| **Site reliability engineer (SRE)** | Kittichet Arayasujin | role card S (operations side) |
| **Security & privacy officer** | Pasin Thanyakasikol | role card X (security and privacy) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-events.md` by Ashira Aungsumal  

**Board members and roles:** Architect (Ashira), AI analyst (Kittichon), Product owner (Peeravas), SRE (Kittichet), Security & privacy officer (Pasin)  

**Decision:** accept with changes (unanimous vote among all 5 members; AI analyst read the verdict from the AI review: "Accept with changes"). Status reverts to `Proposed` until all revisions are addressed.  

**Required changes:**
1. **Policy on publication failure:** Orders must not fail. Introduce a fallback or reconciliation mechanism so paid orders still receive confirmation emails and loyalty points. Define bounded publish timeouts and test publish-failure edge cases after payment.
2. **Sizing and bounds for email processing:** Business must agree on a maximum acceptable latency ceiling; size consumers for 50 msgs/s at 1.5s latency; establish queue length ceilings and backlog alerting; run realistic peak load tests; ensure email consumers deduplicate by `order_id`.
3. **HA design and data governance:** Specify queue type and replica count, TLS, service-specific credentials, minimize payload fields to only what each consumer needs, define DLQ retention, owners, alerting, and replay procedures, assign on-call ownership and runbooks, prepare rollback plans, and confirm Data Processing Agreement (DPA) with MailFast prior to go-live.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Publish failure after payment; order must succeed and needs fallback mechanism | AI review (F4); PO (PO2) | Yes in F4, but lacked business cost impact on customer loyalty | Technical gap: Yes, visible from ADR. Business cost: No, requires membership commitments and business cost data. |
| 2 | Unbounded email backlog drains in ~90 mins with no agreed max latency ceiling | AI review (F5, F10); PO (PO1); SRE (SRE2) | Yes in F5, F10, but omitted customer impact (duplicate orders, support load) | Numbers: Yes, derivable from ADR. Customer behavior: No, requires product/sales domain knowledge. |
| 3 | Duplicate emails because email consumer is not idempotent and provider charges per email | PO (PO3) | No (F2 only covered loyalty deduplication) | Duplicate emails: Plausibly derivable from ADR. Per-email pricing: No, provider billing model was not in inputs . |
| 4 | MailFast is a 3rd party receiving customer PII: contracts, storage location, retention | Security (SEC2) | No | Could only know MailFast receives emails. Contracts, GDPR/PDPA, and retention were outside ADR and scenario. |
| 5 | Broker memory/disk alarm blocks publishers and stalls `PlaceOrder` | SRE (SRE2) | Partially (F9 confirm timeout, F5 growth), but lacked broker flow-control blocking | Uncertain; requires real-world production RabbitMQ operational experience . |
| 6 | No on-call ownership, runbooks, node failure drills, or rollback plan before 11.11 | SRE (SRE1, SRE3) | Partially (F12 DLQ ownership, F11 stateful), but omitted operational processes | No, team internal operating procedures were not in the provided evidence. |
| 7 | Event broadcasts full order details to all consumers; DLQ retains PII | AI review (F7); Security (SEC1, SEC3) | In F7 & Change #3, but omitted TLS, credential isolation, and retention policies | Data fields: Yes. Security configurations: Could observe ADR was silent, but details were unknowable. |
| 8 | Single email consumer is a single point of failure | AI review (F6) | Yes | Yes, derivable directly from the architecture and numbers in ADR. |

**The AI's best finding** (one the humans had missed): F5 and F6, which translated raw ADR numbers into a concrete ~90-minute drain time and ~75 pending email dispatches. The architect had only vaguely described buffering, and PO/SRE adopted these calculations directly.
 
**The AI's weakest finding** (generic, wrong or irrelevant): F12, which was an unprioritized laundry list of "missing items" (alternate brokers, outbox specifics, peak duration). Also F3 ("2 of 4 developers know RabbitMQ") was flagged as a non-risk by AI, but SRE argued that basic familiarity is far from being production on-call capable .

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
