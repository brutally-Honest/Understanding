# Cache Strategies

## Caching at Different Places

A single request can pass through multiple caches before it ever touches a database.

```
Browser → CDN Edge → Reverse Proxy → App (in-process) → Distributed (Redis) → DB
  │           │             │              │                   │
HTTP        static       full HTTP      hot objects        shared state
cache       assets       responses      (this process)     (all processes)
```

Request can short-circuit at any arrow — if it hits at the browser, nothing downstream even fires.

| Place | Tool / Mechanism | What's cached | Who controls it |
|---|---|---|---|
| Browser | HTTP cache headers | Pages, JS, CSS, images | Server response headers |
| CDN | Cloudflare, Akamai, Fastly | Static assets, sometimes full API responses | TTL/purge rules |
| Reverse Proxy | Nginx, Varnish | Full HTTP responses | Proxy config |
| Application (in-process) | JS Map, Caffeine, Guava | Hot objects, computed values | Your app code |
| Distributed | Redis, Memcached | Shared data across all app instances | Your app code + cache config |
| Database | Buffer pool, query cache, materialized views | Disk pages, query results | DB engine |

### In-Memory: Application vs Dedicated

The "in-memory cache" name hides two very different things.

| | Application (in-process) | Dedicated (Redis/Memcached) |
|---|---|---|
| Lives in | Your app's own heap | Separate process/server |
| Shared across instances? | No — each instance has its own copy | Yes — one shared source |
| Access cost | Nanoseconds (same process) | Network round trip (still fast, ~ms) |
| Survives app restart? | No | Yes (Redis can persist; cache itself outlives any one app instance) |
| Risk | N instances → N different cached values | Single point all instances depend on |
| Use when | Per-instance hot data, no consistency need across instances | Shared truth needed across horizontally scaled app |

### Database Caching

Not something you build — it's the DB engine's own internal cache layer.

| Mechanism | What it holds |
|---|---|
| Buffer pool | Recently read/written disk pages (InnoDB, Postgres shared_buffers) |
| Query cache | Full query result for an identical query (deprecated in MySQL 8.0 — too easily invalidated) |
| Materialized view | Precomputed query result, refreshed on a schedule, not on every read |

### HTTP Caching

Controlled by response headers — no application code involved.

| Directive | Decides |
|---|---|
| `max-age=N` | Browser trusts response for N seconds |
| `s-maxage=N` | Same, but only for CDN/proxy, not browser |
| `no-cache` | Cache it, but always revalidate first |
| `no-store` | Never cache, anywhere |
| `public` / `private` | Shared caches (CDN) allowed? Or browser-only? |
| `immutable` | Never revalidate — content literally can't change |

Revalidation (ETag), not re-download:

```
1st request:  Browser ──GET──────────────▶ Server
              Browser ◀──200 + ETag:"abc"── Server

2nd request:  Browser ──GET, If-None-Match:"abc"──▶ Server
              unchanged → Browser ◀──304 (no body)── Server
              changed   → Browser ◀──200 + new ETag── Server
```

304 = bandwidth saved, correctness kept.

---

## Eviction Policies

What to remove when the cache is full.

```
[front/MRU] A  B  C  D [back/LRU]

access B  →   B  A  C  D        (move to front)
cache full →  evict D            (from the back)
```

| Policy | Tracks | Weakness |
|---|---|---|
| FIFO | Insertion order | Ignores access pattern entirely |
| LRU | Recency | One big scan evicts everything hot ("cache pollution") |
| LFU | Frequency | Old hot keys resist eviction even after going cold ("inertia") |
| MRU | Recency, inverted | Only correct for scan-heavy workloads |
| Random | Nothing | Surprisingly competitive; avoids LRU's pathological cases |
| ARC | Recency + frequency, adaptive | Self-tuning, but complex to implement |
| W-TinyLFU | Recency + frequency, sketch-based | Best real-world hit rates (Caffeine, Redis option) |

---

## Read Strategies

```
Cache-Aside:        App ──miss──▶ App fetches DB ──▶ App writes Cache ──▶ returns
Read-Through:       App ──miss──▶ Cache fetches DB itself ──▶ Cache stores ──▶ returns
```

| | Cache-Aside | Read-Through |
|---|---|---|
| Who fetches on miss | Application | Cache layer |
| Code complexity | Higher, explicit | Lower, one interface |
| Control | Full | Less |
| Most common in practice | Yes | — |

---

## Write Strategies

```
Write request
     │
     ├─ Write-Through ─▶ Cache (sync) ─▶ DB (sync) ─▶ ack
     │
     ├─ Write-Behind  ─▶ Cache (sync) ─▶ ack
     │                       └──(async)──▶ DB
     │
     └─ Write-Around  ─▶ DB (sync) ─▶ ack          [cache untouched]
```

| Strategy | Write latency | Freshness | Data loss risk | Best for |
|---|---|---|---|---|
| Write-Through | High (2x) | Always fresh | None | Read-heavy, consistency-critical |
| Write-Behind | Low | Eventually consistent | High if crash before flush | Write-heavy, loss-tolerant (counters, metrics) |
| Write-Around | Medium | Stale until next read | None | Write-once, rarely-read data |

---

## Invalidation Strategies

| Strategy | Freshness | Effort | Risk |
|---|---|---|---|
| TTL | Bounded by timer | Very low | None — but staleness window guaranteed |
| Event-driven delete | Near-real-time | Medium | Miss one write path → stale forever |
| Write-through | Immediate | Medium | Low |
| Key versioning (`user:1234:v7`) | Immediate | Medium-high | None — old key just ages out via TTL |
