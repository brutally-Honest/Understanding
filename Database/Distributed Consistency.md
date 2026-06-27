# Distributed Systems Consistency: ACID, BASE, CAP & PACELC

---

## 1. Why These Concepts Exist

Early databases ran on a single machine. One server, one disk, one process managing all reads and writes. Correctness was straightforward — if a write happened, the next read saw it. There was no disagreement between nodes because there was only one node.

As systems scaled, data had to be spread across multiple machines — for fault tolerance, throughput, and geographic reach. The moment you do that, a new class of problems emerges:

- What if two nodes disagree on the current value?
- What if the network between them drops mid-write?
- What if one node is slow and another is fast — which one do you trust?

These aren't bugs. They're fundamental consequences of distributing state across machines connected by an unreliable network.

**ACID** was formalized to describe what guarantees a database should provide to applications — specifically around transactions. Coined in 1983 by Härder and Reuter.

**BASE** emerged later as distributed systems (Cassandra, DynamoDB, CouchDB) chose availability over strict consistency. It described the *actual* behavior of those systems honestly, rather than pretending they could offer ACID.

**CAP theorem** was proposed by Eric Brewer in 2000, formally proven by Gilbert and Lynch in 2002. It put a theoretical ceiling on what any distributed system can guarantee: you cannot have Consistency, Availability, and Partition Tolerance all at once.

**PACELC** was proposed by Daniel Abadi in 2010. He pointed out that CAP only describes behavior during a partition — a failure scenario. What about the 99.9% of time when everything is healthy? Even then, every distributed system is trading latency against consistency. PACELC models the full picture.

---

## 2. ACID and BASE — Different Scopes, Not Alternatives

ACID and BASE are frequently placed in a side-by-side comparison table as if they are two options for the same decision. They are not. They operate at entirely different levels.

| | ACID | BASE |
|---|---|---|
| **Scope** | Single transaction | Entire distributed system across nodes |
| **Answers the question** | What happens within this one operation? | What does the system guarantee across all nodes over time? |
| **Unit** | One transaction, one DB (or one node) | All nodes, all replicas, over time |
| **Is it a transaction model?** | Yes | No |

**ACID** is a contract between a database and an application about how individual transactions behave. It says nothing about what happens across multiple nodes.

**BASE** is a description of how a distributed system behaves at the system level. It is not a transaction model. Most BASE systems either have no multi-row transactions at all, or offer them as an opt-in at a smaller scope. BASE describes the *consequence* of choosing availability and scale over strict coordination — not a competing transaction protocol.

They connect indirectly: when a system cannot afford the coordination overhead required for ACID across distributed nodes, it ends up exhibiting BASE behavior at the system level. But a system can offer ACID within a single node or shard and still be BASE across the cluster. MongoDB is exactly this — ACID within a document or session-scoped transaction, BASE behavior across replica nodes by default.

**The comparison that actually makes sense:**

| Question | Options |
|---|---|
| Does this operation need atomic, isolated execution? | ACID transaction (within a node/shard) vs no transaction |
| What does the whole system guarantee across nodes? | Strong consistency (CP) vs Eventual consistency (AP/BASE) |

These are two separate decisions. ACID vs BASE is not one decision.

---

### ACID — Transaction-Level Guarantees

ACID describes what a database promises about individual transactions. Scoped to one operation.

### Property by Property

| Property | What it means | What enforces it |
|---|---|---|
| **Atomicity** | A transaction either fully completes or fully rolls back. No partial writes. | Rollback logs, WAL (Write-Ahead Log) |
| **Consistency** | DB moves from one valid state to another. No constraint violations mid-transaction. | Schema constraints, triggers, foreign keys |
| **Isolation** | Concurrent transactions do not interfere. Each sees a consistent snapshot. | Locks, MVCC (Multi-Version Concurrency Control) |
| **Durability** | Once committed, the write survives crashes. | WAL flushed to disk before ACK is sent |

**Isolation has levels** — this is a hidden dial most engineers ignore:

| Level | Prevents | Allows | Default in |
|---|---|---|---|
| Read Uncommitted | Nothing | Dirty reads, phantom reads | Rarely used |
| Read Committed | Dirty reads | Non-repeatable reads | PostgreSQL, Oracle |
| Repeatable Read | Dirty + non-repeatable reads | Phantom reads | MySQL InnoDB |
| Serializable | Everything | Nothing | Opt-in only |

Higher isolation = fewer anomalies = more lock contention = slower.

---

### BASE — System-Level Behavioral Description

