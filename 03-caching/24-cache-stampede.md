# Q24 — How would you prevent cache stampede?

**Section:** Caching | **Track:** HLD
**Status:** `drafted`

---

## What is a cache stampede?

When one **popular** key expires or misses, **all** the requests for it hit the database at the same moment.

```
1000 requests
      ↓
    Redis
      ↓
 Key expired ❌
      ↓
1000 requests → DB 😵
```

**What we actually want:**

```
Request 1 → DB → cache update
Request 2 ─┐
Request 3 ─┤→ wait / use same result
Request 4 ─┘
```

Also called the **thundering herd** problem.

**Why it's dangerous:** the DB was fine at 1 QPS for this key. Suddenly it gets 1000 QPS of the *same* query. It slows down → requests time out → they retry → even more load → the DB dies. One expired key can take down the whole system.

---

## Fix 1 — Single-flight / distributed lock (main answer)

Only **one** request is allowed to recompute. Everyone else waits.

```
Request 1 → Cache MISS → Lock acquire ✅ → DB
Request 2 → Cache MISS → Lock exists ❌ → wait
Request 3 → Cache MISS → Lock exists ❌ → wait
Request 4 → Cache MISS → Lock exists ❌ → wait

Request 1 → Cache SET
              ↓
       Others get cached value
```

**In Redis:**

```
SET lock:key <random-value> NX EX 10
```

| Part | Why it's there |
|---|---|
| `NX` | Set **only if not exists** → makes it atomic. Only one winner. |
| `EX 10` | Auto-expire. If the winner crashes, the lock doesn't stay stuck forever. |
| random value | On release, delete only if the value is yours — so you never delete someone else's lock. |

**The waiters have 2 choices:**
- **Wait + retry** (sleep ~50 ms, check cache again) — simple, but requests are held up
- **Serve stale data** — better UX, needs the old value kept around

---

## Fix 2 — TTL jitter

If 10,000 keys are all cached at the same time with the same TTL, they all expire **at the same second** → mass stampede.

```
❌  TTL = 300
✅  TTL = 300 + random(0, 30)
```

Expiries spread out instead of firing together. **One line of code, huge win.**

---

## Fix 3 — Early / probabilistic refresh

Don't wait for the key to actually expire. Refresh it *slightly before*, in the background.

```
TTL 300s ──────────────────────────────►
                          ▲
                          │ at ~270s, one request refreshes it
                          │ in the background
                     cache never
                     actually goes empty
```

The cache is never empty, so there is no miss to stampede on.

---

## Fix 4 — Stale-while-revalidate

Keep two timers on the value: **fresh until** and **hard expiry**.

```
Request → value is stale but not dead
        → return the stale value immediately ✅
        → refresh in the background
```

Nobody ever waits. Best for data where "a few seconds old" is acceptable.

---

## Which one to use?

| Situation | Use |
|---|---|
| One very hot key (product page, homepage) | **Lock (single-flight)** |
| Many keys cached together (bulk warm-up) | **TTL jitter** |
| Expensive query, must always be fast | **Early refresh** |
| Stale data is acceptable | **Stale-while-revalidate** |

👉 In practice: **jitter + lock together**. Jitter prevents the mass event, the lock handles whatever still slips through.

---

## Interview answer (say this)

> Use **single-flight / distributed locking** so only one request recomputes a missing value, while the other requests either wait or serve stale data. Add **TTL jitter** so keys don't all expire at the same instant. For very hot keys, refresh **before** expiry in the background so the cache is never empty.

---

## Follow-ups to expect

| Question | Answer |
|---|---|
| *"What if the lock holder crashes?"* | Lock has a TTL → auto-releases. Others retry. |
| *"What if the DB query is slower than the lock TTL?"* | A second request gets in. Either extend the lock (heartbeat) or accept the rare duplicate. |
| *"Isn't making requests wait bad?"* | Yes — serve stale data instead of blocking, when correctness allows. |
| *"Cache stampede vs cache penetration?"* | **Stampede** = key exists but expired. **Penetration** = key never existed (fix with a Bloom filter — see [Q1](../01-core-hld-architecture-scalability/01-url-shortener.md)). |

---

## Self-Review
- [x] Explained the problem before the solution
- [x] Gave the exact Redis command, with each flag justified
- [x] Covered lock failure cases
- [x] Distinguished stampede from penetration
