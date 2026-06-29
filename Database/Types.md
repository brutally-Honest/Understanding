# Database Types

---

## 1. SQL

**What it is**
A database paradigm built on the relational model. Edgar Codd, 1970. The insight: describe *what* you want, not *how* to navigate to it. Data lives in tables — rows are records, columns are typed attributes. Relationships expressed via foreign keys. Schema defined and enforced before any data goes in. SQL (Structured Query Language) is the standardized declarative interface to interact with it.

**Nuances worth knowing**

- Schema is a contract enforced at the DB level. If a column is `NOT NULL`, the DB rejects the write — no negotiation.
- Joins are first-class. The query planner figures out the optimal join strategy. Multi-table joins in a single query are native.
- ACID (Atomicity, Consistency, Isolation, Durability) by default. A transaction either fully commits or fully rolls back — no half-states.
- Scales vertically well. Horizontal scaling (sharding across machines) is painful because joins across shards are expensive and consistency guarantees break down.
- Decades of tooling, optimization, and operational knowledge behind it.

---

## 2. NoSQL

**What it is**
Not Only SQL — umbrella term for any DB that doesn't follow the relational model. Not a single thing. Born in the 2000s when web-scale companies (Google, Amazon, Facebook) hit two walls: relational schemas too rigid for fast-moving products, and horizontal scaling across thousands of machines was painful. Different teams built different specialized systems for different data patterns. The commonality: not relational.

**Nuances worth knowing**

**NoSQL does not mean schemaless.** This is the most common misconception.
- MongoDB has `$jsonSchema` validators — you can enforce structure as strictly as you want.
- Cassandra enforces column types and column families.
- DynamoDB has typed attributes.

The difference: schema is *optional and app-enforced* rather than *mandatory and DB-enforced*. You choose how strict to be. The DB won't stop you from being disciplined.

**What it actually means: the schema lives in the application layer**, not the database layer. The flexibility is real — but flexibility isn't the same as chaos.

- Horizontal scaling is native. Systems like Cassandra and DynamoDB were designed from day one to run across hundreds of nodes.
- Consistency is usually eventual — a write may not be immediately visible across all nodes. Some NoSQL DBs now support transactions (MongoDB 4.0+, DynamoDB transactions) but it's not the default design assumption.
- No standardized query language. Each type has its own interface.

---

## 3. SQL vs NoSQL — Practical Differences

| Property | SQL | NoSQL |
|---|---|---|
| Schema enforcement | DB-enforced, rigid | App-enforced, flexible or strict — your choice |
| Schema migration | `ALTER TABLE` — can lock tables for hours on large datasets; coordinated deploys required | Additive changes are transparent; old and new document shapes coexist without breaking anything |
| App behavior on schema change | Remove a column the app depends on → runtime crash. Add `NOT NULL` column without default → existing rows break | Add new field → old documents simply don't have it; app handles both shapes; no crash |
| Joins | Native, query-planner-optimized, multi-table in one query | Either embed data (denormalize), or application-level join (fetch from two collections, join in code), or multiple round trips. `$lookup` in MongoDB exists but is expensive. Wide-column and Key-Value have no join concept at all. |
| Transactions | ACID by default | Eventual consistency by default; some support transactions now (MongoDB 4.0+, DynamoDB) but with caveats |
| Scaling | Vertical (scale up the machine); horizontal is painful | Horizontal native (add machines); vertical less relevant |
| Data duplication | Avoided — normalize into separate tables | Accepted or encouraged — denormalize for read performance |
| When a field is missing | DB rejects write if constraint violated | Document just doesn't have that field — DB accepts it |
| Query language | SQL — standardized across vendors | Type-specific; no universal standard |

**The real mental model shift**: SQL treats data consistency as a DB responsibility. NoSQL shifts that responsibility to the application. Neither is wrong — they're different contracts.

---

## 4. NoSQL Types

| Type | What it is | Core Data Structure | Key Properties | Best for | Examples |
|---|---|---|---|---|---|
| **Document** | JSON-like blobs. One document = one entity, self-contained. Fields vary per document. | B-tree for indexes; BSON/JSON on disk | Flexible schema; rich nested queries; no native joins | User profiles, catalogs, CMS (Content Management Systems) | MongoDB, Firestore, CouchDB |
| **Key-Value** | Simplest model. Key → opaque value. DB knows nothing about what's inside the value. | Hash table (in-memory); B-tree or LSM (Log-Structured Merge) tree on disk | No schema; exact key lookup only; extremely fast | Caching, sessions, rate limiting, leaderboards | Redis, Memcached, DynamoDB |
| **Wide-Column** | Looks like a table but each row can have completely different columns. Built for massive write throughput. | LSM tree + SSTables (Sorted String Tables) | Sparse rows; column families; masterless; sequential disk writes | IoT events, logs, messaging, time-series adjacent | Cassandra, HBase, Bigtable, ScyllaDB |
| **Graph** | Nodes (entities) + edges (relationships). Relationships are first-class, not foreign keys. Traversal is O(1) per hop. | Property graph / adjacency list | Relationship-first; multi-hop traversals cheap; poor at aggregates | Social graphs, fraud detection, recommendations | Neo4j, Amazon Neptune |
| **Time-Series** | Append-only, time-stamped data. Assumes writes always go to latest timestamp. Aggressive compression on adjacent similar values. | LSM tree + columnar compression (TSM — Time-Structured Merge) | Immutable appends; range scan + aggregation queries; excellent compression | Metrics, IoT sensors, financial ticks | InfluxDB, Prometheus, TimescaleDB |
| **Vector** | Stores high-dimensional numeric embeddings. Query by similarity, not equality. | HNSW (Hierarchical Navigable Small World) graph or IVF (Inverted File Index) | ANN (Approximate Nearest Neighbor) search; metadata filtering | Semantic search, RAG (Retrieval-Augmented Generation) pipelines, recommendations | Pinecone, Qdrant, Weaviate, pgvector |

