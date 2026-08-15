# Q23 — Cache invalidation for frequently changing product prices

**Section:** Caching | **Track:** HLD
**Status:** `drafted`

---

## The problem

Price cache mein ₹999 hai, DB mein ₹1200 ho gaya. User ko purana price dikhta hai → wrong order, revenue loss.

```
Product #123, price ₹999

Cache:
  product:123            → product data
  product:123:price      → ₹999
  search:iphone          → results containing #123
  recommendations:123    → results containing #123
```

**Note:** one price change ka effect **4 keys** par hai — product, price, search results, recommendations. Yehi asli problem hai.

---

## Solution 1 — Event-driven invalidation (the core answer)

DB is the source of truth. Har price change ek event publish karta hai.

```
DB UPDATE product (price = ₹1200)
        ↓
ProductPriceUpdated { product_id: 123, version: 8, new_price: 1200 }
        ↓
Cache Invalidation Worker
        ↓
Redis → invalidate / update affected keys
```

**Better: invalidate ki jagah update karo.** Consumer directly `SET product:123 → ₹1200` kar de, to stale window almost zero ho jaata hai (delete karoge to next request ko DB hit karna padega).

---

## Solution 2 — Dependency index (don't scan Redis!)

❌ **Galat approach:** `KEYS product:123:*` chala ke saari related keys dhundhna. Redis pe `KEYS` production mein blocking hai, aur `search:iphone` to us pattern mein aayega hi nahi.

✅ **Sahi approach:** likhte waqt hi record karo ki kaun si keys is product par depend karti hain.

```
product:123
     ↓
┌─────────────────────────────┐
│  dependent cache keys       │
├─────────────────────────────┤
│  product:123                │
│  product:123:price          │
│  search:iphone:page:1       │
│  recommendation:user:456    │
└─────────────────────────────┘

PriceUpdated(123) → look up index → invalidate exactly these keys
```

**Alternative — versioned keys** (no invalidation at all):

```
product:123:v7   ← price ₹999
       ↓ price changes, version bumps
product:123:v8   ← price ₹1200
```

New requests automatically `v8` padhte hain. Purani key khud TTL se expire ho jaayegi. **Delete karne ki zarurat hi nahi** — isliye race condition bhi nahi.

---

## Solution 3 — Transactional Outbox (reliability)

**The real production bug:** DB update ho gaya, lekin event publish fail ho gaya (broker down / process crash). Ab cache **hamesha ke liye** stale hai. 😵

Fix: event ko **usi transaction** mein likho.

```
DB Transaction
 ├── UPDATE product SET price = 1200
 └── INSERT into outbox (ProductPriceUpdated)
          ↓
       COMMIT      ← dono saath, ya dono nahi
          ↓
   Event Publisher (outbox se padhta hai)
          ↓
    Cache Consumer
          ↓
        Redis
```

Ab price update aur invalidation event **atomically** linked hain. Ek ho gaya to doosra guaranteed hoga.

---

## Interview answer (say this)

> For frequently changing prices I'd make the **database the source of truth** and publish a `ProductPriceUpdated` event on every change. A consumer **updates or invalidates** the affected cache entries. For derived caches like search results, I'd maintain an **explicit dependency index** rather than scanning Redis. And I'd use a **transactional outbox** so the invalidation event can never be lost if the broker is down.

---

## Follow-ups to expect

| Question | Answer |
|---|---|
| *"Event out of order aa gaya?"* | Version number check karo — `if event.version < cached.version: ignore`. |
| *"Consumer lag kar raha hai?"* | Price keys pe **short TTL (30–60s)** rakho as a safety net. Event fast path hai, TTL backstop. |
| *"Checkout pe stale price?"* | Checkout par **kabhi cache trust mat karo** — price hamesha DB se padho. Cache sirf browsing ke liye. |
| *"Multi-region cache?"* | Event ko har region ke consumer tak broadcast karo. |

💡 **Golden rule:** browsing = cache OK, paying = DB only.

---

## Self-Review
- [x] Explained why one change affects many keys
- [x] Rejected the naive approach (Redis scan) with a reason
- [x] Covered the failure case (lost event → outbox)
- [x] Named where cache must NOT be used (checkout)
