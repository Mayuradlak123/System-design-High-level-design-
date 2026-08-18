# Q101 — Redis primary down ho jaaye to? Design HA so no acknowledged write is lost.

**Track:** HLD | **Status:** `drafted`
**Reference:** [FinTrack Backend](https://github.com/Mayuradlak212/Fintrack-backend) — Flask + Redis Sentinel (`app/core/redis_ha/`, `docker/redis-ha/`, `REDIS_HA.md`)

> **Q26** poochhta hai *"Redis gir gaya to app kya kare?"* (fallback, degrade).
> Ye question ulta hai: *"Redis ko girne hi kyun diya?"* — topology, failover, aur write safety.

---

## The whole design in one picture

```
        ┌────────────┐ ┌────────────┐ ┌────────────┐
        │ sentinel-1 │ │ sentinel-2 │ │ sentinel-3 │
        └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
              │  quorum 2/3, majority 2/3   │
              └──────────────┼──────────────┘
                             │ monitor + promote
                             ▼
   App ──"primary kaun?"──►  ┌──────────────────┐
       ◄──redis-primary───   │     PRIMARY      │
       ──writes────────────► │ min-replicas-    │
                             │  to-write 1      │
                             └────────┬─────────┘
                          async replication (+ WAIT)
                          ┌───────────┴───────────┐
                          ▼                       ▼
                    ┌──────────┐            ┌──────────┐
                    │ REPLICA 1│            │ REPLICA 2│
                    └──────────┘            └──────────┘
```

**One line:** Sentinel batata hai primary kaun hai, `min-replicas-to-write` unsafe write lene se **rokta** hai, aur `WAIT` batata hai write replica tak **pahuncha ya nahi**.

---

## The problem

Single Redis = **SPOF**. Node gaya → poora cache gaya. Ye to obvious hai.

Asli problem chhupi hui hai — **replication async hai**:

```
App ──SET k v──► PRIMARY
                   │
                   ├──► "OK" ✅  (client ko turant)
                   │
                   └──► replica ko bhejna... (baad mein)
                              ↓
                        PRIMARY CRASH 💥
                              ↓
                   Replica promote hua — par uske paas `k` hai hi nahi
```

⚠️ **Client ko "OK" mil chuka tha.** App ne user ko success bol diya. Data gayab. Ye **silent** hai — koi error log nahi, koi alert nahi. Failover ke baad bas... key missing hai.

Idempotency key, session token, rate-limit counter — kuch bhi ho sakta hai.

---

## Fix 1 — Primary + Replica (availability)

Replicas primary se stream lete hain. Primary mara → ek replica promote.

| Setting | Value | Kyun |
|---|---|---|
| `replica-read-only yes` | on | Replica pe write = failover pe wo data **udd jayega**. Split-brain ka seed. |
| `repl-backlog-size 64mb` | bada | Chhota blip → **partial** resync. Chhota backlog → poora full resync (mehnga) |
| `repl-diskless-sync yes` | on | Har replica handshake pe disk pe RDB likhna avoid |

👉 **Config primary aur replica dono pe same rakho.** Replica ek din primary *banega* — jo safety setting sirf purane primary pe thi wo us waqt chupchaap gayab ho jayegi.

---

## Fix 2 — Sentinel (automatic failover)

Sentinel primary ko watch karta hai, mara to replica promote karta hai, aur clients ko naya address batata hai.

```
primary quiet  →  +sdown   (ek sentinel ko laga: down hai)
               →  +odown   (QUORUM sentinels agree — ab officially down)
               →  election (MAJORITY vote → ek leader chuna gaya)
               →  +switch-master  ✅
```

### Quorum ka asli maths (yahi pe log fasste hain)

**Do alag thresholds hain, dono satisfy hone chahiye:**

| Threshold | Kitna | Configurable? | Kaam |
|---|---|---|---|
| **quorum** | 2 | ✅ haan | Kitne sentinels agree karein ki primary down hai |
| **majority** | 2 of 3 | ❌ nahi (`floor(N/2)+1`) | Kitne vote dein failover *karne* ke liye |

```
sentinel monitor fintrack-primary redis-primary 6379 2
                                                    ↑ quorum
```

**3 sentinels, quorum 2** — dono threshold barabar ho gaye. Exactly **1 sentinel failure** tolerate hota hai.

| Agar quorum... | To kya hoga |
|---|---|
| `1` | Ek sentinel ka network kharab → healthy primary ko dead declare → **bewajah failover** |
| `2` ✅ | Sahi. Ek sentinel mar sakta hai, failover phir bhi chalega |
| `3` | Ek bhi sentinel gaya → failover **freeze**. Cluster theek chalta rahega... jab tak sach mein failover ki zarurat na pade |

⚠️ **2 sentinels HA nahi hai.** 2 ka majority bhi 2 hi hai → koi ek mara, bacha hua akela kisi ko promote **nahi** kar sakta. 3 minimum hai, aur **alag-alag machine/AZ pe** — ek hi host pe 3 sentinel sirf process crash se bachata hai, rack failure se nahi.

---

## Fix 3 — `min-replicas-to-write` (unsafe write ko rok do)

Ye wo half hai jo **write hone hi nahi deta** jab replica healthy nahi hai.

```conf
min-replicas-to-write 1     # itne replicas connected hone chahiye
min-replicas-max-lag  10    # aur itne second se zyada peeche nahi
```

Condition fail → primary write **reject** kar deta hai:

```
> SET anything value
(error) NOREPLICAS Not enough good replicas to write.
```

| | |
|---|---|
| ✅ Reads chalte rehte hain | Primary zinda hai, bas aisa data lene se mana kar raha hai jo replicate nahi kar sakta |
| ✅ Error **dikhta** hai | App retry kar sakti hai, alert baj sakta hai |
| ⚠️ Ye availability trade hai | 1 replica required + wo replica gaya = **writes band**. 3am pe pata chale, uske pehle decide karo |

👉 **2 replicas hain to `1` maango, `2` nahi.** `2` maangoge to ek replica ka simple restart bhi writes gira dega — availability ka nuksaan bada, durability ka faayda mamuli.

---

## Fix 4 — `WAIT` (write sach mein pahuncha?)

`min-replicas-to-write` batata hai *"replica maujood hai"*. `WAIT` batata hai *"**ye** write us tak pahunch gaya"*.

```
> SET critical:key value
OK
> WAIT 2 1000
(integer) 2      ← dono replicas ke paas hai ✅

> WAIT 5 1000
(integer) 2      ← 1 second baad. Jhooth nahi bolta — jo mila wahi count
```

```python
result = ha.critical_write(lambda c: c.set("session:abc", token, ex=900))
if not result.durable:
    log.warning("only %d/%d replicas acked", result.acked_replicas,
                result.required_replicas)
```

⚠️ **`WAIT` transaction NAHI hai.** Timeout ho gaya to write **already ho chuka hai** — rollback nahi hota. Wo sirf *verdict* deta hai: "is waqt tak ye durable nahi tha."

⚠️ **`WAIT` connection-scoped hai** — usi connection ke writes cover karta hai. Isliye write aur WAIT **ek hi client** pe chalne chahiye, pool se doosra connection uthाke nahi.

### Teen layer, teen alag kaam

| Layer | Karta kya hai | Kya **nahi** kar sakta |
|---|---|---|
| **Sentinel** | Dead primary detect + replica promote | Jo write replicate hua hi nahi, use wapas nahi la sakta |
| **`min-replicas-to-write`** | Unsafe write **accept hi nahi** karta | Ye nahi bata sakta ki koi ek specific write pahuncha ya nahi |
| **`WAIT`** | Ek specific write ka replica-ack confirm karta hai | Na pahunche to rollback nahi kar sakta |

👉 **Teeno milke window chhoti karte hain — band nahi karte.** Jo data kho hi nahi sakta (payments, ledger) wo **Postgres** mein jaata hai, Redis mein nahi.

---

## Fix 5 — Client failover handle kare

Sentinel ne promote kar diya — par app ke haath mein abhi bhi **purane node ka connection** hai.

```
App ──SET──► purana primary (ab demote ho chuka)
         ◄── -READONLY You can't write against a read only replica ❌
                          │
                          ▼
              connection pool DROP karo
                          │
                          ▼
              Sentinel se dobara poochho → naya primary → retry ✅
```

| Kya | Kyun |
|---|---|
| Sentinel se resolve karo, host hardcode mat karo | Failover pe redeploy nahi karna padega |
| `ReadOnlyError` pe **pool disconnect** | Purane pooled connections purane address pe hi jaate rahenge |
| Backoff ke saath 2–3 retry | Election `down-after-ms` + few sec leta hai |
| `fn` **idempotent** rakho | Retry hoga, isliye do baar chalne pe safe hona chahiye |

👉 Failover ka cost ab error nahi, bas **kuch sau millisecond extra latency** hai.

---

## Failure modes

| Failure | Blast radius | Recovery |
|---|---|---|
| Ek replica gaya | Kuch nahi. `min-replicas-to-write 1` abhi bhi satisfy | Wapas aake catch-up |
| **Dono** replicas gaye | Writes → `NOREPLICAS`. Reads theek | Replica aate hi writes chालू |
| Primary gaya | ~5s detect + election. Us window mein writes fail | Sentinel promote karta hai; purana node **replica banke** wapas judta hai (isliye split-brain nahi) |
| Ek sentinel gaya | Kuch nahi — 2/3 bache, majority intact | — |
| **Do** sentinels gaye | ☠️ Data nodes bilkul healthy dikhenge, par **automatic failover mar chuka hai** | Sentinel wapas lao |
| Replica lag > 10s | Primary writes reject karega | Lag ki wajah dhoondo (network/slow disk) |

⚠️ **Sabse khatarnak row wahi hai jisme kuch toota hua nahi dikhta** — 2 sentinels down. Har node green, dashboard green, traffic normal. Bas cluster ne primary failure se recover karne ki **kshamta** kho di hai. Isliye sentinel count ko data nodes se **alag** monitor karo.

```bash
redis-cli -p 26379 sentinel ckquorum fintrack-primary
# OK 3 usable Sentinels...        ✅
# NOQUORUM ... not enough available sentinels to reach the majority   ☠️
```

---

## Health check — kya expose karna hai

```
GET /api/health/redis
{
  "status": "ok",
  "primary":  { "role": "master", "connected_replicas": 2,
                "min_replicas_to_write": 1, "accepting_writes": true },
  "replicas": [ { "lag_seconds": 0, "lag_bytes": 0, "status": "ok" } ],
  "sentinels":[ { "quorum": 2, "known_sentinels": 3, "quorum_ok": true } ]
}
```

| Field | Kyun matter karta hai |
|---|---|
| `accepting_writes` | Primary UP hai par `NOREPLICAS` de raha ho — wo "healthy" nahi hai |
| `lag_seconds` | Isi pe `min-replicas-max-lag` judge hota hai |
| `lag_bytes` | **Abhi failover ho jaye to kitna data udega** — asli operational number |
| `quorum_ok` | Failover ki kshamta zinda hai ya nahi |

👉 **Degraded pe 200 do, sirf "no primary" pe 503.** Ek lagging replica ya ek dead sentinel pe page bhejoge to log page ignore karna seekh jayenge.

👉 App ka lag threshold server ke `min-replicas-max-lag` **se thoda kam** rakho — taaki health **pehle** warn kare, writes fail hone ke saath-saath nahi.

---

## Trade-offs & what I would NOT build

| Decision | Kyun |
|---|---|
| **Sentinel, not Redis Cluster** | Problem availability ki hai, capacity ki nahi. Cluster sharding deta hai + multi-key ops/Lua pe restrictions. Data ek node pe fit hai to Cluster sirf operational bojh hai. |
| **`WAIT`, not `WAITAOF`** | `WAITAOF` (7.0+) disk fsync ka intezaar karta hai — cache workload pe latency bahut. |
| **Nahi banaunga: apna custom failover** | Sentinel already solved hai. Khud likhoge to split-brain khud debug karoge. |
| **Nahi banaunga: multi-region replication** | Cross-region lag `min-replicas-max-lag` ko harega → writes reject. Alag problem, alag design ([Q28](28-multi-region-cache.md)). |
| **Redis ko source of truth nahi banaunga** | HA ≠ durability. Money/ledger → Postgres. |

---

## Interview answer (say this)

> Ek primary, do replicas, aur **teen** Sentinels **quorum 2** ke saath. Sentinel primary ko monitor karta hai aur failure pe replica promote karta hai; app Sentinel se primary resolve karta hai, isliye promotion ke liye redeploy nahi chahiye.
>
> Quorum 2 isliye kyunki do threshold hain — quorum (kitne agree karein ki down hai) aur majority `floor(N/2)+1` (kitne failover authorize karein). 3 sentinels pe dono 2 hote hain, matlab exactly ek sentinel failure tolerate hota hai. 2 sentinels HA nahi hai kyunki uska majority bhi 2 hi hai.
>
> Replication async hai, to acknowledged write ab bhi kho sakta hai. Isliye **do** taraf se cover karta hoon: primary pe `min-replicas-to-write 1` + `min-replicas-max-lag 10` — replica healthy na ho to write **accept hi nahi hota** (`NOREPLICAS`), aur critical writes pe client side `WAIT` — jab tak replicas ack na karein, write ko done nahi maanta.
>
> Ye window ko chhota karta hai, **band nahi karta** — Redis durable store nahi hai. Jo kho hi nahi sakta wo Postgres mein rakhta hoon.

---

## Follow-ups

| Question | Answer |
|---|---|
| Purana primary wapas aaya to fight karega? | Nahi — Sentinel use naye primary ka **replica** bana deta hai. Yahi split-brain rokta hai. |
| Failover ke waqt writes ka kya? | ~5s (`down-after-ms`) + election tak fail hote hain. Client retry with backoff karta hai; user ko latency dikhti hai, error nahi. |
| `WAIT` har write pe kyun nahi? | Har write pe network round-trip + replica ack ka intezaar. Sirf **critical** writes pe — session, idempotency key. Rate-limit counter pe nahi. |
| Failover pe kitna data khoyega? | Jo primary ne ack kiya par replicate nahi hua. `WAIT` + `min-replicas-to-write` se ye window chhoti; `lag_bytes` se measure hoti hai. |
| `min-replicas-to-write 2` kyun nahi? | 2 replicas hain — 2 maangoge to ek ka restart bhi writes gira dega. Availability ka nuksaan > durability ka faayda. |
| Sentinel vs Cluster? | Sentinel = availability (1 primary, saara data). Cluster = sharding (16384 slots). Data fit ho raha hai to Sentinel. |
| Sentinel down par Redis zinda — impact? | Data path bilkul theek. Bas **automatic failover nahi hoga**. Silent risk — isliye alag alert. |
| Test kaise karoge? | `docker stop redis-primary` → sentinel logs mein `+sdown → +odown → +switch-master`. Aur `docker stop` dono replicas → `NOREPLICAS`. |

---

## Self-Review
- [x] Problem pehle (async replication ka silent loss), solution baad mein
- [x] Quorum **aur** majority — dono, aur kaunsa configurable hai
- [x] Har config value justify ki (`1` vs `2` replicas, quorum `2` vs `1`/`3`)
- [x] `WAIT` ki limitation explicitly boli (rollback nahi, connection-scoped)
- [x] Availability trade-off maana (`min-replicas-to-write` writes gira sakta hai)
- [x] Scope control — Cluster, multi-region, custom failover: **nahi banaunga**
- [x] Sabse khatarnak failure mode identify kiya (majority gaya, sab green dikh raha hai)
