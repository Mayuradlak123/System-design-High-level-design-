# Q2 — Design a rate limiter for an API used by millions of users

**Track:** HLD | **Priority:** **TOP 15** | **Status:** `drafted`
**Reference:** `finance tracker/backend/app/core/rate_limit/`

---

## The whole design in one picture

```
Request
   ↓
Identity     user_id ho to user, warna IP
   ↓
Policy       api:read → 300/min, burst 60
   ↓
Bucket key   sha256(policy | identity)
   ↓
L1: in-process dict  ~1 µs   (always on)
L2: Redis + Lua      ~0.5 ms (shared across all servers)
   ↓
tokens >= cost ?
   ├── YES → debit → handler → RateLimit-* headers
   └── NO  → 429 + Retry-After (jittered)
```

## 1–4. Requirements & Scale

| | |
|---|---|
| **Functional** | Per-user/per-IP limits, different limits per endpoint, tell client remaining quota |
| **Non-functional** | < 1 ms added latency, approximate OK, **limiter down ≠ API down** |
| **QPS** | = total API QPS (runs on *every* request) → must be **1 Redis round trip** |
| **Data** | 2 numbers per caller (`tokens`, `last_refill`) → 10M users ≈ **1 GB**. Needs TTL. |

## 5. API — a decorator, not a service

```python
@jwt_required()            # identity pehle
@rate_limit("api:write")   # phir limit charge
def create_transaction(): ...
```

```
RateLimit-Limit: 60      RateLimit-Remaining: 43
RateLimit-Reset: 0       RateLimit-Policy: 60;w=60;burst=20
```
Reject → `429` + `Retry-After` + JSON body.

## 6. Policy table (all limits in ONE place)

| Policy | Limit | Scope | Fail closed |
|---|---|---|---|
| `auth:login` | 10 / 15 min, burst 5 | IP | ✅ |
| `auth:login_account` | 15 / 15 min, burst 6 | target email | ✅ |
| `api:read` | 300 / min, burst 60 | user or IP | — |
| `api:write` | 60 / min, burst 20 | user or IP | — |
| `api:report` | 300 / min, **cost 3** | user or IP | — |

💡 **Requests are priced, not counted.** Summary endpoint zyada rows scan karta hai → 3 tokens same read budget se, apni alag policy nahi.

💡 **Login pe 2 policies:** per-**IP** (ek host, bahut passwords) + per-**email** (botnet, 1000 IPs, ek account). Neither covers the other.

## 7. Which algorithm?

| Algorithm | Problem |
|---|---|
| Fixed window | Boundary pe **2× limit** nikal jaata hai |
| Sliding window log | Har timestamp store → **O(N) memory** per caller |
| **Token bucket** ✅ | 2 numbers, lazily refilled, burst-friendly |

```
capacity = 60          ← burst (kitna bank kar sakte ho)
refill   = limit/window

request → elapsed se refill → tokens >= cost ? debit : reject
```

Burst **jaanbujh kar** allowed — dashboard 5 calls ek saath bhejta hai. Lekin login pe burst kam, kyunki wahan burst hi abuse hai.

## 8. Why atomic (Redis Lua)?

❌ `GET` → compute → `SET` from app:
```
Worker A: GET → 1        Worker B: GET → 1    ← same value padh liya
Worker A: allow, SET 0   Worker B: allow      ← quota leak 😵
```
✅ Poora check-and-debit **ek Lua script** mein. Redis single-threaded → atomic. **Ek round trip.**

## 9. Two tiers

| Tier | Latency | Role |
|---|---|---|
| **L1** in-process | ~1 µs | Free, always on, obvious abuse pakadta hai |
| **L2** Redis | ~0.5 ms | Shared across all workers/instances |

⚠️ L1 alone: 4 workers → effective limit **4×**. Production mein Redis zaroori.

## 10. Failure posture

| Failure | Behaviour |
|---|---|
| Redis down | **Fail OPEN** → L1 pe giro. Limiter jo API 503 kar de, wo abuse se bada outage hai. |
| Login / reset / MFA | **Fail CLOSED** → reject. Unmetered login attempt khatarnak hai. |
| Redis slow | 50 ms timeout + **circuit breaker** (5 fails → 5 sec off), warna har request 50 ms extra khaati hai |

👉 **Fail-open by default, fail-closed by exception** — aur exception list chhoti rakho.

## 11. Correctness details

- Key = `sha256(policy|identity)` → IPs/emails Redis keyspace mein plaintext nahi
- Multiple policies per route → headers mein **tightest** (lowest remaining) bhejo
- `Retry-After` **jittered** → sabko same number = sab ek hi ms pe wapas = self-inflicted thundering herd

## 12. Scaling

- App stateless, Redis holds state → servers freely add karo
- Redis cluster: bucket key se shard, natural spread
- **Multi-region:** per-region limiter. Global counter ke liye har request ocean paar bhejna latency se bada nuksaan hai.

---

## Client side (usually forgotten — say this)

Frontend headers padhta hai aur per-policy **cooldown** rakhta hai; 429 ke baad cooldown khatam hone tak request bhejta hi nahi.

> Retrying only spends quota the user needs for their **next real action**.

⚠️ Token refresh pe 429 = **logout nahi**. Throttled refresh session invalid hone ka signal nahi hai.

## Follow-ups

| Question | Answer |
|---|---|
| IP spoofing? | `X-Forwarded-For` sirf proxy ke peeche trust karo, warna caller apna bucket choose kar lega |
| NAT, 1000 users 1 IP? | Logged-in ko `user_id` pe key karo; IP sirf anonymous fallback |
| Gateway ya app? | Gateway = coarse (per-IP DDoS), app = fine (per-user policy). Dono. |
| Paid tiers? | Policy user's plan se resolve karo, hardcoded name se nahi |
| 429 vs 503? | 429 = tumne limit todi. 503 = hum down hain. Never mix. |

## Self-Review
- [x] 12 points covered
- [x] 3 algorithms compared with each one's failure mode
- [x] Race condition traced, atomicity justified
- [x] Failure posture both ways + client side
