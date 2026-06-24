# Prerequisites for DB Selection: Index Data Structures & Access Patterns

Three things determine which database you should use — how it stores and retrieves data (performance), what it can do for you beyond raw speed (functional), and how it behaves when operated at scale (operational). SQL vs NoSQL answers none of these questions.

---

## Section 1: Performance — Storage Engines & Index Fit

> Full internals, per-index good/bad breakdowns, and production use cases: [Index Internals.md](./Index%20Internals.md)

### Index Types at a Glance

| Index | Built for | Wins when | Loses when |
|---|---|---|---|
| **B+Tree** | Mixed OLTP, range + equality | Reads and writes are balanced, data is single-valued per row | Write-heavy streams, multi-valued columns, low-cardinality filters |
| **Hash** | Pure equality, in-memory speed | You only ever ask `WHERE key = X` | Any range, prefix, or sort needed |
| **LSM Tree** | Write-dominant, append-heavy | Writes vastly outnumber reads, data is time-ordered | Random reads on cold data, frequent updates to same key |
| **GIN** | Multi-valued data (arrays, JSONB, full-text) | One row has many indexable things (tags, tokens, nested fields) | High write rate, single scalar equality |
| **Bitmap** | Low-cardinality multi-filter analytics | Read-heavy, `WHERE a=x AND b=y AND c=z` on categorical columns | High-write OLTP, high-cardinality columns |

### The One-Line Mental Model

B+tree is a generalist — it pays a cost to keep options open. Every other index is a specialist that wins hard in one scenario by refusing to support the others.

### Access Pattern → Index Fit

| Access pattern | Right index | Wrong index (and why) |
|---|---|---|
| `WHERE id = 42` | Hash (O(1)) or B+tree | Bitmap (explodes on high cardinality) |
| `WHERE created_at BETWEEN t1 AND t2` | B+tree | Hash (no order), Bitmap (wrong tool) |
| `WHERE tags @> ARRAY['go', 'redis']` | GIN | B+tree (no multi-value model) |
| 10k writes/sec, time-ordered keys | LSM | B+tree (random I/O bottleneck) |
| `WHERE status='A' AND region='B' AND tier='C'` | Bitmap | B+tree (separate scans, merge cost) |
| Full-text `@@ to_tsquery('backend')` | GIN | All others |

### Combinations Worth Knowing

| Combination | What it enables |
|---|---|
| **B+Tree + Adaptive Hash** (InnoDB) | B+tree for range/writes; auto-built in-memory hash for hot equality reads. Transparent, no config needed. |
| **LSM + Bloom Filters** (RocksDB, Cassandra) | Bloom filter per SSTable skips files that definitely don't contain a key. Turns read amplification from catastrophic to tolerable. |
| **B+Tree + GIN on same table** (Postgres) | B+tree on `id`, `created_at`; GIN on `tags`, `body_tsv`. Two separate structures, same table. Standard OLTP + search pattern. |
| **B+Tree + Runtime Bitmap** (Postgres BitmapAnd) | Planner scans two B+tree indexes, builds bitmaps in memory, ANDs them. Automatic — no configuration. |
| **LSM + Inverted Index** (Elasticsearch / Lucene) | LSM for write throughput on segment files; GIN-like inverted index within each segment. Segments merge like SSTables. |
| **Columnar + Bitmap** (ClickHouse, Redshift) | Column storage compresses low-cardinality data well; bitmap enables fast multi-filter scans on compressed columns. |
| **LSM + B+Tree split by layer** (TiDB / TiKV) | RocksDB (LSM) for raw write path; B+tree-based MVCC index on top for transactional reads. |
| **LSM + Columnar hybrid** (TimescaleDB, InfluxDB IOx) | Recent data ingested via LSM (fast write). Old data compacted and re-encoded as columnar chunks for analytics. |

### DB → Storage Engine Map

