# Time-Series Databases — Revision Notes

## What it is

A database optimized around one assumption: **every write has a timestamp, writes are append-only, updates to old data are rare, queries are mostly time-range scans.** That assumption drives storage layout, indexing, and compression — everything else in this doc is a consequence of it.

Analogy: a logbook (TSDB) vs a filing cabinet (relational DB). Filing cabinet = jump to any folder by name (B-tree). Logbook = written strictly in time order; you don't need an index to find "3pm," time itself *is* the index.

## Why it was built (short version)

Relational DBs hit three walls as time-stamped data grew large: B-tree indexes are built for point lookups, not range scans; ACID/transactional overhead taxes every insert even though time-series data is never edited; row-oriented storage has no way to exploit "this value barely changes between rows," so compression was poor.

Timeline: ops/monitoring teams hit this first (late 1990s–2000s). 2010s: Prometheus (SoundCloud, narrow scope — metrics/monitoring only, floats only) and InfluxDB (broad scope — general time-series) emerged as the two flagship answers, followed by a second wave (TimescaleDB, VictoriaMetrics, ClickHouse-for-TSDB) reacting to gaps in the first.

Two philosophies still active today: **(1)** time-series needs its own engine → Prometheus/InfluxDB/VictoriaMetrics. **(2)** it's a workload pattern, extend what you have → TimescaleDB (Postgres extension), ClickHouse (general OLAP, incidentally great at this).

## Use cases

| Domain | What's tracked |
|---|---|
| Observability/DevOps | CPU%, latency, error rates from servers/k8s pods — the original/largest use case |
| IoT/industrial | Sensor streams (vibration, temp, pressure) → predictive maintenance |
| Financial | Tick data, trades — needs µs/ns precision (kdb+ territory) |
| Energy/smart grids | Meter readings, supply/demand balancing |
| Product analytics | Timestamped user events (click, view) — softer fit, often ClickHouse/Druid instead |
| Healthcare | Patient vitals, device telemetry |

**Caveat:** "has a timestamp" ≠ "needs a TSDB." Netflix built a high-throughput immutable event store and explicitly avoided a general-purpose TSDB shape for it — their access pattern was event lookup, not time-range aggregation. Match the tool to the *query pattern*.

---

## Key terms

| Term | Meaning |
|---|---|
| **Cardinality** | count of distinct values |
| **Series cardinality** | count of unique tag-combination series = product of each tag's cardinality. `host(10)×region(3)`=30 (fine). `×request_id(1M)`=30M (death) |
| **Series** | one unique timeline = one tag/metadata combination tracked over time |
| **Tag / Label** | metadata defining series identity (Influx: tags, Prometheus: labels) — keep cardinality LOW |
| **Field / Value** | the actual measurement — high cardinality here is fine, expected |
| **Chunk** | contiguous time-bucketed block of data, compressed/expired as a unit |
| **Hot / cold chunk** | currently-writing (less compressed, in memory) vs sealed (compressed hard) |
| **Retention policy** | auto-delete data older than X — cheap, drops whole chunks |
| **Downsampling** | replace raw data with lower-res summaries as it ages |
| **WAL** | every write appended here first (durability) before being organized into compressed storage |
| **Continuous aggregate** | pre-computed rollup so dashboards don't scan raw rows live |
| **Hypertable** | TimescaleDB's user-facing abstraction — looks like one table, physically many chunks |
| **Constraint exclusion** | query planner skips chunks whose time range doesn't overlap the query — the core time-bound speed trick |
| **Memtable / SSTable** | in-memory sorted buffer / immutable sorted file — LSM tree building blocks |
| **Inverted index** | maps tag value → list of series (cardinality/tag lookup structure, separate from the LSM tree) |

**Data model — three fields, one common confusion:**

| Field | Cardinality wanted | Why |
|---|---|---|
| Timestamp | N/A (always increasing) | drives chunking |
| Value | high is fine | compression works on *pattern of change*, not distinct-value count |
| Metadata/Tag | must stay LOW | defines index size — **this is the one that kills production, not the value field** |

---

## Compression / encoding — 3 layers, stacked

### Layer 1 — value-aware encodings

| Encoding | Does | Used on | Example |
|---|---|---|---|
| Delta | store diff from previous value | timestamps, slow ints | `1000,1010,1020`→`1000,+10,+10` |
| Delta-of-delta | store diff *between* deltas | regular-interval timestamps | deltas `10,10,10`→`10,0,0` |
| XOR (Gorilla algo) | XOR consecutive float bits, store only differing range | slow-changing floats | 22.4 XOR 22.5 → mostly zero bits |
| Varint | use only as many bytes as the number needs (7 data bits + 1 continuation bit/byte) | small ints, esp. post-delta | `+10` → 1 byte not 8 |
| Bit-packing | exact bit-width for a block's known range | small ints in narrow range | more precise, more complex than varint |
| Dictionary | map distinct strings → small ints | low-cardinality strings/tags | `"OK"→0,"ERROR"→1` |
| RLE | store value once + repeat count | long identical/zero runs (post delta-of-delta) | `0,0,0,0,0`→`(0,count=5)` |
| Frame-of-reference | one base value/block + small offsets | ints clustered in narrow range | base=100 → `+2,+5,+1` |

Varint almost always pairs with delta (delta shrinks the number, varint shrinks the byte-count needed to store it).

### Layer 2 — general-purpose compression (no domain knowledge, just byte patterns)

Snappy/LZ4 = fast, lower ratio. Gzip = better ratio, slower. Zstd = best balance. Works well *because* Layer 1 already produced long zero-runs/repeated patterns.

### Layer 3 — columnar storage (layout, not compression itself)