BASE describes how the distributed system as a whole behaves across nodes and over time. It is not scoped to any operation.

| Property | What it means | Real example |
|---|---|---|
| **Basically Available** | System always responds, but response may be stale | Read from a Cassandra node that hasn't caught up |
| **Soft State** | System state can change without new input, as replicas catch up | Two Redis replicas temporarily disagree on a value |
| **Eventually Consistent** | Given no new writes, all replicas converge to the same value — eventually | DNS propagation after an A record update |

BASE is not a transaction model. It has no concept of atomicity, isolation, or rollback. It describes the observable behavior of a distributed system that has chosen availability over strict per-read consistency.

---

### When ACID Transactions Are Required vs When BASE Behavior Is Acceptable

These are two separate questions — one about operation scope, one about system behavior.

| Question | Requires ACID | BASE behavior acceptable |
|---|---|---|
| Does this operation touch multiple rows/documents atomically? | ✓ | |
| Is wrong data a financial or legal liability? | ✓ | |
| Is wrong data a mildly stale UI element? | | ✓ |
| Does the system need to stay up regardless of node failures? | | ✓ |
| Strong regulatory / audit trail required? | ✓ | |
| Write throughput > what a single primary can handle? | | ✓ |

A BASE-natured distributed system can still offer ACID transactions at a smaller scope — within a single node, shard, or document. MongoDB offers ACID within a session-scoped transaction. DynamoDB offers ACID within a single partition key. The BASE behavior emerges only when crossing node boundaries. These are not contradictions — they are scoped guarantees.

---

## 3. Where This Applies

These concepts are not database-specific. They apply to **any system that holds shared mutable state across multiple nodes connected by a network**.

| System type | CAP applies? | Why |
|---|---|---|
| Single-node PostgreSQL | No (CA, not distributed) | No replication, no partition possible |
| PostgreSQL with read replicas | Yes | Async replication → stale reads possible |
| MongoDB replica set | Yes | Primary + secondaries replicate state |
| Redis (single node) | No | One source of truth, SPOF not CAP |
| Redis with replicas | Yes | Async replication, AP behavior |
| Redis Cluster | Yes | Sharded + replicated, CP by default |
| Kafka brokers | Yes | Replicated partition logs across brokers |
| Stateless app servers (Node.js) | No | No shared state between instances |
| Stateful app servers (in-memory cache) | Yes | Each server holds different state |
| Microservices calling each other | Yes | Service A caching Service B's data = shared state problem |
| Zookeeper / etcd | Yes | Distributed coordination state |

**The rule**: wherever you replicate mutable state across nodes connected by a network, CAP applies. The layer doesn't matter — database, cache, broker, or application.

---

## 4. Distributed Systems Problems

Before CAP and PACELC make full sense, three core problems need to be clear.

**Network Partitions**
A network partition means some nodes can reach each other, but not all. Not a total outage — a split. Node A and B can talk. Node C is cut off. All three are still running, still receiving client requests.

Partitions happen due to: network hardware failure, misconfigured firewall rules, AZ (availability zone) outages, packet loss, cloud provider hiccups. They are not rare edge cases. They are expected.

**Replication Lag**
When data is written to one node and needs to propagate to others, there is always a window of time where nodes disagree. Synchronous replication (wait for all nodes to confirm) eliminates this window but adds latency. Asynchronous replication (fire and forget to replicas) keeps latency low but creates a staleness window.

**Split Brain**
When a partition causes two groups of nodes to each believe they are the authoritative primary, and both accept writes independently. When the partition heals, the writes conflict. Both sides thought they were right. Resolving this requires a merge strategy — Last-Write-Wins, vector clocks, CRDTs, or manual intervention.

---

## 5. CAP Theorem

**Claim**: A distributed system can guarantee at most 2 of the following 3 properties simultaneously.

| Property | Meaning |
|---|---|
| **Consistency (C)** | Every read returns the most recent write, or an error. Not stale — the *latest* value. |
| **Availability (A)** | Every request gets a non-error response. May be stale. Never turns the client away. |
| **Partition Tolerance (P)** | System continues operating even if some nodes cannot communicate. |

---

### The Scenario

**Setup**: 3 servers — S1, S2, S3. All store the same data.

Initial state on all three: `balance = 100`

A client writes `balance = 50` to S1. S1 replicates to S2 — succeeds. S1 tries to replicate to S3 — fails. S3 is partitioned.

A second client immediately reads `balance` from S3.

**What does S3 do?**

---

### CP — Consistency + Partition Tolerance