| DB | Storage engine | Index type underneath |
|---|---|---|
| Postgres | Custom heap + buffer manager | B+tree (default), GIN, GiST, BRIN, Hash |
| MySQL InnoDB | InnoDB | B+tree (clustered + secondary), adaptive hash |
| MongoDB | WiredTiger | B+tree |
| Cassandra | Cassandra engine | LSM (SSTables + MemTable) |
| RocksDB | RocksDB | LSM |
| Redis | In-memory | Hash tables, skip lists, zip lists |
| ClickHouse | MergeTree family | LSM-like merge + columnar + bitmap |
| Elasticsearch | Lucene | LSM segments + inverted index (GIN-like) |
| DynamoDB | Internal (B+tree-like) | B+tree on primary + GSI |

---

## Section 2: Functional & Operational Selection Factors

Performance fit narrows the category. These factors determine the actual pick — or override the performance-optimal choice entirely.

### Horizontal Scaling

| DB | Model | Shard key flexibility | Rebalancing |
|---|---|---|---|
| Postgres | Manual (Citus or app-level) | Full control, high effort | Manual |
| MongoDB | Native sharding via mongos | Hash or range on any field | Automatic chunk migration |
| Cassandra | Consistent hashing ring, no master | Partition key only — **immutable post-creation** | Automatic via token ring |
| CockroachDB | Raft-based range partitioning | Automatic | Automatic |
| DynamoDB | Fully managed | Partition + sort key — **immutable post-creation** | AWS-managed |
| Redis Cluster | Hash slot sharding (16384 slots) | Key-based, supports hash tags | Manual slot migration |

Cassandra and DynamoDB's partition key is immutable. Getting it wrong means a full table rewrite. Mongo's shard key choice affects hotspot risk — a monotonically increasing key (like ObjectId) funnels all writes to the same shard until it splits.

### Replication & Failover

| Model | DBs | Consistency guarantee | Failover time |
|---|---|---|---|
| Leader-follower | Postgres, MySQL, MongoDB replica sets | Strong on primary; eventual on replicas | ~30s (Patroni), ~10s (Mongo) |
| Leaderless (quorum) | Cassandra, DynamoDB | Tunable: ONE / QUORUM / ALL | No failover — any node takes writes |
| Raft consensus | CockroachDB, TiDB, etcd | Linearizable | Near-instant |
| Sentinel / Cluster | Redis | Eventual (async replication default) | 10–30s |

### Query Expressiveness

| DB | JOINs | Aggregations | Full-text | Verdict |
|---|---|---|---|---|
| Postgres | ✅ Full | ✅ Window functions, CTEs | ✅ GIN + tsvector | Most expressive |
| MongoDB | ⚠️ $lookup (correlated scan, not a real join) | ✅ Aggregation pipeline | ✅ Atlas Search | High, with JOIN caveat |
| Cassandra | ❌ | ❌ Client-side only | ❌ | Very constrained — query model = table design |
| DynamoDB | ❌ | ❌ Client-side | ❌ | Constrained — GSI/LSI limited and expensive |
| ClickHouse | ✅ Analytics-optimized | ✅ Best-in-class | ❌ | Analytics only |
| Redis | ❌ | ✅ Sorted sets, HyperLogLog | ❌ | Narrow but deep |

Cassandra's constraint is intentional: you design tables around queries, not data. Access pattern changes often require a new table. This is the price of LSM write throughput.

### Schema Evolution Cost

| DB | Add field | Change type | Rename | Immutable |
|---|---|---|---|---|
| Postgres | Fast (PG11+ metadata-only) | Slow (table rewrite) | Migration required | Nothing |
| MySQL InnoDB | Fast (online DDL) | Slow (table copy) | Migration required | Nothing |
| MongoDB | Free (write new field forward) | App-level | App-level | — |
| Cassandra | Fast (metadata) | New table required | New table required | Partition + clustering key |
| DynamoDB | Free (non-key attrs are schemaless) | App-level | App-level | Partition + sort key |

