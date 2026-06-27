# Database Indexes

> Prerequisites: [Storage Fundamentals](Storage%20Fundamentals.md) — pages, heap files, TIDs, I/O cost model.

---

## 1. What an Index Is

An index is a separate, sorted side-structure that maps a column's values to TIDs (page + slot address of the matching row). It doesn't change the heap — the data stays unsorted. It just tells the DB which page to go to instead of reading all of them.

```
Without index:  scan every page → check every row → O(N pages)
With index:     traverse index (3–4 pages) → TID → fetch 1 heap page → O(log N)
```

The cost: every index you add means every insert/update has to write to the heap *and* update the index. Read speed traded for write overhead.

---

## 2. Evolution — how we got here

| Stage | Idea | What it gave us |
|---|---|---|
| Heap scan | Read every row, no structure | Baseline — O(N) |
| Sorted file + binary search | Sort rows physically | First O(log N) reads |
| ISAM (1960s) | Separate sparse index from data pages | **Index structurally separate from data** — still true today |
| B-Tree (1972) | Self-balancing tree, rebalances on writes | Guaranteed O(log N) regardless of insert order |
| B+ Tree (1970s–80s) | Data only at leaves, leaves linked in sorted order | Fast range scans + higher fanout → universal default |
| LSM Tree (1996) | Buffer writes in memory, flush as sorted immutable files | Optimised the write side — for workloads where B+ Tree writes were the bottleneck |

**The throughline:** every stage asked the same question — *minimise pages touched per operation* — and answered it for a different access pattern.

### What killed each prior approach

| Approach | Breaking point | Symptom |
|---|---|---|
| Heap scan | Table grows past a few hundred pages | Latency scales linearly — 1M rows = seconds |
| Sorted file | Any insert/delete | Inserting one row mid-file shifts every row after it — O(N) write |
| ISAM | Page fills up after repeated inserts | Overflow chains form; 2-page lookup degrades to 10+ pages |
| B-Tree | Range queries (`BETWEEN`, `ORDER BY`) | Data scattered across nodes — can't walk a range without re-traversal |
| B+ Tree | Extremely write-heavy workloads | Every write costs a random-ish page write + possible split |

---

## 3. Reading — without an index vs with one

| | **Without index (seq scan)** | **With index (B+ Tree)** |
|---|---|---|
| **Equality** (`id = X`) | Read every page, check every row. O(N). | Traverse tree → TID → 1 heap page. O(log N). |
| **Range** (`BETWEEN`) | Full scan, discard non-matching rows. | Find start leaf, walk linked leaf chain. No re-traversal. |
| **Sort + Limit** (`ORDER BY ... LIMIT 20`) | Load all rows, sort, return top 20. | Leaves already sorted — read 20, stop. Biggest real-world win. |
| **Compound filter** (`a = X AND b = Y`) | Full scan, check both per row. | Equality narrows subtree, remaining condition filtered within it. |
| **Write cost** | Heap write only | Heap write + index write per index; possible page split |

**Covering scan:** if every column the query needs is inside the index, the heap is never touched — no TID fetch required.

---

## 4. Tradeoffs Every Index Negotiates

| Constraint | One side | Other side |
|---|---|---|
| **Read vs write speed** | More indexes → faster queries | More indexes → slower writes, more splits |
| **Storage vs read speed** | Wider (covering) indexes → index-only scans | Wider indexes → larger index, more I/O to scan it |
| **Ordering vs write locality** | Sequential PK → no splits, predictable writes | Random PK (UUID) → constant splits |
| **Specificity vs general-purpose** | Specialised index (GIN, Hash) → excellent for one pattern | B+ Tree → good at everything, no blind spots |

Underneath all of them: *which operation gets to touch fewer pages, the read or the write?*

---

## 5. Index Data Structures

> Full internals — structure, amplification mechanics, natural fit vs wrong tool per type: [Index Internals](Index%20Internals.md)

| Structure | Built for | Wins when | Loses when |
|---|---|---|---|
| **B+ Tree** | Mixed OLTP, range + equality | Reads and writes are balanced | Write-heavy streams, multi-valued columns |
| **Hash** | Pure equality | You only ask `WHERE key = X` | Any range, prefix, or sort needed |
| **LSM Tree** | Write-dominant, append-heavy | Writes vastly outnumber reads | Random reads on cold data |
| **GIN** | Multi-valued data (arrays, JSONB, full-text) | One row has many indexable things | High write rate, single scalar equality |
| **Bitmap** | Low-cardinality multi-filter analytics | Read-heavy `WHERE a=x AND b=y AND c=z` | High-write OLTP, high-cardinality columns |

---

## 6. Index Classification (by role)

| Type | What it means | Pros | Cons |
|---|---|---|---|
| **Primary** | Auto-created on PK, one per table, unique | Free, guarantees uniqueness | Only one per table |
| **Secondary** | Manually created on any non-PK column | Fast lookup on any column | Each adds write overhead |
| **Clustered** | Table rows physically sorted by this key | Range scans on key = sequential reads | Only one per table; random keys (UUID) fragment the whole table |
| **Non-clustered** | Index separate from heap | Multiple indexes coexist cleanly | Extra hop: index → heap fetch |
| **Compound** | Index on multiple columns e.g. `(a, b, c)` | One index serves many query shapes | Column order critical — leftmost prefix rule |
| **Covering** | Compound index containing every column the query needs | Index-only scan — heap never touched | Larger index, more write overhead |
| **Partial** | Indexes only rows matching a `WHERE` predicate | Tiny index, zero overhead for non-matching rows | Only usable when query predicate matches exactly |

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
| MongoDB `ObjectId` | Time-prefixed (timestamp + random + counter) | Rare by default | Mongo's default `_id` — split-resistant out of the box |

