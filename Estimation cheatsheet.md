# System Design Estimation Cheat Sheet

---

## 1. Time Conversions (memorize cold)

| Unit | Exact | Round to |
|---|---|---|
| Seconds/day | 86,400 | **100,000 (10^5)** |
| Seconds/month | ~2.6M | **3,000,000 (3×10^6)** |
| Seconds/year | ~31.5M | **30,000,000 (3×10^7)** |
| Days/month | 30 | 30 |
| Days/year | 365 | 365 |

**Rule:** QPS = (events per day) / 100,000

---

## 2. Byte Conversions

| Unit | Bytes | Notes |
|---|---|---|
| 1 KB | 10^3 | small text record |
| 1 MB | 10^6 | image / short doc |
| 1 GB | 10^9 | |
| 1 TB | 10^12 | |
| 1 PB | 10^15 | |
| 1 EB | 10^18 | |

**Unit multiplication shortcuts** (this is what trips people up):

| Operation | Result |
|---|---|
| KB × Thousand (10^3) | MB |
| KB × Million (10^6) | GB |
| KB × Billion (10^9) | TB |
| MB × Thousand | GB |
| MB × Million | TB |
| MB × Billion | PB |
| GB × Million | PB |
| GB × Billion | EB |

> Trick: add the exponents. KB=10^3, Million=10^6 → 10^9 = GB. Always sanity-check this way instead of counting zeros.

---

## 3. Default Payload Sizes (use unless problem specifies)

| Item | Size |
|---|---|
| Tweet / short text msg | ~280 bytes → round to **0.5 KB** |
| Chat message (text) | **1 KB** |
| DB row / metadata record | **1 KB** |
| URL (short + long + meta) | **500 bytes – 1 KB** |
| Thumbnail image | **50 KB** |
| Medium/profile image | **200–300 KB** |
| Full-res photo | **2–5 MB** (use 3MB) |
| 1 min compressed video (480p) | **~10–50 MB** (use 10MB for SD) |
| HD video (1 min) | **~100 MB** |
| Audio file (3 min song) | **~5 MB** |
| Average webpage | **~2 MB** |
| HTTP redirect response | **~500 bytes** |

---

## 4. Common Assumptions (state these out loud if not given)

| Parameter | Default Assumption |
|---|---|
| Peak traffic multiplier | **2× average** (sometimes 3-5x for spiky apps) |
| Replication factor | **3x** (for durability/fault tolerance) |
| Storage overhead (indexes, metadata) | **+20-30%** |
| Retention (logs/messages) | 30 days unless "permanent" stated |
| Retention (user content - photos/posts) | 5 years / "forever" |
| Read:Write ratio if unspecified | Guess based on product type (see §7) |
| Active users vs registered | DAU is usually **10-20%** of total registered users |
| Image storage | If "stored in N sizes," multiply by N (state this assumption) |

---

## 5. Core Formulas

### QPS
```
Total events/day = DAU × actions per user per day
Avg QPS = Total events/day / 100,000
Peak QPS = Avg QPS × 2
```

### Storage
```
Data/day = writes/day × size per record
Total storage = Data/day × retention period (days)
Apply replication/overhead multiplier at the end
```

### Bandwidth
```
Ingress (write) = write QPS × avg request payload size
Egress (read)  = read QPS × avg response payload size
```

### Caching / Hot Data (Pareto estimate)
```
If "80% of reads go to 20% of data" → cache only needs to hold ~20% of dataset
Cache size = 20% × total active dataset (not total historical storage)
```

### Servers Needed (rough)
```
Servers = Peak QPS / QPS-per-server-capacity
(assume 1 server handles ~1,000-10,000 simple QPS depending on workload)
```

---

## 6. Math Shortcuts (the actual arithmetic)

| Operation | How to do it fast |
|---|---|
| X / 100,000 | Move decimal 5 places left. `500M / 100K = 5,000` |
| X% of Y | **Multiply** by (X/100). Never divide by X. `20% of 80B = 0.2 × 80B = 16B` |
| Big number × big number | Separate coefficients and powers. `500M × 50 = (500×50) × M = 25,000M = 25B` |
| Weighted average | `(% × size) + (% × size)`, e.g. `0.8×1KB + 0.2×1MB ≈ 200KB` |
| Converting QPS → daily | Multiply by 100K (reverse of QPS formula) |

**#1 mistake to avoid:** "X% of Y" ≠ "Y / X". 20% of 80B is `0.2 × 80B = 16B`, NOT `80B / 20 = 4B`.

---

## 7. Read:Write Ratio → Architecture Decision

| Ratio | Pattern | Why |
|---|---|---|
| High (10:1 to 1000:1) | **Cache + CDN (pull)** | Same data read by many → amortize fetch cost |
| ~1:1 | **Queue + Push** | Each item consumed once → nothing to cache |
| Write-heavy (1:10 reverse) | **Batch/async writes, append-only logs** | e.g. analytics, IoT sensor ingestion |

| System type | Typical ratio |
|---|---|
| URL shortener | ~10:1 |
| Social media feed / photo host | ~50:1 to 1000:1 |
| Messaging (1:1 chat) | ~1:1 |
| Analytics/logging | reversed (write-heavy) |

---

## 8. The Universal Estimation Framework

```
1. Clarify scale: DAU, actions/user/day, payload sizes, retention
   → State assumptions explicitly if not given

2. Traffic:    Daily events → Avg QPS (÷100K) → Peak QPS (×2)
               Split into read QPS and write QPS

3. Storage:    Daily write volume × size/record × retention × overhead

4. Bandwidth:  QPS × payload size (separately for read/write)

5. Insight:    Read:Write ratio → caching/CDN/queue decision
               Biggest number → biggest bottleneck → design priority
```

---

## 9. Sanity-Check Reference Points

Use these to judge if your final number is "reasonable":

| Scale descriptor | Order of magnitude |
|---|---|
| Small startup | thousands of QPS, GB-TB storage |
| Medium product (millions of users) | 10K-100K QPS, TB-PB storage |
| Big tech (hundreds of millions - billions of users) | 100K-1M+ QPS, PB-EB storage |

