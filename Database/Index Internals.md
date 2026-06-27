# Index Data Structures: Internals & Use Cases

Detailed breakdown of each index type — internal structure, read/write mechanics, and production fit.

---

## Structural Comparison

| | B+Tree | Hash Index | LSM Tree | GIN | Bitmap |
|---|---|---|---|---|---|
| **Core idea** | Sorted balanced tree, data at leaves | Hash key → bucket, direct jump | Write to memory, flush to disk, merge levels | Term → posting list of row IDs | Distinct value → bit array (1 bit per row) |
| **On-disk layout** | Internal routing nodes + linked leaf pages | Bucket files + overflow pages | MemTable (RAM) + immutable SSTables (disk) | B+tree of terms + compressed posting lists | Bit arrays per distinct value, compressed |
| **Read complexity** | O(log n) | O(1) avg | O(log n × levels) + bloom filter skip | O(log t + p), t=terms p=posting list size | O(n/64) bitwise scan |
| **Write complexity** | O(log n), random I/O | O(1) avg | O(1) amortized (sequential writes) | O(k log n), k=tokens per row | O(n/64) per value bitmap |
| **Space overhead** | Moderate (~70% page fill factor) | Low–moderate (load factor dependent) | Moderate + space amplification during compaction | High (posting lists + main B+tree) | Low for low-cardinality, explodes for high |
| **Range queries** | ✅ Native (leaf chain walk) | ❌ None | ✅ Within SSTable levels | ❌ Wrong tool | ❌ Wrong tool |
| **Equality queries** | ✅ Good | ✅ Best | ✅ Good | ✅ Containment semantics | ✅ Good on low-cardinality columns |
| **Multi-value containment** | ❌ | ❌ | ❌ | ✅ Best | ✅ Multi-filter AND/OR |
| **Write throughput** | Moderate | High | Very high (sequential I/O) | Low | Low–moderate |
| **Read throughput** | High | Very high | Moderate (read amplification) | High for GIN-native queries | Very high (bitwise ops) |
| **Background maintenance** | Page splits on overflow | Rehashing on resize | Compaction (background, tuneable) | Fastupdate pending list + vacuum | Full rebuild on high churn |
| **Write concurrency** | Good (page-level locking) | Good | Good (MemTable is concurrent) | Degraded (pending list bottleneck) | Bad (value-level bitmap lock) |
| **Primary territory** | OLTP, mixed workloads | Caches, session stores, in-memory | Append-heavy, time-series, logs | JSONB, arrays, full-text | OLAP, analytics, low-cardinality filters |

---

## B+Tree

**Structure:** Balanced tree. Internal nodes hold keys for routing only. All actual data lives at leaf level. Leaves are doubly-linked — range scan walks the chain without re-traversing the tree.

**Analogy:** Phone book. Sorted, so you can find everyone named "K something" because order is maintained end-to-end.

### Natural fit — access patterns this index is built for

| Use case | DB | Why B+tree fits |
|---|---|---|
| User lookup by ID or email | MySQL InnoDB, Postgres | Equality + point access, single value per row |
| Order history by date range | MySQL InnoDB | Range query on timestamp is natural to sorted leaves |
| Primary key (clustered) index | Almost every RDBMS | Rows stored in key order; sequential inserts fast |
| Composite indexes (prefix scans) | Postgres, MySQL | Leading column equality + trailing column range |
| Foreign key lookups | MySQL, Postgres | Joins traverse B+tree on referenced column |

### Wrong tool — where this index fights the access pattern

| Use case | Why it struggles |
|---|---|
| Write-heavy IoT stream (10k+ events/sec) | Random I/O per insert; disk seek cost accumulates |
| `WHERE tags @> ARRAY['go', 'redis']` | No model for multi-value per row |
| `WHERE status = 'active'` on 40M rows, 3 statuses | Low-cardinality; index stores 40M entries for 3 values |
| Full-text search with relevance ranking | Tokenization and scoring have no native B+tree model |

---

## Hash Index

**Structure:** Hash function maps key → bucket number → direct I/O to that bucket. No tree traversal. No ordering maintained. Overflow chains or extendible hashing handles collisions and resizing.

**Analogy:** Locker room. You hash the name to get locker 47, go straight there. But "all lockers between 30–50"? The locker room has no concept of order.

### Natural fit — access patterns this index is built for

| Use case | DB | Why Hash fits |
|---|---|---|
| Session store lookup by token | Redis, Memcached | Pure equality, high QPS, memory-resident |
| Primary key equality on hot rows | InnoDB adaptive hash (automatic) | InnoDB detects hot B+tree pages and auto-builds hash on top |
| In-memory join probes | Postgres hash join (query planner) | Hash table built at query time for join operations |
| Feature flag lookup by user ID | Any in-memory store | Equality only, microsecond latency |
| Deduplication check at ingest | Redis SET | O(1) membership test |

### Wrong tool — where this index fights the access pattern

| Use case | Why it struggles |
|---|---|
| `WHERE created_at BETWEEN t1 AND t2` | No order; range is impossible |
| `ORDER BY` on indexed column | No sorted structure to walk |
| `LIKE 'prefix%'` search | Requires sorted structure for prefix scan |
| High-churn key space (constant inserts/deletes) | Overflow chains or rehashing degrades performance |
| Durability-critical data (some implementations) | Some hash indexes are in-memory only, not WAL-backed |

---

## LSM Tree (Log-Structured Merge Tree)