"Schema-less" doesn't mean schema changes are free. In Mongo or DynamoDB, old documents still have the old shape. The migration still happens — it's just in application code, unvalidated, and harder to audit.

### Operational Complexity

| DB | Self-host complexity | Key burden |
|---|---|---|
| Postgres | Low–moderate | Vacuum, replication lag, connection pooling (pgBouncer) |
| MongoDB | Moderate | Shard key choice, index bloat, oplog sizing |
| Cassandra | **High** | Compaction tuning, JVM heap, repair runs, tombstone accumulation |
| DynamoDB | None (fully managed) | Capacity planning (RCU/WCU), hot partition avoidance, GSI cost |
| Redis | Low (single) / Moderate (cluster) | Memory management, eviction policy, persistence config |
| ClickHouse | Moderate | MergeTree tuning, replica sync |
| Elasticsearch | **High** | JVM heap, shard sizing, index lifecycle, split-brain risk |

Cassandra is the most demanding open-source DB in widespread use. Tombstone accumulation from deletes causes read amplification that silently worsens over time. Repairs must run regularly or replicas diverge. Many teams migrated to ScyllaDB (C++ rewrite) to cut this burden.

### Ecosystem & Driver Maturity

| DB | Node.js | Go | ORM | Tooling |
|---|---|---|---|---|
| Postgres | ✅ pg, postgres.js | ✅ pgx (best-in-class) | ✅ Prisma, GORM | pgAdmin, DataGrip, pg_stat_statements |
| MongoDB | ✅ First-class (Mongoose, native) | ✅ mongo-go-driver | ✅ Mongoose | Compass, Atlas UI |
| MySQL | ✅ mysql2 | ✅ go-sql-driver | ✅ Sequelize, GORM | Workbench, DataGrip |
| Cassandra | ⚠️ Less maintained | ⚠️ gocql | ❌ | Limited |
| DynamoDB | ✅ AWS SDK v3 | ✅ AWS SDK Go v2 | ⚠️ DynamoDB-specific | NoSQL Workbench |
| Redis | ✅ ioredis | ✅ go-redis | N/A | RedisInsight |

MongoDB's Node.js driver is a legitimate selection factor. For teams deep in Node.js, the driver maturity and Mongoose ecosystem are a real edge over Cassandra or DynamoDB.

### Multi-Tenancy Patterns

| Pattern | How | DB fit | Isolation | Scale |
|---|---|---|---|---|
| Shared schema (row-level) | `tenant_id` on every table, RLS at DB level | Postgres (RLS), MySQL | Low | High |
| Schema-per-tenant | Separate schema, shared instance | Postgres | Medium | Medium |
| DB-per-tenant | Separate instance per tenant | MongoDB, Postgres | High | Low (ops overhead) |

Postgres Row-Level Security is underused for multi-tenancy. Policy defined once at DB level — application bugs cannot accidentally leak cross-tenant data.

### Cost Model

| DB | Expensive when | Cheap when |
|---|---|---|
| DynamoDB | Large scans, on-demand at high QPS, large item sizes | Predictable provisioned throughput, small items |
| MongoDB Atlas | Large cluster sizes; Atlas Search billed separately | Small–medium clusters, moderate workloads |
| Postgres (RDS) | Oversized instance for bursty workload | Right-sized, steady, predictable |
| Cassandra (self-hosted) | Ops engineering is the dominant cost | Massive sustained write scale on commodity hardware |
| ClickHouse Cloud | Complex queries at high concurrency | Batch analytics, infrequent queries |
| Redis (Upstash) | High QPS with large values | Low-frequency, small-value access |

DynamoDB on-demand pricing surprises teams. A table scan on 50M items can exceed a month of RDS costs. Access patterns must be locked in before you build — unlike Postgres, where you can add an index post-hoc.

### Vendor Lock-in

