# Q1 — Design a URL Shortener like Bitly

**Track:** HLD | **Priority:** **TOP 15** | **Status:** `drafted`
**Reference:** [SnipURL](https://github.com/Mayuradlak123/URL-shortner-system-design) — Flask + MongoDB + Redis

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
Bloom: seen before?              2. Bloom "never seen"? ──► 404 ✅
   │ maybe        │ no                   │ maybe exists
   ▼              │                      ▼
check MongoDB     │                  3. MongoDB ──► cache it ──► 302 ✅
   │ collision    │
   └──► retry     ▼
            insert into MongoDB → add to Bloom + cache
```

**One line:** short code → long URL lookup → 302. Baaki sab us lookup ko sasta banane ke liye hai.

## 1–4. Requirements & Scale

| | |
|---|---|
| **Functional** | Shorten, redirect, click tracking, user's recent links |
| **Out of scope** | Custom aliases, expiry, user accounts |
| **Non-functional** | < 50 ms redirect, 99.9% uptime, eventual consistency OK |
| **QPS** | 100M writes/day ≈ **1,150/sec** · reads 100× ≈ **115k/sec** (peak 230k) |
| **Data** | URLs ~18 TB/year · visits 10B/day → **must expire (TTL 90d)** |

👉 **Key insight:** reads dominate 100:1 → read path optimize karo, baaki sab slow chal sakta hai.

## 5. APIs

```http
POST /api/v1/data/shorten   { "longUrl": "https://..." }
  → 201 { "shortUrl": "https://sni.pt/abc1234" }

GET /abc1234  → 302 Location: https://...
```

⚠️ **302, not 301.** 301 browser mein forever cache hota hai → click dobara dikhega hi nahi. **302 = analytics bachi rehti hai.**

## 6. Schema

**`urls`** — `short_code` (**unique index**), `long_url`, `user_ip`, `created_at`
→ compound index `(user_ip, created_at)` for history

**`visits`** — `short_code`, `ip`, `user_agent`, `visited_at`
→ index `(short_code, visited_at)` + **TTL 90 days** on `visited_at`

TTL zaroori hai: 10M redirects/day = ~2 GB/day = **~730 GB/year** warna.

## 7. SQL or NoSQL?

**NoSQL (MongoDB).** Access pattern ek single key lookup hai (`short_code → long_url`), koi join nahi, schema barely changes, aur `short_code` se sharding easy. *SQL bhi chal jaata — deciding factor sharding ease hai, data nahi.*

## 8. Caching

`short_code → long_url` in Redis, **TTL 24 h**, cache-aside.

💡 **Mappings immutable hain** → cache invalidation ki problem hi nahi. Huge win.

Redis policy `volatile-lru` — sirf TTL'd keys evict ho, kyunki Bloom bitset aur visit queue **kabhi** evict nahi hone chahiye.

## 9. Queues & Workers

Analytics redirect ko slow nahi kar sakti.

```
redirect → RPUSH visit to Redis list → return 302   (~2 Redis ops, 0 DB writes)
                     ↓
   background worker pops 500 → ek bulk insert
```

## 10. Failure modes

| Failure | Behaviour |
|---|---|
| Redis down | Cache miss → MongoDB. Slow, par correct ✅ |
| Bloom unavailable | "maybe exists" → DB pe fall through. **Fails safe** ✅ |
| MongoDB primary dead | Replica set election, driver follow karta hai ✅ |
| Bulk insert fail | Visits queue ke head pe wapas, retry ✅ |
| Worker killed mid-batch | Us batch ke clicks lost — analytics ke liye acceptable ⚠️ |

👉 **Rule: degrade, don't break.** Slow redirect >> broken redirect.

## 11. Concurrency & Idempotency

**Problem:** do requests same random code generate kar de.

**Teen layer:**
1. Keyspace huge — 62⁷ ≈ **3.5 trillion**. 10M URLs pe ~14 collisions *total*.
2. Bloom filter "probably taken" DB touch kiye bina pakadta hai
3. **Unique index on `short_code`** — DB duplicate reject karta hai, retry karo

👉 Pehle do **optimization** hain. Sirf #3 **correctness** hai.

## 12. Scaling

- App tier **stateless** → load balancer ke peeche servers add karo
- MongoDB: shard by `short_code` (random → even spread, no hot shard)
- Redis cluster, key se sharded
- Reads: replicas add karo — reads writes se independent scale karti hain

---

## Bloom filter kyun? (the interesting bit)

Scanner random codes maar raha hai (`/aaaaaa1`, `/aaaaaa2`…) → **har baar** cache miss → MongoDB pe useless queries ki barsaat.

Bloom filter ek chhota bitset hai jo batata hai: **"ye maine kabhi dekha hi nahi?"**

| Answer | Trust? |
|---|---|
| "Definitely not present" | ✅ 100% reliable → instant 404, no DB hit |
| "Maybe present" | ❌ false positive ho sakta → DB check karo |

**12.5 MB Redis mein ~10M codes.** Poora attack absorb kar leta hai.

⚠️ **Catch:** "definitely absent" tabhi safe hai jab filter mein **saare** existing codes ho. Isliye startup pe DB se seed hota hai, aur seeding poori hone tak check **band** rehta hai — warna live URLs pe 404 aa jaayega.

## Trade-offs

| Decision | Chose | Instead of | Why |
|---|---|---|---|
| Code generation | Random + check | Counter + Base62 | Unguessable; par 1 read per write |
| Redirect | 302 | 301 | Analytics bachti hai |
| Click tracking | Async queue | Inline DB write | Redirect fast rehta hai |
| Visit retention | 90-day TTL | Forever | ~730 GB/year warna |

**Better for scale:** counter + **Feistel permutation** → uniqueness read hi nahi, aur codes random dikhte hain.

## Follow-ups

| Question | Answer |
|---|---|
| Custom aliases? | Separate path, unique index still guards it |
| Malicious links? | Scheme validate karo (`javascript:` block), Safe Browsing check |
| Real-time analytics? | Redis list ki jagah Kafka + stream processor |
| Same URL do baar? | `long_url` index karo, existing code return — product decision hai |

**Known gaps:** `long_url` validate nahi hota (open-redirect risk) · `X-Forwarded-For` trusted (history spoofable) · visit queue pe size cap nahi.