S3 knows it is cut off. It cannot confirm whether it missed a write. It refuses the read.

```
Client → S1 : write balance = 50
S1     → S2 : replicate (success)
S1     → S3 : replicate (FAIL — partitioned)

Client → S3 : read balance
S3     → Client : ERROR — cannot guarantee fresh data
```

No client ever receives wrong data. But S3 was unavailable during the partition.

**What you sacrificed**: Availability. During a partition, S3 turns clients away rather than risk a stale response.

| | CP Behavior |
|---|---|
| Write to S1 | Succeeds |
| Replication to S2 | Succeeds |
| Replication to S3 | Fails (partition) |
| Read from S3 during partition | ERROR |
| Read from S3 after partition heals | `50` (correct) |

**Real systems**: Zookeeper, etcd, HBase, Redis Cluster

---

### AP — Availability + Partition Tolerance

S3 is cut off but does not refuse. It serves whatever it has locally.

```
Client → S1 : write balance = 50
S1     → S2 : replicate (success)
S1     → S3 : replicate (FAIL — partitioned)

Client → S3 : read balance
S3     → Client : 100   ← stale, wrong, but a response
```

S3 had no way to know it missed a write. It responded. It was available. The data was wrong.

When the partition heals, S1/S2/S3 reconcile. S3 eventually gets `balance = 50`. Until then, reads from S3 return stale values.

| | AP Behavior |
|---|---|
| Write to S1 | Succeeds |
| Replication to S2 | Succeeds |
| Replication to S3 | Fails (partition) |
| Read from S3 during partition | `100` (stale) |
| Read from S3 after partition heals | `50` (converged) |

**What you sacrificed**: Consistency. Clients can get stale data during partitions.

**Real systems**: Cassandra, DynamoDB (default), CouchDB, MongoDB (default)

---

### CA — Consistency + Availability (No Partition Tolerance)

If a partition happens, the entire system shuts down or stops accepting requests. No partial service. Everything waits until the partition heals.

In practice: the moment you have more than one server, a partition is possible. Choosing CA means "if any partition happens, I go fully down." That is a valid strategy only for a single-node system — which has no partition by definition.

| | CA Behavior |
|---|---|
| Write to S1 | Succeeds |
| Replication to S3 | Fails (partition) |
| Response to all clients | System goes offline until partition heals |

**Real systems**: Single-node PostgreSQL, single-node MySQL. Not distributed systems.

---

### CAP Summary Table

| Combination | During partition | Normal ops | Real examples |
|---|---|---|---|
| **CP** | Refuse requests, return errors | Consistent reads | etcd, Zookeeper, HBase, Redis Cluster |
| **AP** | Serve stale data | Eventually consistent | Cassandra, DynamoDB, CouchDB, MongoDB (default) |
| **CA** | System goes fully down | Consistent + available | Single-node RDBMS only |

---

### The Practical Takeaway from CAP

Partition tolerance is not optional in a distributed system. Partitions will happen. Choosing "no partition tolerance" means choosing to go fully offline when they do. That is not a distributed system.

**The real binary choice is CP or AP** — when a partition hits, do you want correctness or continued availability?

---

## 6. PACELC

**CAP's gap**: it only describes what happens during a partition. What about normal operation — when all nodes are up, the network is healthy, and everything is working?

Even then, you face a tradeoff.

**PACELC states**: During a **P**artition, choose **A** or **C** (same as CAP). **E**lse (normal operation), choose **L**atency or **C**onsistency.

Every distributed system gets two labels: one for partition behavior, one for normal behavior.

---

### The Normal Operation Scenario

**Setup**: Same S1, S2, S3. No partition. All healthy. Client writes `balance = 50` to S1.

S1 must replicate to S2 and S3. It has two options.

---

#### EL — Else, Low Latency (Async Replication)

S1 writes locally, immediately returns `OK` to client. Replicates to S2 and S3 in the background.

```
Client → S1 : write balance = 50
S1 writes locally
S1     → Client : OK   (fast — returned before replicas confirmed)

[async, happens after]
S1 → S2 : replicate balance = 50
S1 → S3 : replicate balance = 50
```

A second client reads from S2 one millisecond after the write. S2 may not have the update yet.

```
Client → S2 : read balance
S2     → Client : 100   ← stale, replication not complete yet
```

Low latency. Possible stale reads. This is EL behavior.

| | EL (Low Latency) Behavior |
|---|---|
| Write to S1 | Returns immediately |
| Replication to S2, S3 | Async, in background |
| Read from S2 immediately after | May return stale value |
| Read from S2 after replication completes | Correct value |

