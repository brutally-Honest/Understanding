# Cache Distributed

## Why Distributed

```
Local cache (no coordination):        Distributed cache (shared truth):

App1 [cache: stale]                   App1 ─┐
App2 [cache: fresh]   → diverge       App2 ─┼──▶  Shared Cache
App3 [cache: empty]                   App3 ─┘
```

N instances, N independent local caches → N different answers to the same question. A distributed cache makes "shared state" the whole point: one write, every reader sees it — at the cost of a network hop per access.

Analogy: local cache is N people each keeping their own notebook of "what the boss said." Distributed cache is one shared whiteboard — slower to walk over to, but nobody's working off outdated notes.

---

## Consistent Hashing

Naive approach — `hash(key) % N` — breaks the moment N changes.

```
Ring (0 ──────────────────────── 2^32, wraps to 0)

    A(10)      B(90)      C(200)     D(300)
      │          │           │          │
key=50 ──▶ B (next node clockwise)
key=150 ──▶ C
key=250 ──▶ D

Remove B  →  only keys 11–90 remap to C
             A's keys, C's keys, D's keys: untouched
```

| | `hash % N` | Consistent hashing |
|---|---|---|
| Add/remove a node | Remaps almost every key | Remaps only adjacent keys |
| Effect | Mass simultaneous miss (self-inflicted avalanche) | Localized, contained remap |

Analogy: `hash % N` is like renumbering every locker in the building the moment one locker bank is added — everyone's stuff moves. Consistent hashing only reshuffles the lockers physically next to the new bank.

**Virtual nodes (vnodes):** one physical node = many positions on the ring → distribution stays even even with few physical nodes. Without vnodes, a node landing on a bad spot on the ring can get a disproportionate slice of keys — vnodes average that out by giving each physical node many small, scattered slices instead of one big one.

---

## Replication

| Role | Handles | Trade-off |
|---|---|---|
| Primary | Writes | Single point of truth |
| Replica | Reads (offloaded) | Replication lag = staleness window |

**Replication modes:**

| Mode | Primary waits for replica ack? | Risk |
|---|---|---|
| Async | No | Replica can lag; data loss on primary crash before replication |
| Semi-sync | Waits for ≥1 replica | Slower writes, safer |
| Sync | Waits for all replicas | Defeats the purpose of a fast cache |

**Failover (general pattern):**

```
Primary down →  Monitoring processes (odd number, so a majority is always possible) vote
             →  Majority agrees primary is down
             →  Promote a replica to primary
             →  Other replicas re-point to new primary
```

Split-brain risk: if the network partitions and the old primary comes back while a new one was promoted, you can briefly have two primaries accepting writes. This is why the voting group is odd-numbered — ties are what let split-brain happen in the first place.

---

## Cache Coherence — CAP and Quorums

Distributed caching is a live CAP trade-off, not a theory exercise.

| Model | N (copies) | W (write quorum) | R (read quorum) | Guarantee |
|---|---|---|---|---|
| Strong | 3 | 3 | 1 | Every read sees latest write |
| Quorum (Dynamo-style) | 3 | 2 | 2 | W+R > N → read-your-write guaranteed |
| Eventual (typical cache default) | 3 | 1 | 1 | Fastest, no ordering guarantee |

Most distributed caches lean **AP**: during a network partition, the majority side keeps serving; the minority side may serve stale data or reject writes — there's no built-in quorum write across replicas by default. That's a deliberate trade: a cache prioritizing strict consistency over availability has mostly defeated the purpose of being a cache.

**When you actually need strong consistency on cached data:** write-through + route all reads through the primary only. You're explicitly giving up horizontal read scaling to get it — there's no shortcut.

---

## Distributed Locking

The general problem: multiple processes, no shared memory, need mutual exclusion anyway.

```
Distributed mutex, general shape:

1. Client requests a lock with a TTL, from a set of independent lock-holders
2. Lock counts as "acquired" only if a majority of holders agree in time
3. TTL auto-expires the lock — if the holder crashes, the lock isn't stuck forever
4. Known weakness: clock drift across nodes can break the timing guarantee
```

Why majority, not just one lock-holder: a single lock-holder is a single point of failure — if it dies mid-lock, you either wait forever or fail unsafe. Requiring majority agreement means the lock survives any one holder going down, at the cost of needing multiple independent holders to ask in the first place.

| | Single-node lock | Quorum-based distributed lock |
|---|---|---|
| Survives one node failing | No | Yes (majority still agrees) |
| Complexity | Low | Higher — needs multiple independent lock-holders |
| Common use | App-level mutex (e.g. stampede fix) | Cross-service critical sections |

---

## Failure Patterns

### Penetration

**What happens:** a request asks for a key that doesn't exist in the cache *or* the database — a bad ID, an enumeration attack probing sequential IDs, or a bug generating malformed keys. Because it's genuinely absent everywhere, every single request for it walks straight past the cache and slams the DB. Unlike a normal miss, there's no "warm it up and move on" — the DB gets hit again, and again, for a key that will never resolve.

**Why it's dangerous specifically:** normal cache misses are self-healing (miss once, cache it, done). Penetration misses are *not* self-healing, because there's nothing to cache — so a burst of bad-ID requests (accidental or an attacker deliberately probing `user/1`, `user/2`, `user/3`...) turns your cache layer transparent and your DB takes 100% of that traffic.

