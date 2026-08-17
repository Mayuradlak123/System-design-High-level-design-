# Q24 — How would you prevent cache stampede?

**Track:** HLD | **Status:** `drafted`

---

## The problem

Ek **popular** key expire hui → saari requests ek saath DB pe.

```
1000 requests
      ↓
    Redis
      ↓
 Key expired ❌
      ↓
1000 requests → DB 😵
```

**Jo chahiye tha:**
```
Request 1 → DB → cache update
Request 2 ─┐
Request 3 ─┤→ wait / use same result
Request 4 ─┘
```

Isko **thundering herd** bhi kehte hain.

⚠️ **Khatra kya hai:** DB is key ke liye 1 QPS pe theek tha. Achanak 1000 QPS **same query** ka. Slow → timeout → retry → aur load → DB dead. **Ek expired key poora system gira sakti hai.**

---

## Fix 1 — Single-flight / distributed lock

Sirf **ek** request recompute karegi. Baaki wait karengi.

```
Request 1 → MISS → Lock acquire ✅ → DB → Cache SET → release
Request 2 → MISS → Lock exists ❌ → wait/retry
Request 3 → MISS → Lock exists ❌ → wait/retry
                                        ↓
                                  Cache HIT ✅
```

**Redis mein:**
```
SET lock:key <random-value> NX EX 10
```

| Part | Kyun |
|---|---|
| `NX` | Set **only if not exists** → atomic, ek hi winner |
| `EX 10` | Auto-expire. Winner crash ho gaya to lock stuck na rahe. |
| random value | Release pe apni value hi delete karo — dusre ka lock na hatao |

**Waiters ke 2 options:** wait+retry (simple, par requests atki rehti hain) ya **stale data serve** (better UX).

---

## Fix 2 — TTL jitter

10,000 keys same time pe same TTL se cache hui → sab **ek hi second** pe expire → mass stampede.

```
❌  TTL = 300
✅  TTL = 300 + random(0, 30)
```

Expiry spread ho jaati hai. **Ek line ka code, bada faayda.**

---

## Fix 3 — Stale-while-revalidate

Value pe do timer: **fresh until** aur **hard expiry**.

```
Request → value stale hai par dead nahi
        → stale value turant return ✅
        → background mein refresh
```

**Koi wait nahi karta.** Cache kabhi khaali hi nahi hota, so stampede karne ke liye miss hi nahi milta.

---

## "1 million requests aa gaye to?"

Haan, request #1 lock leti hai — par baaki 999,999 **queue mein nahi jaati**. Yahi asli point hai.

⚠️ **1M requests ko ek Redis lock pe ladana bhi galat hai** — pileup DB se Redis pe shift ho gaya, bas. Har waiter ek thread, connection, socket hold kar raha hai.

**Sahi answer — do layer, sasti pehle:**

```
1,000,000 requests
  → single-flight per server   →  100 requests   (in-memory, free)
  → distributed lock in Redis  →    1 request    (one network hop)
  → DB                         →    1 query ✅
```

Single-flight requests ko **app process ke andar** collapse karta hai — zero network calls. 100 servers ho to 1M → 100, Redis touch hone se pehle. Phir lock 100 → 1.

Aur hot keys pe **stale-while-revalidate** → 999,999 ko purani value turant, 1 background refresh. **Nobody blocks.**

---

## Which one to use?

| Situation | Use |
|---|---|
| Ek bahut hot key (homepage, product page) | **Lock** |
| Bahut keys saath cache hui (bulk warm-up) | **TTL jitter** |
| Stale data acceptable hai | **Stale-while-revalidate** |

👉 Practice mein: **jitter + lock saath**. Jitter mass event rokta hai, lock jo bach jaaye usko.

---

## Interview answer (say this)

> Use **single-flight / distributed locking** so only one request recomputes a missing value, while others wait or serve stale data. Add **TTL jitter** so keys don't all expire at the same instant.
>
> At very high concurrency I'd avoid making a million requests contend on one Redis lock — **coalesce in-process first**, then the distributed lock, and **serve stale data** so nobody blocks.

## Follow-ups

| Question | Answer |
|---|---|
| Lock holder crash? | Lock ka TTL hai → auto-release, others retry |
| DB query lock TTL se slow? | Lock extend karo (heartbeat), ya rare duplicate accept karo |
| Requests ko wait karana bura nahi? | Haan — block karne se better hai stale serve karo |
| Stampede vs penetration? | **Stampede** = key thi, expire hui. **Penetration** = key kabhi thi hi nahi (Bloom filter → [Q1](../01-core-hld-architecture-scalability/01-url-shortener.md)) |

## Self-Review
- [x] Problem pehle, solution baad mein
- [x] Exact Redis command with each flag justified
- [x] Lock failure cases covered
- [x] Stampede vs penetration distinguished
