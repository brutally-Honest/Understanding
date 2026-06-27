# Database Indexes

---

## 1. Storage Internals — how data lives on disk

A database file is split into fixed-size **pages** (4–16 KB, the atomic unit of I/O — even 1 byte read pulls the whole page). RAM ~100ns, SSD ~100µs, HDD ~10ms, so every query's real cost is **pages fetched**, not rows examined.

```
┌──────────────────────────────┐
│ Page Header (id, free ptr)   │
├──────────────────────────────┤
│ Slot Array  [→r1][→r2][→r3]  │  grows forward →
├──────────────────────────────┤
│         Free Space           │
├──────────────────────────────┤
│   ...row3 | row2 | row1      │  grows backward ←
└──────────────────────────────┘
```

- **TID (Tuple ID) / RID** = `(page_id, slot_number)` — a row's physical address. This is exactly what an index stores.
- **Heap file** = the collection of all data pages for a table, stored in insertion order — no inherent sort.
- Index lookup = 2 steps: traverse index → get TID → fetch exact heap page + slot.

---

## 2. Evolution — how we got here

| Stage | Idea | What it actually gave us |
|---|---|---|
| Heap scan | Read every row, no structure | Baseline — works, but O(N) |
| Sorted file + binary search | Sort rows physically, binary search them | First O(log N) reads |
| ISAM (1960s) | Separate sparse index from data pages, binary search the index instead of the data | **The index is structurally separate from the data** — still true today |
| B-Tree (1972) | Self-balancing tree, auto rebalance via splits/merges on every write | Guaranteed O(log N) forever, regardless of insert order |
| B+ Tree (1970s–80s) | All data only at leaves, leaves linked in sorted order | Fast range scans + higher fanout → universal default |
| LSM Tree (1996 paper, popularized 2000s–10s) | Buffer writes in memory, flush as sorted immutable files, merge in background | Optimized the write side instead of the read side — for workloads where B+ Tree writes were the bottleneck |

**The throughline:** storage engines kept asking the same question — *minimize pages touched per operation* — and each stage answered it for a different access pattern (reads, then writes, then both).

### What killed each prior approach

| Approach | Breaking point | Concrete symptom |
|---|---|---|
| Heap scan | Table grows past what fits in a few hundred pages | Latency scales linearly with table size — 1M rows = seconds, not ms |
| Sorted file + binary search | Any insert/delete | Inserting one row mid-file shifts every row after it — O(N) write |
| ISAM | Page fills up after repeated inserts | Overflow chains form; 2-page lookup degrades to 10+ pages, needs full reorg |
| B-Tree | Range queries (`BETWEEN`, `ORDER BY`) | Data scattered across nodes — no way to walk a range without re-traversing |
| B+ Tree | Extremely write-heavy workloads (logging, time-series) | Every write still costs a random-ish page write + possible split |
| Single B+ Tree index (general) | Dataset exceeds one machine's disk/memory | Index design stops helping — problem moves to sharding/partitioning |

---

## 3. Reading — without an index vs with one

| | **Without index (seq scan)** | **With index (B+ Tree)** |
|---|---|---|
| **Equality** (`id = X`) | Read every page, check every row. O(N pages). | Traverse tree (3–4 pages) → get TID → fetch 1 heap page. O(log N). |
| **Range** (`BETWEEN`) | Same full scan, discard non-matching rows. | Find start leaf, follow linked-list of leaves until range ends. No re-traversal needed. |
| **Sort + Limit** (`ORDER BY ... LIMIT 20`) | Load **all** rows into memory, sort, return top 20. | Leaves are already sorted — read 20 entries, stop. Most dramatic real-world win. |
| **Compound filter** (`a = X AND b = Y`) | Full scan, check both conditions per row. | Equality narrows subtree, remaining condition scanned within it — if index column order matches query. |
| **Write cost** | None — only heap write | Heap write + index write per index; possible page split |

**Index-only / covering scan:** if every column the query needs is inside the index, the heap is never touched.

---

## 4. The Trade-offs Every Index Negotiates

| Constraint | One side of the trade | Other side of the trade |
|---|---|---|
| **Read speed vs. write speed** | More indexes → faster, more flexible queries | More indexes → slower inserts/updates, more page splits |
| **Storage space vs. read speed** | Wider (covering) indexes → fewer heap hops, index-only scans | Wider indexes → larger index size, more I/O to scan the index itself |
| **Ordering vs. write locality** | Sequential PK → no page splits, predictable writes | Random PK (UUID) → splits constantly, but "fans out" writes to avoid hot-page contention at extreme scale |
| **Consistency vs. latency** (distributed) | Strong consistency → reads always correct, slower writes (wait for replicas) | Eventual consistency → fast writes, reads can briefly miss recent index entries |
| **Specificity vs. general-purpose** | Specialized index (GIN, Bitmap, Hash) → excellent for one pattern | B+ Tree → mediocre-to-good at everything, no blind spots |