Store all timestamps together, all values together, all tags together — column-wise not row-wise. This is what *makes Layer 1 possible*: you can only delta-encode a sequence if it's physically adjacent. (ClickHouse, Parquet/InfluxDB v3, Timescale compressed hypertables.)

### Where compression breaks down

Strings → no meaningful delta, use dictionary encoding instead (or drift toward log tools like Loki/Elasticsearch). Erratic/noisy values → delta/XOR need consecutive values to be *similar*; wild jumps (22.4→87.1→3.0) share few bits → poor compression. This is the literal math reason TSDBs favor smooth metrics over random data.

---

## Decompression / decoding — when it actually happens

```
WRITE PATH:  point → WAL (raw) → hot chunk (buffered) → chunk seals
                   → encode (delta/XOR/varint) + compress (Zstd/LZ4) → disk

READ PATH:   query (time range + tags) → find overlapping chunks (cheap, via metadata/constraint exclusion)
                   → load only those chunks, SKIP the rest entirely
                   → DECOMPRESS → DECODE (reverse delta/XOR) → filter/aggregate → return
```
---

## Worked example — size reduction, 5 points

```
timestamps: 1000000000, 1000000010, 1000000020, 1000000030, 1000000040
values:     22.4,       22.5,       22.5,       22.6,       22.4
```

| Stage | Timestamps | Values |
|---|---|---|
| Raw (8 bytes fixed each) | 40 bytes | 40 bytes |
| + delta/delta-of-delta | ~9 bytes | — |
| + XOR | — | ~12–15 bytes |
| **Combined** | **~24 bytes total** (vs 80 raw, ~3x here) |

Real workloads (1000s of points/series) hit 10–30x — longer zero-runs, more repetition. Matches industry numbers: ClickHouse ~15–30x, TimescaleDB ~10–15x.

---

## Underlying data structures (engine internals)

Almost every TSDB is built on the **LSM tree (Log-Structured Merge Tree)** — same idea, just specific implementations differ (InfluxDB's TSM, Prometheus's engine).

**Why LSM fits:** writes are "vertical" (constant, across all series, right now), reads are "horizontal" (a time range, later). LSM defers expensive sorting work and batches it — higher write throughput, lower random-write latency than B-trees.

| Component | Role |
|---|---|
| WAL | every write appended here first (durability) |
| Memtable | in-memory sorted buffer (skip list/red-black tree) for recent writes |
| SSTable | immutable sorted file — a filled memtable flushes to one |
| Compaction | background merge of SSTables, drops stale/overwritten data |

Memtable→flush→SSTable is the same shape as hot-chunk→seal→cold-chunk, one layer lower.

**Per-DB adaptation:** InfluxDB (TSM) — started on LevelDB (LSM), hit file-handle/write-perf issues, built custom WAL + TSM files, sharded by time block. Prometheus — Head block = hot chunk (≤120 points/2hr span) + WAL; v2+ moved to block structure (series+index per ~2hr) via mmap.

**Inverted index (separate structure, handles tags not values):** maps label value → list of series ("Postings"), same idea as a search engine's word→document index. Prometheus assumes ~4.4M series × ~12 labels → <5,000 *unique label values*, fits in memory — that assumption is the cardinality ceiling baked into the design. Exceed it, this index is what blows up.

**Division of labor:** LSM tree + inverted index = how data is organized/found. Delta/XOR/varint = how bytes inside a file are made small. Two separate, complementary layers.

---

## Storage patterns — right vs wrong

| Area | ✅ Right | ❌ Wrong | Why |
|---|---|---|---|
| Tag design | bounded sets (`host`, `region`) | `request_id`/UUID/raw IP as a tag | unbounded tag → index explosion |
| High-cardinality IDs | store as a field, or relational join | tag it "to filter later" | fields aren't indexed like tags |
| Schema consistency | same field name/type for a series' life | type drift (string↔float) | breaks delta/XOR assumptions |
| Write granularity | match real sensor resolution | over-sample "just in case" | inflates storage, no new info |
| Out-of-order writes | buffer/order before write | assume all TSDBs tolerate late data | forces re-compression of sealed chunks |
| Retention | design tiers upfront | store everything forever | chunk-drop is cheap; retroactive downsampling at scale isn't |
| Multi-tenant tags | tag only if tenant count bounded/known | tag blindly | `tenant×sensor×metric` cardinality multiplies fast |

## Query patterns — right vs wrong

| Area | ✅ Right | ❌ Wrong | Why |
|---|---|---|---|
| Time bound | always bound with explicit range | rely on `LIMIT` with no time bound | LIMIT truncates output, not the scan |
| Long-range dashboards | query pre-aggregated/downsampled data | scan raw data over months | live-aggregating billions of rows/query |
| Filtering | filter on tags first | filter primarily on field/value | tag filters use the index; field filters scan decompressed values |
| Group by | group by bounded tags | `GROUP BY request_id`-like tag | cardinality explosion at query time |
| Point lookups | use relational DB/join for arbitrary fields | expect fast `findOne`-by-ID | no general secondary index outside time+tag |
| Joins | TimescaleDB (real SQL) or app-layer join | expect native joins in Prometheus/Influx | pure TSDBs have no join semantics |
| Updating history | write a new corrected point | treat as routine CRUD update | sealed chunks need decompress→modify→recompress |
| Last-value queries | use native `last()` function | loop per-series `ORDER BY time DESC LIMIT 1` | native version uses chunk metadata directly; looping = N+1 |

**One-liner:** tags + time are free to filter/index on. Everything else costs you — every "wrong" row is someone using a TSDB outside that free lane.

---

## Limitations

Point lookups by arbitrary non-time/non-tag fields. Updating historical rows (fights the append-only design). Joins across series (except Timescale). High-cardinality tags — not just slower, can take the cluster down.

---
