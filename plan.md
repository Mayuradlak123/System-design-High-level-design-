# System Design Prep — HLD + LLD (100 Questions)

A structured, progression-based preparation plan. **No solutions are written yet** — this
repo currently defines the *structure*, *method*, and *order*. Every question file is a
skeleton to be filled in by hand, one at a time.

Started: 2026-08-15

---

## 1. Mental Model

```
                    System Design
                         │
             ┌───────────┴───────────┐
             │                       │
            HLD                     LLD
             │                       │
      ┌──────┼──────┐          ┌─────┼─────┐
      │      │      │          │     │     │
   Scaling DB/Cache Queue     OOP  SOLID  Patterns
      │      │      │          │     │     │
      └──────┼──────┘          └─────┼─────┘
             │                       │
             └──────────┬────────────┘
                        ▼
                Production Thinking
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
   Concurrency      Failure          Consistency
        │               │                │
        ▼               ▼                ▼
   Idempotency       Retry          Transactions
```

The goal is not to memorize 100 answers. It is to internalize **one HLD checklist** and
**one LLD ladder** so any unseen question can be attacked the same way.

---

## 2. Repository Layout

```
plan.md                              ← this file (master plan + tracker)
templates/
  HLD-TEMPLATE.md                    ← the 12-point checklist
  LLD-TEMPLATE.md                    ← the 10-step ladder
01-core-hld-architecture-scalability/   Q1–Q10    HLD
02-database-and-data-modeling/          Q11–Q20   HLD
03-caching/                             Q21–Q30   HLD
04-queues-workers-async/                Q31–Q40   HLD
05-distributed-systems/                 Q41–Q50   HLD
06-payments-and-financial-systems/      Q51–Q60   HLD
07-ecommerce/                           Q61–Q70   HLD
08-social-media-realtime/               Q71–Q80   HLD
09-lld-object-oriented-design/          Q81–Q90   LLD
10-advanced-lld-production/             Q91–Q100  LLD
```

Each section folder contains:
- `README.md` — index table of its 10 questions with priority + status
- `NN-slug.md` — one skeleton file per question, pre-loaded with the right template

---

## 3. The HLD Checklist (answer all 12, every time, in order)

| # | Point | The failure mode if you skip it |
|---|-------|---------------------------------|
| 1 | Functional requirements | You design the wrong system |
| 2 | Non-functional requirements | No basis for any later trade-off |
| 3 | Expected traffic / QPS | Can't justify sharding or caching |
| 4 | Data volume | Can't pick a storage engine |
| 5 | APIs | Design stays vague and unimplementable |
| 6 | Database schema | Hand-waving gets exposed |
| 7 | SQL or NoSQL — and why | Reads as cargo-culting |
| 8 | Where cache is used | Miss the cheapest 10x win |
| 9 | Where queue/workers are used | Everything looks synchronous and fragile |
| 10 | What happens when something fails | The #1 senior-vs-junior separator |
| 11 | Concurrency / idempotency | Fatal in payments, inventory, messaging |
| 12 | How it scales horizontally | The whole point of the interview |

Full template: [`templates/HLD-TEMPLATE.md`](templates/HLD-TEMPLATE.md)

---

## 4. The LLD Ladder (walk top to bottom, every time)

```
Requirements → Entities → Classes → Interfaces → Relationships
→ SOLID → Design Patterns → Thread Safety → Error Handling → Extensibility
```

Full template: [`templates/LLD-TEMPLATE.md`](templates/LLD-TEMPLATE.md)

---

## 5. Priority — The 15 to Master First

If time is short, these carry the most interview surface area.

### HLD (11)
| Q | Question | File |
|---|----------|------|
| 1 | URL Shortener | [01-core-hld-architecture-scalability/01-url-shortener.md](01-core-hld-architecture-scalability/01-url-shortener.md) |
| 2 | Rate Limiter | [01-core-hld-architecture-scalability/02-api-rate-limiter.md](01-core-hld-architecture-scalability/02-api-rate-limiter.md) |
| 10 | Distributed ID Generator | [01-core-hld-architecture-scalability/10-distributed-id-generator.md](01-core-hld-architecture-scalability/10-distributed-id-generator.md) |
| 31 | Queue + Worker System | [04-queues-workers-async/31-background-job-system.md](04-queues-workers-async/31-background-job-system.md) |
| 35 | Kafka / Event Processing | [04-queues-workers-async/35-100m-events-per-day.md](04-queues-workers-async/35-100m-events-per-day.md) |
| 51 | Payment System | [06-payments-and-financial-systems/51-payment-gateway.md](06-payments-and-financial-systems/51-payment-gateway.md) |
| 62 | Inventory System | [07-ecommerce/62-inventory-management.md](07-ecommerce/62-inventory-management.md) |
| 73 | Chat System | [08-social-media-realtime/73-realtime-chat.md](08-social-media-realtime/73-realtime-chat.md) |
| 71 | News Feed / Timeline | [08-social-media-realtime/71-twitter-timeline.md](08-social-media-realtime/71-twitter-timeline.md) |
| 97 | File Storage / Upload | [10-advanced-lld-production/97-resumable-file-upload.md](10-advanced-lld-production/97-resumable-file-upload.md) |
| 66 | Search System | [07-ecommerce/66-product-search.md](07-ecommerce/66-product-search.md) |