**The constraint underneath all of them:** disk I/O is slow and finite. Every trade-off above is really asking — *which operation gets to touch fewer pages, the read or the write?*

---

## 5. Index Data Structures

| Structure | How it works | Supports | Doesn't support | Best for |
|---|---|---|---|---|
| **B+ Tree** | Balanced tree, data only at leaves, leaves linked in sorted order | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, prefix `LIKE` | Full-text, spatial, suffix `LIKE` | Universal default — equality, range, sort |
| **Hash** | `hash(key) → bucket → TID` | `=` only | Any range/sort op — hash destroys order | Session tokens, UUIDs, exact-match-only columns |
| **LSM Tree** | Writes go to in-memory MemTable → flushed as sorted SSTables on disk → background compaction merges them | Fast sequential writes | Reads may check multiple levels (read amplification) | Write-heavy workloads |
| **GIN** (inverted index) | `token → [list of TIDs]` containing that token | Full-text search, JSONB `@>`, array containment | Plain equality/range on scalar columns | Search, JSON, array columns |
| **Bitmap** | One bit per row per distinct value; bitwise AND/OR to combine filters | Fast multi-column AND/OR on low-cardinality columns | Frequent updates (full bitmap rewrite) | Analytics/OLAP, not OLTP |

---

## 6. Index Classification (by role)

| Type | What it means | Pros | Cons |
|---|---|---|---|
| **Primary** | Auto-created on PK, one per table, unique | Free, guarantees uniqueness, fast PK lookup | Only one per table |
| **Secondary** | Manually created on any non-PK column | Enables fast lookup on any column, multiple allowed | Each one adds write overhead |
| **Clustered** | Table rows physically sorted by this key; leaf = actual row data | Range scans on the key are sequential reads — very fast | Only one per table; random key inserts (UUID) fragment the whole table |
| **Non-clustered** | Index separate from heap; leaf = key + TID pointer | Writes don't reorder heap; multiple indexes coexist cleanly | Extra hop: index → heap fetch |
| **Compound** | Index on multiple columns, e.g. `(a, b, c)` | One index serves many query shapes; can become covering | Column order critical — leftmost prefix rule governs usability |
| **Unique** | B+ Tree + uniqueness constraint check on write | Enforces data integrity at storage level | Slightly slower writes; multiple NULLs usually still allowed |
| **Covering** | Compound index containing every column a query needs | Index-only scan — heap never touched, max read speed | Larger index, more write overhead per extra column |
| **Partial** | B+ Tree indexing only rows matching a `WHERE` predicate | Tiny index, zero overhead for non-matching rows | Only usable when query predicate matches the index predicate exactly |

**Leftmost prefix rule** for compound index `(a, b, c)`:

| Query | Usable? |
|---|---|
| `WHERE a = 1` | Yes |
| `WHERE a = 1 AND b = 2` | Yes |
| `WHERE a = 1 AND b = 2 AND c = 3` | Yes, fully |
| `WHERE a = 1 AND c = 3` | Partial — `a` used, `c` filtered after fetch (gap at `b`) |
| `WHERE a > 1 AND b = 2` | Partial — range on `a` stops prefix match for `b` |
| `WHERE b = 2` | No — skips leading column `a` |

**Design rule:** equality columns first, range column last, sort column last.

---

## 7. Why IDs Matter — writes, page splits, ordering

| ID type | Insert pattern | Page splits | Use case |
|---|---|---|---|
| Sequential (`SERIAL`/`BIGSERIAL`) | Always appended at the end | Rare | Internal PK, fast writes |
| UUID v4 (random) | Random position every time | Frequent | Avoid as clustered/primary key |
| UUID v7 / ULID | Time-prefixed → roughly sequential | Rare | Distributed systems needing global uniqueness |
| MongoDB `ObjectId` | Time-prefixed (timestamp + random + counter) | Rare by default | Mongo's default `_id` — already split-resistant |

```
Page split mechanics (random key inserts):
1. New random key arrives, target leaf is full
2. DB allocates new page, splits leaf in half
3. Parent gets new routing key
4. If parent also full → split propagates up, may increase tree height
5. Result: extra write I/O, fragmented ranges, ~50% page fill after split
```

**Secondary index → row lookup, by database:**

| Database | Secondary index leaf stores | Extra hop to fetch row? |
|---|---|---|
| PostgreSQL | TID (physical page + slot) | No — direct heap access |
| MySQL InnoDB | Primary key value | Yes — re-traverse clustered index |
| MongoDB | `_id` value | Yes — fetch document by `_id` |

---

## 8. Where This Shows Up in Practice