```
Page split mechanics (random key inserts):
1. New random key arrives, target leaf is full
2. DB allocates new page, splits leaf in half
3. Parent gets new routing key
4. If parent also full → split propagates up, may increase tree height
5. Result: extra write I/O, fragmented ranges, ~50% page fill after split
```

**Secondary index → row lookup, by database:**

| Database | Secondary index leaf stores | Extra hop? |
|---|---|---|
| PostgreSQL | TID (physical page + slot) | No — direct heap access |
| MySQL InnoDB | Primary key value | Yes — re-traverse clustered index |
| MongoDB | `_id` value | Yes — fetch document by `_id` |

---

## 8. Where This Shows Up in Practice

| Scenario | What's at stake | What you reach for |
|---|---|---|
| User login by email | Point lookup sub-millisecond at any table size | B+ Tree secondary index on `email`, unique constraint |
| Feed sorted by recency, paginated | `ORDER BY created_at DESC LIMIT 20` must not scan the table | Index on `created_at DESC`; keyset pagination, not OFFSET |
| Multi-tenant SaaS (`WHERE tenant_id = X AND status = 'active'`) | Every query scoped to a tenant | Compound index `(tenant_id, status)` — tenant_id leading |
| Full-text search across descriptions | Substring/word search, not exact match | GIN index + `tsvector` — B+ Tree can't do this |
| IoT sensor data, append-only | Extreme write throughput, range queries by time | LSM-backed store, or B+ Tree with time-prefixed key |
| Audit log / event sourcing | Write-once, rarely queried by anything but time | Minimal indexing — extra indexes are pure write tax |
| Analytics dashboard, millions of rows | Few distinct values, AND/OR combinations | Bitmap index (OLAP) — not appropriate for OLTP |

---

## 9. Where Queries Actually Go Wrong

| Category | Issue | Fix |
|---|---|---|
| **Schema** | Wrong data types (dates/numbers as TEXT), over-normalized | Use correct native types; denormalize hot paths |
| **Page splits** | Random PK (UUID v4) on high-write table | Sequential/time-ordered PK (BIGSERIAL, UUID v7, ULID) |
| **Wrong indexes** | Leftmost prefix violated; function on indexed column (`LOWER(email)`) | Reorder to equality→range→sort; use functional index |
| **Ordering** | `ORDER BY` with no matching index; `OFFSET N` on large tables | Index matching sort direction; keyset pagination |
| **Common** | Stale planner statistics; N+1 queries | `ANALYZE` after bulk loads; batch with `IN (...)` |

Every fix reduces to the same move — fewer pages touched, fewer round trips. Schema design is upstream of indexing — no index fully compensates for a bad data model.

---

## 10. Real Systems, Real Choices

| System | What it uses | Why |
|---|---|---|
| **PostgreSQL** | B+ Tree (default), GIN, GiST, BRIN, Hash | General-purpose OLTP — B+ Tree default, specialised opt-in |
| **MySQL (InnoDB)** | Clustered B+ Tree on PK; secondary indexes point to PK | Rows live inside the PK's B+ Tree — why UUID PKs hurt more here |
| **MongoDB (WiredTiger)** | B-Tree; `ObjectId` time-prefixed by default | Default `_id` avoids the random-insert page-split problem |
| **Cassandra / ScyllaDB** | LSM Tree (SSTables + compaction) | Massive write throughput across distributed nodes |
| **RocksDB / LevelDB** | LSM Tree | Embedded engine — same write-optimized reasoning |
| **Elasticsearch** | Inverted index (GIN's cousin) | Built specifically for full-text search |
| **Redis** | In-memory hash table | No disk page concept — I/O-minimization disappears when data fits in RAM |

---

## 11. One-Line Principle

> **An index is a phonebook's thumb-tab, not the phonebook itself.** The data is still all 1,000 pages, unsorted. The index doesn't shrink it — it's a smaller sorted side-structure that tells you which page to flip to. You pay for that shortcut on every write, because the thumb-tabs have to stay in order too.

**The one-liner:** *Every read you speed up with an index, you pay for again on every write — the only question is whether your workload reads more than it writes.*

---

## Quick Heuristics

- Index columns used in `WHERE`, `JOIN`, or `ORDER BY` + `LIMIT` — if selective and table is large.
- Skip indexing low-cardinality columns or small tables (<10K rows).
- Compound index order: **equality → range → sort**.
- Covering index for hottest read queries — eliminates the heap hop entirely.
- Never use random UUID as clustered/primary key on high-write tables — use ULID, UUID v7, or sequential PK.
- Function calls on indexed columns silently disable the index — needs a functional index.
- Unused indexes still cost writes — check usage stats and drop dead ones.