**Why they evolved**: each type emerged because a specific query pattern was too painful on relational DBs — not because engineers wanted variety.

---

## 5. OLTP vs OLAP

**OLTP (Online Transaction Processing)** — the application layer. Serves live users. Many small, fast, concurrent reads and writes. One user, one transaction, milliseconds. Correctness per operation is critical — a half-written bank transfer is a disaster.

**OLAP (Online Analytical Processing)** — the analytics layer. Serves analysts and BI (Business Intelligence) tools. Few, massive, complex queries. Reads millions of rows. Aggregates, group-bys, window functions. Seconds to minutes is acceptable.

They were historically the same system. The problem: a "revenue by region, last year" query running on the same DB handling live orders caused full table scans, locked rows, and killed application performance. The answer was separation — copy data into a dedicated system optimized purely for reads. Data warehouses were born: Teradata in the 80s, then the cloud generation (Redshift 2012, BigQuery 2011). The copy step is an ETL (Extract, Transform, Load) pipeline. It introduces lag — warehouse data is always slightly stale.

| Property | OLTP | OLAP |
|---|---|---|
| Full form | Online Transaction Processing | Online Analytical Processing |
| Who uses it | Application serving live users | Analysts, BI tools, dashboards |
| Query shape | Narrow, indexed, point lookup — "get user 42's orders" | Wide, full scan, aggregation — "revenue by city, last 6 months" |
| Rows touched per query | Few (1–100) | Millions |
| Latency target | Milliseconds | Seconds to minutes |
| Write pattern | High-frequency, small, concurrent | Bulk ETL loads, infrequent |
| Consistency need | High — ACID | Low — read-only, slightly stale is acceptable |
| Typical storage | Row-oriented | Columnar |
| Examples | Postgres, MySQL, MongoDB | BigQuery, Redshift, ClickHouse, Snowflake |

**HTAP (Hybrid Transactional/Analytical Processing)** — both in one system. Eliminates the ETL copy step. Run analytics on live transactional data. NewSQL systems (CockroachDB, TiDB, Google Spanner) play here. You pay distributed coordination cost but gain real-time analytics. Increasingly relevant when the lag from the ETL pipeline becomes a product constraint.

---

## 6. Row vs Columnar

**Row-oriented** — all fields of one row stored together on disk. Write one row, one sequential append. Read one full record, one seek. Natural fit for OLTP.

**Columnar** — all values of one column stored together. Reading "average salary across 10M users" touches only the salary column, not every field of every row. Natural fit for OLAP.

> Analogy: Row = employee folders, one folder per person. Columnar = HR ledgers, one ledger per attribute — all salaries in one place, all departments in another.

| Property | Row-Oriented | Columnar |
|---|---|---|
| Disk layout | All fields of a row together | All values of a column together |
| Fast for | Fetching a full record | Aggregating one column across many rows |
| Slow for | Column-level scans across millions of rows | Reconstructing a full record |
| Write performance | Fast — one sequential append | Slower — N column files to update |
| Compression | Moderate | Excellent — similar values adjacent |
| Natural fit | OLTP | OLAP |
| Examples | Postgres, MySQL, MongoDB | BigQuery, Redshift, ClickHouse, Parquet |

Columnar compression works because sorted column values repeat — run-length encoding turns `[India, India, India...]` into `(India × 10000)`. 5–50x space savings is common. Less I/O, compressed columns fit in memory — this is why OLAP queries are fast despite touching millions of rows.

---

## 7. The Three Axes and How They Cluster

Three independent axes determine what kind of database you need:

- **Axis 1: Data Model** — how is data shaped and related?
- **Axis 2: Workload** — what kind of operations dominate?
- **Axis 3: Storage Layout** — how is data physically arranged on disk?

These are orthogonal in theory. In practice they cluster — because certain data shapes naturally pair with certain workloads, which naturally pair with certain storage layouts.

| Use Case | Data Model | Workload | Storage | Examples |
|---|---|---|---|---|
| App backend | Relational | OLTP | Row | Postgres, MySQL |
| Flexible backend | Document | OLTP | Row | MongoDB |
| Cache / sessions | Key-Value | OLTP | In-memory | Redis |
| Analytics / DWH | Relational* | OLAP | Columnar | BigQuery, Redshift |
| IoT / metrics | Wide-Column / Time-Series | Write-heavy | LSM / Columnar variant | Cassandra, InfluxDB |
| Traversal queries | Graph | Traversal | Property graph | Neo4j |
| Semantic search | Vector | Similarity | ANN index | Pinecone, Qdrant |
| Scale + SQL + ACID | Relational | OLTP + HTAP | Row | CockroachDB, Spanner |

*OLAP systems often expose SQL, but it's scan-optimized SQL — not transaction SQL.

Any new database you encounter: ask what data model, what workload, what storage layout. The answer is never a mystery.
