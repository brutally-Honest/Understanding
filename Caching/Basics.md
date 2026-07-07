# Cache Basics

## Core Concept

A cache = a faster, smaller copy sitting between a slow source and the requester.

```
        ┌─────────────┐
        │ Request key │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │ In cache?   │
        └──┬───────┬──┘
       yes │       │ no
           │       │
     ┌─────▼──┐  ┌─▼──────────────┐
     │ return │  │ fetch source   │
     │ value  │  │ of truth       │
     └────────┘  └───────┬────────┘
                         │
                  ┌──────▼──────┐
                  │ store (TTL) │
                  └──────┬──────┘
                         │
                  ┌──────▼──────┐
                  │ return value│
                  └─────────────┘
```

Every cache, at every layer, runs this same loop.

| Layer | Lives in | Example | Note |
|---|---|---|---|
| CPU L1/L2/L3 | Inside chip | SRAM | hardware, you don't touch it |
| OS Page Cache | Kernel RAM | Linux page cache | "100% RAM used" ≠ problem |
| In-process | App heap | Caffeine, JS Map | private to one process |
| Distributed | Cache cluster | Redis, Memcached | shared across instances |
| Reverse Proxy | In front of app | Nginx, Varnish | caches full HTTP responses |
| CDN Edge | Different city/country | Cloudflare, Akamai | closest to the user |
| Browser | User's device | HTTP cache | controlled by response headers |

Speed and capacity trade off going down this table. Top rows: you don't control them. Bottom rows: configured via headers/CDN rules. Middle (in-process, distributed): the layer you actually design.

---

## History — Time vs Implementation

| Era | System | Trigger (one line) |
|---|---|---|
| 1960s | CPU cache (SRAM) | CPU outran RAM speed |
| 1970s | OS page cache | Disk 1000x slower than RAM |
| ~1995 | HTTP / browser cache | Web traffic re-downloaded same assets |
| 1998 | CDN edge cache | Distance itself became the bottleneck |
| 2003 | Memcached | One DB couldn't serve millions of reads |
| 2009 | Redis | Memcached too dumb — no structures, no atomic ops |

Pattern across every row: *X got too slow relative to Y → faster smaller copy inserted in between.* Only the layer changes.

---

## Terminologies

| Term | Means | Catch |
|---|---|---|
| Hit | Key found in cache | — |
| Miss | Key not found, must fetch source | — |
| Hit Rate | hits / (hits + misses) | <80–90% → cache too small, TTL too short, or data isn't repeat-accessed |
| TTL | Time before entry auto-expires | Sets how long you're willing to be wrong |
| Eviction | Entry removed for space | Space-triggered, not time-triggered — different from TTL |
| Staleness | Cached value ≠ current truth | The tax paid for speed; never zero |
| Working Set | Data actually being accessed now | Cache only needs to fit this, not the whole dataset |
| Hot Key | One key, disproportionate traffic | Can saturate a single cache node |
| Cache Churn | Constant eviction before reuse | Symptom: cache too small for working set |
| Cold Start | Empty cache after restart/deploy | All misses until warmed — DB absorbs full load |