**Fixes:**
- Cache the null result too (with a short TTL) — now the second request for the same bad key hits cache, not DB.
- Bloom filter in front of the cache: a probabilistic "have I possibly seen this key before" check that can say "definitely not present" and reject the request before it touches cache *or* DB.

```
Bit array + k independent hash functions

Insert "user:123":  h1→bit3, h2→bit7, h3→bit1   → set bits 1,3,7 = 1

Check "user:999":    h1→bit2, h2→bit7, h3→bit9
                     bit 2 = 0  →  DEFINITELY NOT in DB → skip cache + DB entirely
```

False positive: possible (bits collide with other keys) — rare, tunable via array size/k. False negative: impossible — bits are only ever set, never cleared.

Analogy: Bloom filter is a bouncer checking a rough guest list photocopy — if your name isn't even on the blurry copy, you're definitely not on the real list, no need to go check the master list at all. But smudges (collisions) can rarely let a stranger's name look plausible.

### Avalanche

**What happens:** a large number of keys expire at (roughly) the same instant — because they were all written with the same TTL in one batch job, or because the cache itself restarted and came up completely empty, or a deploy flushed it. Every one of those keys then misses simultaneously, and every one of those misses becomes a DB query at the same moment.

**Why it's dangerous specifically:** this isn't one hot key — it's the *entire cache* going cold at once. The DB doesn't just take a spike in load for one query pattern, it takes the load the cache was absorbing for everything, all at once. If the DB can't handle that (it usually can't — that's *why* the cache existed), it slows down or falls over, which slows down cache refills, which keeps the DB overloaded — a feedback loop.

**Fixes:**
- TTL jitter: `ttl + random(0, 60)` so writes made in the same batch don't expire in the same second.
- Circuit breaker: if the DB starts erroring/timing out under the flood, stop hammering it and fail fast (serve stale, serve a default, or reject) instead of piling on more requests.
- Pre-warm the cache before switching traffic to it (after a restart or deploy), so it's never serving from empty under real load.
- Persistence (snapshotting cache state to disk periodically, or logging writes) means a restart isn't a *cold* start — it comes back already populated, shrinking the blast radius of "the cache just emptied."

Analogy: avalanche is a stadium where every seat's ticket expires at exactly kickoff time — everyone re-queues at the gate simultaneously instead of trickling in over the day.

### Stampede

**What happens:** narrower than avalanche — *one* specific hot key expires, but that key has many concurrent readers. All of them miss at the same instant and all independently go fetch from the DB and rewrite the cache — for the exact same value.

**Why it's dangerous specifically:** it's wasted, duplicated work. N requests needed the DB to be hit once; instead it gets hit N times, for identical data, in the same instant. The bottleneck isn't "the DB can't handle the query" (it's one query) — it's "the DB got asked the same question N times in the same millisecond" because nothing coordinated the misses.

**Fixes:**
- Mutex on first miss: first request to miss acquires a lock and goes to the DB; every other concurrent request for that key waits and then reads the now-fresh cache value instead of also querying the DB.
- Probabilistic early refresh: refresh the value slightly *before* it actually expires (with some randomness), so it rarely hits true simultaneous expiry.
- Stale-while-revalidate: serve the slightly-stale value immediately while one request refreshes it in the background — nobody waits, nobody duplicates the DB call.
- Request coalescing: same idea as the mutex, generalized — all concurrent callers for the same key share one in-flight fetch instead of issuing N identical ones.

```
Without coordination:  N requests ──▶ N simultaneous DB queries (same query, N times)

With mutex:            Request 1   ──lock──▶ DB ──▶ write cache ──▶ release
                        Request 2..N ──wait───────────────────────▶ read cache
```

Analogy: stampede is a hundred people in the same building independently deciding to call the same restaurant to ask "are you open?" the second the sign flips — instead of one person calling and telling everyone else the answer.

### Telling them apart quickly

| | Scope | Self-healing after? | Root cause |
|---|---|---|---|
| Penetration | One key, forever | No — key doesn't exist anywhere | Bad/missing data, attack, bug |
| Avalanche | Many/all keys, once | Yes, after refill | Simultaneous TTL expiry or empty restart |
| Stampede | One key, briefly | Yes, after first refresh | Concurrent readers on one expiring key |

---

## Multi-Region / Geo-Distributed Caching

| Pattern | How | Trade-off |
|---|---|---|
| Active-Passive | One region's cache is primary, others are read replicas | Simple; failover has lag |
| Active-Active | Multiple regions accept writes independently | Needs conflict resolution (last-write-wins, CRDTs) |
| Independent regional caches + broadcast invalidation | Each region caches locally; writes broadcast an invalidation event | Propagation delay = cross-region staleness window |

**Invalidation broadcast, visualized:**

```
Write to primary DB
     │
     ▼
Publish "key changed" event
     │
     ├──▶ Region A cache: delete key
     ├──▶ Region B cache: delete key
     └──▶ Region C cache: delete key
```

This is the "event-driven invalidation" idea from [Cache Strategies](./Strategies.md), just stretched across geography — same idea, bigger blast radius if a region misses the event.

---

One-line: distributed caching turns a private memory into a public contract — every failure that was invisible locally becomes a production incident at scale, and every consistency guarantee you add back is horizontal scalability you give up.
