# Q1 — Design a URL Shortener like Bitly

**Section:** Core HLD — Architecture & Scalability | **Track:** HLD | **Priority:** **TOP 15**
**Status:** `drafted`
**Reference implementation:** [SnipURL](https://github.com/Mayuradlak123/URL-shortner-system-design) — Flask + MongoDB + Redis

---

## The whole design in one picture

```
CREATE                                READ (redirect)
──────                                ───────────────
POST /shorten                         GET /abc1234
   │                                     │
   ▼                                     ▼
random 7-char code               1. Redis cache?  ──hit──► 302 ✅
   │                                     │ miss
   ▼                                     ▼
Bloom filter: seen before?       2. Bloom says "never seen"? ──► 404 ✅
   │ maybe          │ no                 │ maybe exists
   ▼                │                    ▼
check MongoDB       │                3. MongoDB ──► cache it ──► 302 ✅
   │ collision      │
   └──► retry       ▼
              insert into MongoDB
                    │
                    ▼
          add to Bloom + cache
```

**One line:** short code → look up long URL → 302 redirect. Everything else is making that lookup cheap.

---

## 1. Functional Requirements

- Shorten a long URL → get a short link
- Open short link → redirect to original
- Track clicks (analytics)
- Show a user their recent links

**Out of scope:** custom aliases, expiry dates, user accounts.

## 2. Non-Functional Requirements

| Need | Target |
|---|---|
| Redirect latency | < 50 ms |
| Availability | 99.9% (a dead link = a dead business) |
| Read:write ratio | ~100:1 (heavily read-dominated) |
| Consistency | Eventual is fine — a link 1 sec late is OK |

**Key insight:** reads dominate → optimize the read path, everything else can be slow.

## 3. Traffic / QPS

```
100M new URLs/day  →  100M / 86400  ≈ 1,150 writes/sec
10B redirects/day  →  reads = 100 × writes ≈ 115,000 reads/sec
Peak = 2× average  →  ~230,000 reads/sec
```

## 4. Data Volume

| Item | Size | Per year |
|---|---|---|
| 1 URL row (~500 B) | 100M/day | ~18 TB/year |
| 1 visit row (~200 B) | 10B/day | huge → **must expire** |

👉 Fix: **TTL index on visits (90 days)**. Without it, analytics data eats the whole disk.
*(`app/__init__.py` — `expireAfterSeconds`)*

## 5. APIs

```http
POST /api/v1/data/shorten
  { "longUrl": "https://example.com/very/long" }
  → 201 { "shortUrl": "https://sni.pt/abc1234" }

GET /abc1234
  → 302 Location: https://example.com/very/long
```

Redirect uses **302 (temporary)**, not 301 — a 301 is cached by the browser forever, so you'd never see the click again. **302 = you keep your analytics.**

## 6. Database Schema

**`urls`**
| Field | Index |
|---|---|
| `short_code` | **unique** ← the safety net for collisions |
| `long_url` | — |
| `user_ip` | compound with `created_at` (for history) |
| `created_at` | |

**`visits`**
| Field | Index |
|---|---|
| `short_code` + `visited_at` | for per-link analytics |
| `visited_at` | **TTL 90 days** |

## 7. SQL or NoSQL?

**NoSQL (MongoDB).** Why:
- Access pattern is a single key lookup: `short_code → long_url`. No joins needed.
- Needs easy horizontal sharding by `short_code`.
- Schema barely changes.

*SQL would work fine too — the deciding factor is sharding ease, not the data itself.*

## 8. Caching

| What | Where | TTL |
|---|---|---|
| `short_code → long_url` | Redis | 24 h |

- Mappings are **immutable** → cache invalidation is a non-problem. Huge win.
- Cache-aside: miss → read DB → write cache.
- Redis policy = `volatile-lru` so **only TTL'd keys get evicted** — the Bloom bitset and visit queue must never be evicted.

## 9. Queues & Workers

Analytics must never slow a redirect down.

```
redirect → RPUSH visit to Redis list → return 302   (~2 Redis ops, 0 DB writes)
                    │
background worker ──┴─► pops 500 → one bulk insert into MongoDB
```
*(`app/services/visit_service.py`)*

## 10. What happens when things fail?

| Failure | Behaviour |
|---|---|
| **Redis down** | Cache miss → read MongoDB. Slower, still correct. ✅ |
| **Bloom unavailable** | Returns "maybe exists" → falls through to DB. **Fails safe.** ✅ |
| **MongoDB primary dies** | Replica set elects new primary, driver follows automatically ✅ |
| **Bulk insert fails** | Visits pushed back to head of the Redis queue, retried ✅ |
| **Worker killed mid-batch** | That batch of clicks is lost — acceptable for analytics ⚠️ |

**Rule used everywhere: degrade, don't break.** A slow redirect beats a broken one.

## 11. Concurrency & Idempotency

**Problem:** two requests generate the same random code at the same time.

**Three layers of defence:**
1. Keyspace is huge → 62⁷ ≈ 3.5 trillion. At 10M URLs, ~14 collisions *total*.
2. Bloom filter catches "probably taken" without touching the DB.
3. **Unique index on `short_code`** — the real guarantee. The DB rejects the duplicate; retry.

👉 The first two are *optimizations*. Only #3 is *correctness*.

## 12. Horizontal Scaling

- **App tier is stateless** → add servers behind a load balancer freely.
- **MongoDB:** shard by `short_code` (random → even spread, no hot shard).
- **Redis:** cluster, sharded by key.
- **Reads:** add replicas — reads scale independently of writes.

---

## Why a Bloom filter? (the interesting bit)

A scanner hitting random codes (`/aaaaaa1`, `/aaaaaa2`, …) would miss the cache **every time** and hammer MongoDB with useless queries.

A Bloom filter is a tiny bitset that answers **"have I definitely never seen this?"**

| Answer | Trust it? |
|---|---|
| "Definitely not present" | ✅ 100% reliable → return 404 instantly, no DB hit |
| "Maybe present" | ❌ could be a false positive → check DB |

**12.5 MB of Redis holds ~10M codes.** It absorbs the entire attack.

⚠️ **The catch:** a "definitely absent" answer is only safe once the filter contains *every* existing code. So the filter is seeded from the DB at startup, and the check stays **switched off** until seeding completes. Otherwise you'd 404 real, live URLs.

---

## Trade-offs I made

| Decision | Chose | Instead of | Why |
|---|---|---|---|
| Code generation | Random + check | Counter + Base62 | Unguessable; but costs 1 read per write |
| Redirect status | 302 | 301 | Keeps analytics |
| Click tracking | Async queue | Inline DB write | Redirect stays fast |
| Visit retention | 90-day TTL | Keep forever | Would grow ~730 GB/year |

**Better for scale:** counter + **Feistel permutation** → no uniqueness read at all, codes still look random.

---

## Follow-ups to expect

1. *"How do you handle custom aliases?"* → separate path, still guarded by the unique index.
2. *"How do you stop malicious links?"* → validate scheme (block `javascript:`), check Safe Browsing.
3. *"Analytics in real time?"* → Kafka + stream processor instead of a Redis list.
4. *"Same URL shortened twice?"* → index `long_url` and return the existing code (a product decision, not a technical one).

## Known gaps in the current implementation

1. `long_url` isn't validated → open-redirect / phishing risk
2. `X-Forwarded-For` is trusted → history is spoofable
3. Visit queue has no size cap → grows unbounded if Mongo stalls

---

## Self-Review
- [x] Answered all 12 checklist points
- [x] Numbers estimated, not hand-waved
- [x] Stated consistency model explicitly
- [x] Named what I would NOT build (custom aliases, accounts, expiry)