| DB | Lock-in | Portability |
|---|---|---|
| DynamoDB | High | AWS-specific API; PartiQL doesn't eliminate it |
| MongoDB Atlas | Medium | Self-hostable; API is Mongo-proprietary |
| Postgres | Low | Fully open source, runs everywhere |
| Cassandra | Low | Apache license, multiple managed providers |
| CockroachDB | Medium | Self-hosted is open-source core; CockroachCloud is proprietary |
| ClickHouse | Low | Apache license, self-hostable |

---

## Section 3: Takeaways

### Common Pitfalls

**On indexes:**

- **"I'll add an index to speed things up."**
  Indexes always slow writes. Every insert/update touches every index on that table. Adding 5 indexes means every write does 5 index updates. It's a read/write tradeoff, not a free win.

- **"LSM is always faster than B+tree."**
  LSM wins on writes. For random reads on cold data, B+tree often wins — one tree traversal to the leaf page vs checking multiple SSTables. Bloom filters reduce but don't eliminate the gap.

- **"Hash index is great for any equality lookup."**
  Hash indexes in some engines are in-memory only and not WAL-backed (pre-Postgres 10). They don't support `ORDER BY`, `LIKE prefix%`, or multi-column prefix scans. InnoDB's adaptive hash is automatic and opaque — you don't control when it builds or evicts.

- **"GIN on JSONB means I can query anything inside the document quickly."**
  GIN only indexes what you configure it to. Large, deeply nested documents cause index bloat. Write throughput drops sharply on high-update JSONB columns. It's a read-side optimization with a write tax.

- **"Bitmap indexes are great for filter-heavy queries."**
  Yes — in read-heavy OLAP. In OLTP, bitmap indexes cause write contention across all rows sharing a value. Postgres avoids this by never storing them — it builds bitmaps at query time from B+tree scans instead.

- **"More indexes = better query performance."**
  Past a certain point the query planner picks wrong (stale stats, low sample rate). Write throughput degrades linearly with index count. Indexes also consume buffer pool memory, evicting actual data pages from cache.

**On DB selection:**

- **"NoSQL = fast writes."**
  MongoDB uses B+tree. Redis uses hash tables and skip lists. Cassandra is fast at writes *because of LSM*, not because it's NoSQL. The label says nothing about the write path.

- **"SQL = ACID, NoSQL = eventual consistency."**
  CockroachDB, YugabyteDB, Spanner — fully SQL, distributed, linearizable. MongoDB has ACID since 4.0. Cassandra is NoSQL but consistency is tunable per operation. These are orthogonal axes.

- **"Schema-less = flexible and faster to develop on."**
  Schema-less means you moved schema enforcement from the database into application code. The schema still exists — it's invisible and unvalidated. You'll write migration scripts anyway, just without tooling.

- **"Elasticsearch is a database."**
  Elasticsearch is an index built on Lucene (LSM + inverted index). No transactions. Weak consistency by default. Using it as a primary source of truth is a known production anti-pattern. It belongs as a derived read replica, not the system of record.

- **"Redis is just a cache."**
  Redis has sorted sets, streams, pub/sub, Lua scripting, and optional persistence. It is used as a primary store for leaderboards, rate limiters, and job queues. Treating it as cache-only closes off real design options.

- **"Pick the database your team knows."**
  Team familiarity is a valid operational concern — but not the primary one. Running a time-series workload on Postgres because the team knows SQL will produce a production incident. The right tool with a learning curve beats the wrong tool with comfort.

### Production Lessons

> Patterns observed from engineering post-mortems, incident reports, and conference talks across the industry — not personal production experience.

- **Cassandra's partition key is permanent.** Wrong choice = full table rewrite at scale. Design it with access patterns locked first, not after.

- **MongoDB's $lookup is not a JOIN.** It's a correlated subquery. Applications with heavy joins should not use Mongo for that workload.