**Structure:** Writes land in MemTable (RAM, sorted skip list or red-black tree). WAL ensures durability. When MemTable fills, it flushes to an immutable SSTable on disk — one sequential write. SSTables accumulate in levels. Background compaction merges, deduplicates, removes tombstones. Bloom filters per SSTable skip files that don't contain a key.

**Analogy:** Sticky notes on a desk. New stuff goes on a note (MemTable). When the desk is full, stack them into a labelled folder (SSTable flush). Periodically merge folders and throw out outdated notes (compaction). Writing is always fast. Reading checks a few folders — organised enough that it doesn't blow up.

### Amplification triangle — tune one, others suffer

| Compaction strategy | Write amp | Read amp | Space amp | Best for |
|---|---|---|---|---|
| Size-tiered | Low | High | High | Write-dominant, read-tolerant |
| Leveled | High | Low | Low | Mixed, read matters |
| FIFO (append log only) | Lowest | Highest | Highest | Pure append, TTL-expiry logs |

### Natural fit — access patterns this index is built for

| Use case | DB / Engine | Why LSM fits |
|---|---|---|
| IoT telemetry ingestion | InfluxDB, ScyllaDB, TimescaleDB (hybrid) | Sequential writes, time-ordered keys reduce compaction cost |
| Event log storage | Cassandra, HBase | Append-only patterns; LSM shines on monotonically increasing keys |
| Kafka internal log compaction | RocksDB | Compaction semantics match Kafka's log compaction exactly |
| Feature store bulk writes | RocksDB embedded | Configurable compaction, embedded, no server overhead |
| Audit / activity streams | Cassandra | High write throughput with time-based partitioning |

### Wrong tool — where this index fights the access pattern

| Use case | Why it struggles |
|---|---|
| Heavy random-read OLTP (user profile fetch by ID) | Multiple SSTables to probe before bloom filters narrow it |
| Frequent updates to the same key | Key exists in multiple SSTables until compaction; read amplification on that key |
| Latency-sensitive reads during compaction bursts | Compaction I/O competes with read I/O on the same disk |
| Small datasets | Compaction overhead not justified; B+tree wins easily |

---

## GIN Index (Generalized Inverted Index)

**Structure:** B+tree of "entries" (tokens, array elements, JSONB keys/values). Each entry points to a compressed, sorted posting list of row IDs. Multi-condition queries intersect posting lists via sorted merge. "Generalized" means the operator class is pluggable — same structure handles `tsvector`, JSONB, arrays, range types.

**Analogy:** Back-of-book index. Look up "binary search" → pages 23, 47, 89. GIN is exactly this — not the data reorganised, but a map from terms to locations where they appear.

### Natural fit — access patterns this index is built for

| Use case | DB | Why GIN fits |
|---|---|---|
| `WHERE doc @> '{"role": "admin"}'` | Postgres JSONB | GIN indexes all JSONB keys and values |
| `WHERE tags && ARRAY['kafka', 'redis']` | Postgres arrays | Array overlap → posting list intersection |
| Full-text: `WHERE search_vector @@ to_tsquery('backend')` | Postgres `tsvector` | Tokenized terms → posting lists, relevance via `ts_rank` |
| Multi-keyword product search at moderate scale | Postgres | GIN on `tsvector`; no Elasticsearch needed below ~10M docs |
| `WHERE metadata ? 'region'` (key existence) | Postgres JSONB | GIN covers key-existence operator natively |

### Wrong tool — where this index fights the access pattern

| Use case | Why it struggles |
|---|---|
| High-write catalog (product updates every second) | Every write updates multiple posting lists; fastupdate defers but doesn't eliminate cost |
| Equality on a single scalar column | B+tree is cheaper; GIN overhead not justified |
| Range queries on array elements | GIN does not maintain order across elements |
| Large, frequently mutated JSONB blobs (live telemetry) | Posting list churn + index bloat + vacuum pressure |

---

## Bitmap Index

**Structure:** One bit array per distinct value of a column. Bit position = row number. Multi-condition filtering = bitwise AND/OR across arrays. Modern implementations use Roaring Bitmaps or EWAH compression. Postgres does not store bitmap indexes on disk — it builds them at query time from B+tree scans (BitmapAnd / BitmapOr nodes in the query plan).

**Analogy:** Transparency sheets. Each filter is a sheet with holes punched where rows match. AND = stack the sheets. Light only passes through where all sheets align. Stacking is O(n/64) — 64 bits processed per CPU instruction.

### Natural fit — access patterns this index is built for

| Use case | DB | Why Bitmap fits |
|---|---|---|
| `WHERE region='south' AND status='active' AND tier='premium'` | Oracle, ClickHouse, Redshift | Multi-column low-cardinality AND = bitwise intersect |
| Dashboards on categorical dimensions | ClickHouse, Redshift, DuckDB | Read-heavy aggregation over low-cardinality columns |
| Postgres multi-condition query execution | Postgres BitmapAnd (runtime) | Planner builds bitmap from B+tree scans, ANDs at runtime |
| Ad targeting: `age_group AND geo AND device_type` | Pilosa, Roaring Bitmap engines | Combinatorial filters over low-cardinality dimensions |

### Wrong tool — where this index fights the access pattern

| Use case | Why it struggles |
|---|---|
| High-cardinality columns (user_id, email) | One bitmap per unique value → storage explosion |
| High-write OLTP tables | Writers on same-value rows contend on the same bitmap |
| Frequent value updates | Every change updates two bitmaps (old value off, new value on) |
| Point lookups by unique key | B+tree or hash wins; bitmap provides no benefit |