### LLD (6)
| Q | Question | File |
|---|----------|------|
| 81 | Parking Lot | [09-lld-object-oriented-design/81-parking-lot.md](09-lld-object-oriented-design/81-parking-lot.md) |
| 82 | Elevator | [09-lld-object-oriented-design/82-elevator-system.md](09-lld-object-oriented-design/82-elevator-system.md) |
| 91 | Payment SDK | [10-advanced-lld-production/91-payment-sdk.md](10-advanced-lld-production/91-payment-sdk.md) |
| 92 | LRU Cache | [10-advanced-lld-production/92-thread-safe-lru-cache.md](10-advanced-lld-production/92-thread-safe-lru-cache.md) |
| 93 | Rate Limiter (thread-safe) | [10-advanced-lld-production/93-thread-safe-rate-limiter.md](10-advanced-lld-production/93-thread-safe-rate-limiter.md) |
| 94 | Job Scheduler | [10-advanced-lld-production/94-job-scheduler.md](10-advanced-lld-production/94-job-scheduler.md) |

---

## 6. Study Order (do NOT solve these randomly)

The sections are ordered so each one depends only on what came before.

| Phase | Weeks | Sections | Why here |
|-------|-------|----------|----------|
| **P0 — Priority sprint** | 1–2 | The 15 above | Fastest path to being interview-viable |
| **P1 — Foundations** | 3 | 01 Architecture, 02 Database | Every later design reuses these primitives |
| **P2 — Performance** | 4 | 03 Caching | Cheapest scaling lever; teaches invalidation + consistency |
| **P3 — Asynchrony** | 5 | 04 Queues & Workers | Unlocks decoupling, retries, DLQs, backpressure |
| **P4 — Distribution** | 6 | 05 Distributed Systems | Locks, ordering, partitions, multi-region |
| **P5 — Correctness under money** | 7 | 06 Payments | Where idempotency and ledgers become non-negotiable |
| **P6 — Applied HLD** | 8 | 07 E-Commerce, 08 Social/Real-Time | Composite designs built from P1–P5 |
| **P7 — LLD core** | 9 | 09 OOP Design | SOLID + patterns on small, closed problems |
| **P8 — LLD production** | 10 | 10 Advanced LLD | Thread safety, extensibility, resilience |
| **P9 — Consolidation** | 11–12 | Full re-pass, timed | Second pass from memory, no notes |

**Weekly cadence:** 5 questions/week (1/day, 45 min timed) + 2 revisits of older answers.

---

## 7. Working Method (per question)

1. **Timebox 45 minutes**, closed-book. Write the answer in the question file.
2. Answer the checklist/ladder **in order**. Do not skip to the architecture diagram.
3. State assumptions explicitly rather than asking for clarification forever.
4. Draw the diagram — text/ASCII is fine.
5. **Then** open references and mark what you missed in a `## Gaps` section.
6. Update `Status` and `Attempts` in the file header, and the section `README.md`.
7. Revisit after 3 days, then 2 weeks (spaced repetition). Target: `can-explain-in-45min`.

### Status ladder
`not-started` → `in-progress` → `drafted` → `reviewed` → `can-explain-in-45min`

---

## 8. Cross-Cutting Themes to Watch

These recur across sections; note each occurrence rather than re-deriving it.

- **Idempotency** — Q52, Q54, Q55, Q36, Q40, Q60, Q91
- **Concurrency / races** — Q46, Q63, Q67, Q92, Q93
- **Failure & recovery** — Q26, Q36, Q47, Q53, Q99
- **Consistency models** — Q17, Q27, Q48, Q49, Q58
- **Partitioning / sharding** — Q10, Q16, Q28, Q29, Q44
- **Retry & backoff** — Q37, Q39, Q60, Q99
- **Extensibility (OCP)** — Q95, Q96, Q90, Q100

---

## 9. Progress Tracker

| Section | Questions | Drafted | Reviewed | Mastered |
|---------|-----------|---------|----------|----------|
| 01 Core HLD — Architecture & Scalability | 10 | 2 | 0 | 0 |
| 02 Database & Data Modeling | 10 | 0 | 0 | 0 |
| 03 Caching | 10 | 2 | 0 | 0 |
| 04 Queues, Workers & Async | 10 | 0 | 0 | 0 |
| 05 Distributed Systems | 10 | 0 | 0 | 0 |
| 06 Payments & Financial Systems | 10 | 0 | 0 | 0 |
| 07 E-Commerce | 10 | 0 | 0 | 0 |
| 08 Social Media & Real-Time | 10 | 0 | 0 | 0 |
| 09 LLD / Object-Oriented Design | 10 | 0 | 0 | 0 |
| 10 Advanced LLD + Production | 10 | 0 | 0 | 0 |
| **Total** | **100** | **4** | **0** | **0** |
