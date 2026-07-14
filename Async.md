# Async Messaging — Foundations & Evolution

This doc is scoped to async messaging: queues and pub/sub. A few sections here — ordering, delivery semantics, persistence, brokers — aren't messaging patterns themselves, but you can't pick the right broker without understanding them; they're the ground truth that queue/pub-sub decisions get made against. EDA closes the doc as the payoff, not a topic of its own — it's *why* any of this exists in the first place.

## 1. Sync vs Async

```
SYNC:
Client ──request──> Server
Client <──response── Server     (client blocked in between)

ASYNC:
Client ──msg──> [Queue/Broker] ──msg──> Consumer
Client continues immediately, doesn't wait
```

| | Synchronous | Asynchronous |
|---|---|---|
| Caller behavior | Blocks, waits for response | Fires and moves on |
| Coupling | Tight — both must be up, at the same moment | Loose — receiver can be down |
| Failure mode | Caller fails if callee fails | Caller unaffected, message just waits |
| Examples | REST/HTTP, gRPC unary, DB query | Kafka produce, SQS send, webhook fire |
| Consistency | Immediate | Eventual |

**Analogy:** sync is a phone call — both people present, live, blocked until it ends. Async is a WhatsApp message — you send it and move on, they reply whenever they're free.

**The tradeoff:** sync buys simplicity and instant consistency, but failures cascade — if the callee is slow, you're slow. Async buys resilience and independent scaling, but you now own eventual consistency, ordering, and duplicates.

## 2. Queues

Producer → queue → **one** consumer processes each message. Once consumed, it's gone (or moves to a DLQ on failure).

```
Producer ──> [ Queue ] ──> Consumer A
                  │
                  └─(if A fails)──> Consumer B picks it up
```

**Competing consumers pattern:** multiple workers pull from the same queue, each message goes to exactly one of them. This is how you horizontally scale processing — 10 workers draining one image-resize queue, each image handled once.

**Analogy:** a single ticket counter — one customer served by whichever teller is free next. Once served, that ticket's done. Adding more tellers (consumers) doesn't change the rule: each ticket still gets served exactly once.

**When queues fit:** task distribution, background jobs, rate-limiting outbound calls, anything where "exactly one worker handles this" is the requirement.

## 3. Pub/Sub

Producer → topic → **eligible** subscribers each get a copy. Broadcast, not point-to-point.

Two independent dials decide what happens to a published message: **Topology** (how many copies exist, and whether a copy gets split further) and **Selection** (which subscribers are even eligible to receive it in the first place). A third dial, **Durability**, isn't a pub/sub-specific thing — it's the same ephemeral/retention question from Section 6, just applied per-subscriber instead of per-topic.

| Dial | Question it answers | Options |
|---|---|---|
| **Topology** | How many copies, and does any copy get split further? | Plain fan-out (each subscriber gets the full copy) → Partitioned fan-out (subscribers in a group divide one copy among themselves) |
| **Selection** | Which subscribers are eligible for this message? | Everything (no selection) → Routing (broker decides using the topic/address, before delivery) → Content-based filtering (decided using the message body — broker-side or consumer-side) |
| **Durability** | What happens if a subscriber wasn't there? | See Section 6 — ephemeral / retention window / infinite retention / compaction, scoped to one subscriber |

**Example — one matching engine event, 5 subscribers, 5 different dial settings:**

```
                              Matching Engine
                                    |
                    emits: trade.executed.{assetClass}.{symbol}
                                    |
                                    v
                        [ topic: trade.executed ]
                                    |
        +---------------+---------------+---------------+---------------+
        |               |               |               |               |
     LEDGER         COMPLIANCE     FX SETTLEMENT     FRAUD ALERT      TICKER UI
   (full copy)      (full copy)     (full copy)      (full copy)      (full copy)
        |
        +-- worker A (symbols A-M)
        +-- worker B (symbols N-Z)
```

| Consumer | Topology | Selection | Durability |
|---|---|---|---|
| **Ledger** | Partitioned fan-out — 2 workers split the symbol range | Everything | Durable — worker resumes from last offset |
| **Compliance** | Plain fan-out | Everything | Durable — audit trail can't have gaps |
| **FX Settlement** | Plain fan-out | Routing — bound to pattern `*.forex.*`; broker never delivers equity trades here | Durable |
| **Fraud Alert** | Plain fan-out | Content-based filtering — reads `amount` field in the body, topic name is irrelevant | Durable |
| **Ticker UI** | Plain fan-out | Everything | Transient — closed tab misses it, next tick just overwrites |

**Analogy:** topology is BCC'ing an email to 3 departments (fan-out) vs. splitting one shared inbox across 3 reps by ticket number (partitioned fan-out). Selection is the difference between subscribing to a specific channel (routing — decided by address) and a keyword alert that scans every channel for a word (filtering — decided by content). Durability is whether a missed message is a voicemail waiting for you, or a phone that just rang once.

**Why this trips people up:** routing and broker-side filtering mean the message never reaches a non-matching consumer's queue at all — it's not "everyone gets it, consumer ignores it." That version only holds for **client-side filtering** — e.g. a Kafka consumer reading its whole assigned partition, then checking `amount > 100000` locally, after delivery. Same label, opposite bird's-eye picture, depending on whether the broker or the client is doing the deciding. Always check which layer actually enforces the rule before calling two systems "the same strategy."

## 4. Ordering