| Scenario | What's at stake | What you reach for |
|---|---|---|
| User login by email | Point lookup must be sub-millisecond at any table size | B+ Tree (or Hash) secondary index on `email`, unique constraint |
| Feed sorted by recency, paginated | `ORDER BY created_at DESC LIMIT 20` must not scan the whole table | Index on `created_at DESC`; cursor/keyset pagination, not OFFSET |
| Multi-tenant SaaS query (`WHERE tenant_id = X AND status = 'active'`) | Every query scoped to a tenant — most common access pattern | Compound index `(tenant_id, status, ...)` — tenant_id leading |
| Full-text search across descriptions | Substring/word search, not exact match | GIN index + `tsvector` — B+ Tree can't do this |
| IoT sensor data, append-only, time-series | Extreme write throughput, range queries by time window | LSM-tree-backed store, or B+ Tree with time-prefixed key |
| Geo "find nearby" queries | 2D spatial proximity, not a single sortable key | GiST / R-Tree spatial index |
| Audit log / event sourcing table | Write-once, rarely queried by anything but time and ID | Minimal indexing — extra indexes are pure write tax here |
| Analytics dashboard, millions of rows by category | Few distinct values, AND/OR combinations | Bitmap index (OLAP) — not appropriate for OLTP |
| Distributed system, data too large for one node | Cross-node query and write distribution | Shard key chosen for colocated access pattern, not just uniqueness |

---

## 9. Where Queries Actually Go Wrong

| Category | Issue | Fix |
|---|---|---|
| **Schema** | Over-normalized (6 joins per query) or wrong data types (dates/numbers as TEXT) | Denormalize hot paths; use correct native types |
| **Page splits** | Random PK (UUID v4) on high-write table; updates that grow row size | Sequential/time-ordered PK (BIGSERIAL, UUID v7, ULID); leave fill factor headroom |
| **Wrong indexes** | Leftmost prefix violated; function wrapped around column (`LOWER(email)`); too many indexes on a write-heavy table | Reorder to equality→range→sort; functional index; audit and drop unused indexes |
| **Ordering** | `ORDER BY` with no matching index; `OFFSET N` pagination on large tables | Index matching sort direction; keyset pagination (`WHERE id > last_seen_id`) |
| **Distributed** | Cross-shard joins; hot shard from uneven key distribution; replication lag | Colocate via shard key design; high-cardinality shard key; read from primary when freshness matters |
| **Common** | Stale planner statistics; N+1 queries from application code | Run `ANALYZE` after bulk loads; batch fetch with `IN (...)` instead of looping |

**The pattern underneath all of it:** every fix reduces to the same move — fewer pages touched, fewer round trips. A "hot shard" in a distributed system and a "hot page" in a single B+ Tree are the same root problem (uneven key distribution) at different scales. Schema design is upstream of indexing — no index fully compensates for a bad data model.

---

## 10. Real Systems, Real Choices

| System | What it uses | Why this choice fits its workload |
|---|---|---|
| **PostgreSQL** | B+ Tree (default), GIN, GiST, BRIN, Hash | General-purpose OLTP — B+ Tree default, specialized structures opt-in |
| **MySQL (InnoDB)** | Clustered B+ Tree on PK; secondary indexes point to PK, not TID | Table rows live inside the PK's B+ Tree itself — why UUID PKs hurt more here |
| **MongoDB (WiredTiger)** | B-Tree (default) or LSM-tree mode; `ObjectId` time-prefixed by default | Default `_id` avoids the random-insert page-split problem out of the box |
| **Cassandra / ScyllaDB** | LSM Tree (SSTables + compaction) | Massive write throughput across distributed nodes |
| **RocksDB / LevelDB** | LSM Tree | Embedded engine — same write-optimized reasoning, used inside other DBs |
| **Elasticsearch** | Inverted index (GIN's cousin) | Built specifically for full-text search |
| **Redis** | In-memory hash table | No disk page concept — when everything fits in RAM, I/O-minimization disappears as a problem |

---

## 11. The Mental Model to Keep

> **An index is a phonebook's thumb-tab, not the phonebook itself.**
> The data is still all 1,000 pages, unsorted, full of everything. The index doesn't shrink the data — it's a smaller, sorted side-structure that tells you exactly which page to flip to. You pay for that shortcut every time someone adds a new entry, because the thumb-tabs have to stay in order too.

**The one-liner that travels into any interview or design doc:** *Every read you speed up with an index, you pay for again on every write — the only question is whether your workload reads more than it writes.*

---

## One-line heuristics (practical recap)

- Index any column used in `WHERE`, `JOIN`, or `ORDER BY` + `LIMIT` — if selective and table is large.
- Skip indexing low-cardinality columns or small tables (<10K rows).
- Compound index column order: **equality → range → sort**.
- Covering index for hottest read queries — eliminates the heap hop entirely.
- Never use random UUID as a clustered/primary key on high-write tables — use ULID/UUID v7 or a sequential internal PK.
- Function calls on indexed columns silently disable the index — needs a functional index.
- Unused indexes still cost writes — check usage stats and drop dead ones.