---

#### EC — Else, Consistency (Sync Replication)

S1 writes locally, waits for S2 and S3 to confirm receipt. Only then returns `OK`.

```
Client → S1 : write balance = 50
S1 writes locally
S1 → S2 : replicate balance = 50 (waits for ACK)
S1 → S3 : replicate balance = 50 (waits for ACK)
S1 → Client : OK   (slower — all replicas confirmed)
```

Any read from any server after this returns `50`. No stale reads.

```
Client → S2 : read balance
S2     → Client : 50   ← correct
```

Higher write latency. Consistent reads from any node.

| | EC (Consistency) Behavior |
|---|---|
| Write to S1 | Waits for all replicas to ACK |
| Replication to S2, S3 | Sync, blocking |
| Read from S2 immediately after | Correct value always |
| Write latency | Higher (slowest replica is the bottleneck) |

---

### Full PACELC Classification

| System | During Partition | Normal Ops | Label | Notes |
|---|---|---|---|---|
| Cassandra | Available (serves stale) | Low latency (async) | **PA/EL** | Tunable via consistency level |
| DynamoDB (default) | Available | Low latency | **PA/EL** | Strong consistency opt-in available |
| Riak | Available | Low latency | **PA/EL** | |
| HBase | Consistent (refuses) | Consistent (sync) | **PC/EC** | |
| Google Spanner | Consistent | Consistent | **PC/EC** | Uses TrueTime for global sync |
| VoltDB | Consistent | Consistent | **PC/EC** | |
| MongoDB (default) | Available | Low latency | **PA/EL** | Async replication, read from primary |
| MongoDB (tuned) | Available | Consistent | **PA/EC** | writeConcern: majority + readConcern: majority |
| Yahoo! PNUTS | Consistent | Low latency | **PC/EL** | Per-record master, local reads |
| Zookeeper | Consistent | Consistent | **PC/EC** | Coordination service, correctness critical |

---

### CAP vs PACELC

| | CAP | PACELC |
|---|---|---|
| **Covers** | Partition behavior only | Partition + normal operation |
| **Normal ops tradeoff** | Not modeled | Latency vs Consistency |
| **When it matters most** | Failure scenarios | Every day, every write |
| **Best used for** | High-level system classification | Actual tuning and design decisions |
| **Limitation** | Incomplete picture | More dimensions to reason about |

---

### Cassandra — Tunable Consistency (PA/EL in depth)

Cassandra lets you move your PACELC position per query via **consistency level**.

| Write CL | Read CL | Behavior |
|---|---|---|
| ONE | ONE | Fastest. Most stale. Fully EL. |
| QUORUM | QUORUM | Write and read overlap guaranteed. Strong eventual consistency. |
| ALL | ONE | Maximum write durability. Slower writes. |
| ALL | ALL | Effectively EC. Slowest. Any node down = failure. |

QUORUM = ⌈n/2⌉ + 1. For 3 replicas: QUORUM = 2.

QUORUM write + QUORUM read guarantees at least one node in the read set received the write. This is strong eventual consistency — not linearizability. There is no single global ordering of events, but no completed write will be missed.

---

## 7. Production Scenarios and Use Cases

These are reasoning frameworks — the right answer always depends on the cost of wrong data versus the cost of latency or unavailability.

---

### Payment / Money Movement

**Core question**: What is worse — a failed transaction or a double debit?

A failed transaction is a recoverable user experience problem. A double debit is a financial and trust problem.

| Decision | Choice | Reason |
|---|---|---|
| Transaction requirement | ACID within the DB | Money movement must be atomic and isolated |
| System consistency | CP | During partition, reject the request rather than risk a double debit |
| PACELC | PC/EC | Pay write latency cost, not correctness cost |
| DB choice | PostgreSQL, CockroachDB, Spanner | |

**Production gotcha**: Distributed transactions across services are tempting but dangerous. Two-phase commit (2PC) is slow, and if the coordinator dies mid-transaction, you get a blocking lock. Most teams use sagas + compensation logic instead — ACID within each service, compensating transactions across services.

---

### Social Feed / Activity Counts

**Core question**: Does it matter if a user sees 1,243 likes instead of 1,244?

No. The business cost of stale data here is zero.

| Decision | Choice | Reason |
|---|---|---|
| Transaction requirement | None needed | No atomicity required for a like count or feed append |
| System consistency | AP | Feed must always load — staleness has no business impact |
| PACELC | PA/EL | Optimize for write throughput and read speed |
| DB choice | Cassandra, DynamoDB, Redis | |