| Guarantee | What it means | Where you get it |
|---|---|---|
| **Global ordering** | Every consumer sees every event in the exact same single sequence | Single-partition topic, or a single queue — kills parallelism |
| **Partition-level ordering** | Order guaranteed only within a partition/shard, not across the whole topic | Kafka's default — pick a partition key so related events land together |
| **Causal ordering** | Events that depend on each other stay in order, unrelated events can interleave | Requires the app to carry causality info (version numbers, vector clocks) — brokers don't do this natively |
| **No ordering guarantee** | Whatever arrives first gets processed first, no promises | Naive fan-out or multi-broker setups |

**Analogy:** partition-level ordering is like separate WhatsApp group chats — within one group, messages appear in send order. But message 5 in Group A and message 3 in Group B have no defined relationship. If you need "all messages about Order #123 in order," you send them all to the same group (partition).

**Why this trips people up:** "Kafka guarantees ordering" is a half-truth. It guarantees ordering **per partition**. Global ordering across a topic means one partition — zero parallelism. Say that tradeoff explicitly, always.

## 5. Delivery Semantics

| Semantic | Guarantee | Failure mode | Typical mechanism |
|---|---|---|---|
| **At-most-once** | Delivered zero or one time | Can silently lose messages | Fire-and-forget, no ack/retry (`acks=0`, Redis Pub/Sub) |
| **At-least-once** | Delivered one or more times | Can duplicate | Ack after processing; crash before ack → redeliver (Kafka/RabbitMQ/SQS default) |
| **Exactly-once** | Delivered and processed exactly once | Hardest to achieve, usually scoped to "within one system" | Idempotent producer + transactions (Kafka), or app-level dedupe on top of at-least-once |

**Analogy:** at-most-once is a walkie-talkie message with no confirmation — unheard, it's gone. At-least-once is registered mail — they'll keep re-sending until you sign, but a glitch can get you two copies. Exactly-once is a legal document with a single serial number — the system actively refuses to file that serial number twice.

## 6. Persistence (Event Storage)

Where events actually live and for how long.

| Model | Behavior | Example |
|---|---|---|
| **Ephemeral (no persistence)** | Delivered live, gone forever after | Redis Pub/Sub |
| **Durable log, retention window** | Stored for N days/GB, then deleted | Kafka topic with a 7-day retention |
| **Durable log, infinite retention** | Never deleted, used as source of truth | Kafka topic for event sourcing / a dedicated event store |
| **Log compaction** | Keeps only the latest event per key, discards older ones for that key, retained forever | Kafka compacted topic for "current user profile" |

**Analogy:** a retention window is a security camera that overwrites footage after 7 days. Infinite retention is a permanent archive room. Compaction is a filing cabinet that keeps only your *most recent* form per person — old superseded forms shredded, one current copy always on file.

*Aside — stream processing:* some systems don't just store events, they run continuous computation directly on the stream as it arrives (e.g., "purchases in the last 5 minutes," updated live via windowed aggregation). Think live cashier-register total vs. a monthly bank reconciliation. Kafka Streams / Flink live here — worth its own note if you want to go deeper later, not core to this doc.

## 7. Brokers (Event Broker vs Event Bus)

| | Event Broker | Event Bus |
|---|---|---|
| What it is | The actual infrastructure that transports and (often) stores events | An architectural abstraction — a layer your code publishes to / subscribes from |
| Scope | Usually a distributed, networked system | Can be in-process (e.g. an `EventEmitter`) or distributed |
| Examples | Kafka, RabbitMQ, SQS, Redis Pub/Sub — these ARE brokers | A service's internal pub/sub layer, possibly backed by Kafka, possibly just in-memory |
| Concern | Durability, delivery guarantees, partitioning, scaling | Decoupling publishers from subscribers in your codebase, regardless of what's underneath |

**Analogy:** the broker is the actual postal service — trucks, sorting centers, delivery guarantees. The event bus is your office's internal mail-tray abstraction — "drop it in the outgoing tray, someone routes it" — which might be backed by the postal service, or might just be an intern walking it down the hall if everyone's in the same building.

**Why the distinction matters:** you can build an event bus abstraction (`publish(event)` / `subscribe(type, handler)`) that's backed by Kafka in production and an in-memory emitter in tests — the bus is the interface, the broker is one possible implementation behind it.

## 8. EDA — the conclusion

Everything above is the toolbox. **Event-Driven Architecture** is what you get when you actually build a system out of it: services stop calling each other directly and instead emit events, reacting to what happened rather than being told what to do.

```
[Order Service] ──emits──> OrderPlaced event
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                      ▼
[Inventory Svc]        [Notification Svc]      [Analytics Svc]
```

Two ways to coordinate this: **choreography** (services react independently, no central brain — loose but hard to debug) or **orchestration** (a central process explicitly sequences each step — tighter coupling, easier to trace). Two supporting patterns keep it honest in practice: the **outbox pattern** (write the DB change and the event in one transaction, so you never lose one or the other) and the **saga pattern** (a chain of local transactions with compensating rollbacks, since distributed transactions don't work cleanly across services).

None of this works without the pieces above: pub/sub for the fan-out, ordering so related events don't arrive scrambled, delivery semantics so you know what "reliable" actually means here, persistence so a late-joining consumer isn't lost, and a broker to carry it all. EDA is the destination — everything before it in this doc is what makes the destination reachable.

## 9. Decision table

| Need | Pick |
|---|---|
| Task distributed to exactly one worker | Queue (SQS/RabbitMQ) |
| Multiple independent services need to know something happened, with history/replay | Pub/Sub with persistence (Kafka) |
| Real-time ephemeral signal, don't care about missed messages | Broadcast pub/sub, no persistence (Redis Pub/Sub) |
| Need strict ordering for related events | Partition/shard by a key that groups related events together |
| Need to guarantee no message is silently dropped | At-least-once delivery + idempotent consumer |
| Services should react to business events without direct coupling | EDA, built on top of queue/pub-sub |

---