- **Postgres BitmapAnd is automatic.** You don't need to create bitmap indexes. The planner builds bitmaps at runtime when it detects multi-index AND queries. Knowing this tells you when to trust the planner and when to create a composite index instead.

- **DynamoDB scan = bill shock.** On-demand pricing + full table scan on 50M rows can exceed a month of RDS spend in a single query. Access pattern discipline is not optional.

- **GIN fastupdate is a time bomb on high-write tables.** Deferred inserts accumulate in the pending list. When vacuum flushes them, you get a latency spike. Disable fastupdate if write latency consistency matters more than throughput.

- **InnoDB adaptive hash is invisible until it isn't.** It silently builds hash indexes on hot B+tree pages. Under memory pressure, it evicts them — and the next cold read hits the B+tree again. If you see unpredictable latency spikes on high-read tables, this is one place to check.

---

---

## Section 4: System Design Interview Cheatsheet

This section is purely for interview use — pattern recognition and justification speed, not deep internals.

### The 5-Question Mental Model

1. What are the access patterns? (equality / range / containment / aggregation)
2. What is the read:write ratio?
3. What is the cardinality of filter columns?
4. What consistency level is required?
5. Single-node or distributed?

---

### Scenario → DB/Index Cheatsheet

| Scenario | DB choice | Index / structure | Reason |
|---|---|---|---|
| URL shortener | Redis or Postgres | Hash on short code | Pure equality lookup — no range needed, O(1) is the goal |
| Rate limiter | Redis | Hash + atomic INCR / Lua | In-memory counter, sub-millisecond, atomic ops |
| Notification / feed system | Cassandra | LSM + time-based partition key | Write-heavy, time-ordered, fan-out to many users |
| Real-time leaderboard | Redis | Sorted set (skip list) | Range by score + rank in O(log n) — B+tree adds overhead |
| Time-series metrics | InfluxDB / Cassandra / TimescaleDB | LSM | Sequential append, time-ordered keys minimize compaction cost |
| Full-text product search | Postgres (moderate) / Elasticsearch (large) | GIN / inverted index | Multi-keyword containment — B+tree has no model for this |
| Analytics dashboard (low-card filters) | ClickHouse / Redshift | Columnar + Bitmap | `region AND status AND tier` = bitwise AND, read-only |
| User profile CRUD | Postgres / MySQL | B+tree | Mixed reads/writes, relational, ACID |
| Event/audit log | Cassandra / Kafka + RocksDB | LSM | Append-only, high write throughput, time-range reads |
| Typeahead / autocomplete | Postgres / Redis | B+tree prefix scan or Trie | `LIKE 'prefix%'` uses B+tree leading-edge range scan |
| Session store | Redis | Hash | Equality-only on token, in-memory, TTL support |
| Job queue / task scheduler | Redis / Postgres | Sorted set / B+tree | Priority ordering or scheduled execution time |
| Graph relationships | Neo4j / Postgres (recursive CTE) | Adjacency + native graph index | B+tree isn't traversal-optimized; graph DBs store adjacency natively |
| Vector similarity search | pgvector / Qdrant | HNSW / IVF | Approximate nearest neighbour — B+tree doesn't operate in vector space |

---

### Quick Elimination Table

| If you need... | Eliminate immediately |
|---|---|
| Range queries | Hash index |
| Write throughput > 5k/sec sustained | B+tree as primary store |
| Multi-keyword or array containment | B+tree, Hash, LSM without an inverted index layer |
| Low-cardinality multi-filter (OLAP) | LSM, GIN, Hash |
| Strong ACID across tables | Cassandra, DynamoDB (no multi-table transactions) |
| Flexible schema with heavy JOIN requirements | MongoDB |
| In-memory sub-millisecond latency | Any disk-based DB as primary |
| Graph traversal (3+ hops) | Any non-graph DB without recursive CTE support |

---

*One-line principle: A database is a combination of storage engine, data model, and consistency model. The query language is the interface. Match the index to the access pattern — everything else follows.*