**Production note**: Write amplification is the real cost here. Fanout-on-write (pre-computing feeds) vs fanout-on-read (aggregating at read time) is the main tradeoff. Both are AP decisions.

---

### IoT Sensor Ingestion + Alerting

Two operations with different consistency requirements in the same system.

**Ingestion path** — high velocity, write-heavy:

| Decision | Choice | Reason |
|---|---|---|
| Transaction requirement | None needed | Individual sensor writes are independent |
| System consistency | AP | Ingest must not block |
| PACELC | PA/EL | Async writes, batch replication |
| DB choice | Kafka → TimescaleDB / InfluxDB / MongoDB |

**Alert evaluation path** — reading to make a decision:

| Decision | Choice | Reason |
|---|---|---|
| Read preference | Primary only | Cannot read from a stale replica for threshold checks |
| Pattern | Separate consumer on Kafka stream | Stream is ordered and durable — avoids replica staleness entirely |
| DB read concern (MongoDB) | majority | Guarantees the read reflects committed state |

The key insight: you do not need one consistency model for the whole system. Async ingestion (BASE) + synchronous evaluation (closer to CP) in the same pipeline is a pattern, not a contradiction.

---

### Inventory / Seat Booking (The Oversell Problem)

Last seat problem: two users see 1 seat available, both book, both get confirmations.

| Option | Mechanism | Tradeoff |
|---|---|---|
| **Pessimistic locking** | Lock the row for the duration of the transaction | No oversell. Lock contention under load. |
| **Optimistic concurrency** | Both proceed, one fails with version conflict, retries | No locks. Retry storms under contention. |
| **Allow oversell + compensate** | Accept both bookings, handle downstream | Requires business process for compensation (upgrade, refund). Airlines do this. |
| **Redis atomic gate + DB persistence** | Redis DECR via Lua script reserves fast, DB persists async | Fast gate, durable record, eventual consistency between gate and DB. |

The question to answer before picking: is overselling worse than "checkout is slow during peak traffic"? That is a business question. The answer determines the tradeoff, not any theorem.

---

### Microservices Calling Each Other

Service A depends on Service B's data. Service A caches Service B's response locally.

This is CAP at the service layer, not the database layer.

| Scenario | Behavior | Classification |
|---|---|---|
| B is down, A serves from cache | Stale but available | AP |
| B is down, A returns error | Unavailable but correct | CP |
| A invalidates cache on every read (calls B always) | Fresh but high latency | EC |
| A uses cached value with TTL | Low latency, possibly stale | EL |

The moment a service caches another service's data, it becomes a distributed system with replicated state. CAP and PACELC apply.

---

### Your Stack Reference

| System | Default behavior | Classification | Key tuning |
|---|---|---|---|
| MongoDB (default) | Async replication, reads from primary preferred | PA/EL | writeConcern: majority + readConcern: majority → PA/EC |
| Redis (single node) | No replication | CA (single node, SPOF) | Add replicas to change this |
| Redis with replicas | Async replication, stale reads from replicas | PA/EL | Read-your-own-writes: route to primary after a write |
| Redis Cluster | Refuses writes during split | CP | Prefers consistency during partition |
| Kafka | Refuses producers if under-replicated | CP | min.insync.replicas controls this |

---

## 8. Takeaways

**ACID and BASE are not alternatives at the same level.** ACID is a transaction-level contract scoped to one operation. BASE is a system-level behavioral description scoped to the entire distributed cluster over time. A system can offer ACID within a shard and exhibit BASE behavior across shards simultaneously. Comparing them directly misframes the decision.

**Partition tolerance is not optional.** In a distributed system, partitions will happen. Choosing to not tolerate them means going fully offline when they occur. The real choice in CAP is CP or AP.

**CAP tells you what happens when things break. PACELC tells you what happens every day.** The EL vs EC tradeoff (async vs sync replication) affects every write in normal operation. This is usually more impactful than partition behavior.

**Consistency is a dial, not a switch.** Cassandra's tunable consistency levels, MongoDB's write/read concern, PostgreSQL's isolation levels — these are all ways to move position on the same spectrum between strict consistency and high throughput.

**The right model depends on the cost of wrong data vs the cost of unavailability or latency.** That is a business question first, an engineering question second. Knowing the tradeoffs lets you ask the right business question.

**CAP classifies systems. PACELC classifies decisions.** You use CAP to describe what a system is. You use PACELC to make decisions about how to configure and use it.
