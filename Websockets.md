# WebSockets

## 1. Core Protocol (RFC 6455)

RFC 6455 defines only: handshake, frame format, opcodes, masking, fragmentation, close frames, ping/pong.

**Not defined by the protocol** (all app/library responsibility): auth, reconnection, heartbeat policy, rooms/pub-sub, message format, resume-after-disconnect.

- No routes after the handshake. Routing happens via message content (`{"type": "..."}`), not URLs.
- Socket.IO namespaces (`/chat`, `/admin`) are a **library abstraction**, not part of the WebSocket protocol.
- **Browser gap**: JS in the browser cannot send a Ping frame — only the server can send Ping. The browser auto-replies Pong. Client-initiated liveness checks must be an app-level heartbeat *message*, not a protocol Ping.

Why care? If you assume the spec gives you rooms, reconnection, or a client-side heartbeat for free, you'll ship a WS server that silently has none of that — and find out in production when a client goes half-open (§3) with no app-level heartbeat to catch it.

Analogy: a WebSocket connection is a phone call, not a series of letters (HTTP requests). Once connected, there's no "envelope routing" — just a live line where the app decides what each sentence means.

---

## 2. Architecture Patterns

- One process = one HTTP server = one port. Express/Socket.IO don't fight over the port — the HTTP server owns the socket and routes based on whether the request has `Upgrade: websocket` headers.
- **Standalone WS-only servers are real and common** (uWebSockets, Gorilla WebSocket, Phoenix Channels, game servers).
  - Use cases: pure real-time products, performance isolation (HTTP is bursty/stateless, WS is long-lived/memory-heavy — mixing starves one or the other), microservice split via a broker instead of a shared DB.
- Progression pattern: `Express+Socket.IO → separate API+WS servers on shared DB → API/WS servers + broker + business services + DB`.
- WS server can act as **connection manager only** — auth, `userId → socket` map, subscribe/broadcast — with persistence in a separate service reached via Kafka/Redis.

---

## 3. Connection Liveness — the Half-Open Problem

**Half-open connection**: TCP doesn't know a peer is gone just because packets stop.
Analogy: a phone call where the other person's battery dies mid-sentence — no "call ended" tone, just silence.

Layered liveness detection, each layer faster than the one below:

```
App heartbeat (message-level)
   ↓
WS Ping/Pong (RFC-defined, timing NOT specified)
   ↓
TCP Keepalive (OS default ~2hr — way too slow to rely on)
   ↓
TCP retransmission timeout (minutes/hours, OS-dependent)
```

| Failure mode | How server learns |
|---|---|
| Graceful close (FIN) | Near-instantly |
| Killed process (RST) | Near-instantly |
| Silent death (lid closed / WiFi off / cable pulled) | Only on next write attempt (EPIPE/ECONNRESET) or heartbeat timeout |

- TCP keepalive proves the **host** is alive, not the **application** — a frozen/deadlocked client still answers TCP-layer probes but won't Pong at the app layer.
- Broken pipe errors only surface **on write**, not on the silent disappearance itself.

Why care? Without an app-level heartbeat, presence systems, chat, games, and stock tickers all show stale users as still online — the server has no way to know they're gone until it tries to write and fails, or a heartbeat times out. This is the single most common cause of "ghost users" bugs in production.

---

## 4. Reconnection & Recovery

- No "resume" concept in TCP or WebSocket — every reconnect is a brand-new handshake.
- IP change (switching networks) breaks the old TCP connection — it's bound to a 4-tuple (see §8).
- Server doesn't remember old sockets. Reconnection maps back to the same logical user only via **re-authentication** (JWT/session), never socket-level continuity.
- Missed messages during disconnect are **not buffered by the protocol** — recovery is app-specific:
  - Chat: `GET /messages?after=lastSeenId`
  - Prices/games: just resubscribe, take latest state
- Auto-reconnect (exponential backoff) is a **library** feature (Socket.IO etc.), not native WebSocket behavior.

Why care? Every client eventually reconnects — flaky WiFi, phone sleep, laptop lid. If recovery isn't designed deliberately, the default behavior is silent data loss: messages sent while disconnected simply vanish, because the protocol keeps no memory of what a client missed.

---

## 5. WebSocket vs SSE vs Long Polling

Three independent concepts people conflate: **transport** (WS / long-polling / SSE) and **liveness** (heartbeat) — heartbeat is orthogonal to transport choice.

| | WebSocket | SSE | Long Polling |
|---|---|---|---|
| Direction | Bidirectional | Server → client only | Bidirectional (via repeated requests) |
| Data type | Binary + text | Text only | Whatever HTTP carries |
| Multiplexing | One pipe, many message types | One stream per subscription | N/A — fallback transport |
| Complexity | Higher | Lower, simpler to debug | Simplest, most overhead |


### Misconceptions corrected 
- Assumed WS servers are always bundled with Express → **wrong**, standalone WS-only servers are standard.
- Assumed "internet off" ≈ "Ctrl+C / tab close" → **wrong**, one is silent (no FIN/RST), the other is a clean close the server sees immediately.
- Assumed stock data *needs* WebSockets → partially wrong; SSE+HTTP suffices for pure viewers, WS is only a hard requirement for order-book depth / same-pipe trade ordering.
- Assumed Socket.IO's long-polling fallback was about heartbeat → wrong, transport and liveness are unrelated concerns.

---

## 6. WebSocket / SSE over HTTP/1.1, HTTP/2, HTTP/3
 
### What the specs define
 
| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| **WebSocket** | RFC 6455 — `Upgrade: websocket` | RFC 8441 — Extended CONNECT (`:protocol = websocket`) on one stream | RFC 9220 — same idea over QUIC |
| **SSE** | No RFC — long-lived `GET` with `text/event-stream` | No new spec — rides HTTP/2 | No new spec — rides HTTP/3 |
 
---
 
### What is actually deployed
 
| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| **WebSocket** | Universal. Safe production default. | **Real & automatic** in Chrome 95+ / Firefox (no flag). Server/proxy support is patchy — this gap breaks production. | **No production browser support** as of early 2026. |
| **SSE** | Universal. ~6 connections/domain cap. | Works automatically. Cap gone (~100 streams/connection). | Works automatically. No TCP HOL blocking. |

 
---
 
### Bottom line
 
- **HTTP/1.1 WS** — safe default.
- **HTTP/2 WS** — browsers do it automatically; servers/proxies lag.
- **HTTP/3 WS** — spec-only, no browser ships it.
- **HTTP/3 alternative** — **WebTransport** (QUIC-native, multiple streams + unreliable datagrams).

---

## 7. TCP Connection Identity & Theoretical Limits

Connections are uniquely identified by the **5-tuple**: `(protocol, src IP, src port, dst IP, dst port)`.

- `protocol` is why a TCP socket and a UDP socket can both bind the exact same IP:port (e.g. `:443` for WebSocket-over-TCP and QUIC-over-UDP simultaneously) — the kernel's connection table distinguishes by protocol too.
- Within one protocol (TCP), the **4-tuple** (src IP, src port, dst IP, dst port) is what actually differentiates connections.

### The "65536 connection limit" misconception

That number applies **only** to one machine making outbound connections to one fixed (dst IP, dst port) — i.e., a client's ephemeral port budget. It does **not** cap how many connections a server can *accept*.

Analogy: a server's listening port is a company's one published phone number. The company isn't limited to 65,536 simultaneous calls, because each call is identified by *your number + their number* — and "their number" differs every time.

### What actually caps a server's connection count

1. **File descriptors** — every socket is a fd (`ulimit -n`, `fs.file-max`/`fs.nr_open`).
2. **Memory** — kernel socket buffers (`net.core.rmem_max`/`wmem_max`) + app-level per-connection state. Usually the real ceiling: `total ≈ RAM / per-connection footprint`.
3. **CPU** — event loop (epoll/kqueue) servicing every connection's I/O readiness. C10K → C10M evolution.
4. **Bandwidth** — irrelevant to *holding* idle connections open, relevant only to active throughput.

### Real-world benchmarks

| Setup | Result |
|---|---|
| Node.js/Go, event-driven, tuned kernel, single node | ~240,000 concurrent, sub-50ms latency |
| Elixir/Phoenix, 40-core/128GB box | 2 million concurrent WS clients, ceiling was `ulimit`, not hardware |
| C++ (LAppS), single instance | ~2.46 million, capped by a 2GB-per-VM memory limit |
| Same, 5 instances | 12.3 million total |
| oatpp benchmark, 16-core/104GB boxes | 5 million |

Pattern in every case: **the ceiling was a configured limit (ulimit, VM memory cap, kernel tuning), never the protocol itself.**


---

## 8. Load Balancer → Backend: the *real* port-exhaustion case

The one place the ~64K number is genuinely real: when the **LB terminates the client connection and opens a new outbound TCP connection to a backend** (L7 proxy: Nginx, HAProxy, some ALB configs).

- LB's client-facing side: same as §7, effectively unbounded (resource-limited only).
- **LB → backend side**: LB is now acting as a *client* with (usually) one source IP, connecting to one fixed (backend dst IP, dst port). Capped by the ephemeral port range — typically **~28,000–60,000 usable ports** (`net.ipv4.ip_local_port_range`), not the full 65,536.

For a fixed `(LB src IP, dst IP, dst port)`, only `src port` varies → **~64K ceiling is per (LB source IP, backend instance) pair**, not a global pool shared across all instances.

`Total capacity ≈ (# LB source IPs) × (# backend instances/ports) × ~60K`

Two refinements:
- Usable range is **less than 65,536** — ports 0–1023 reserved, real ephemeral range OS-configured (Linux default ~32768–60999, ~28K usable).
- **TIME_WAIT** eats into it: closed connections hold their tuple in TIME_WAIT (~60s on Linux) before reuse. Barely matters for long-lived WS (they don't churn), but reconnect storms or short-lived backend connections can exhaust the pool before hitting the theoretical count. Knob: `net.ipv4.tcp_tw_reuse`.

Why care? LB silently starts rejecting new backend connections once traffic crosses a per-destination port ceiling nobody budgeted for. It shows up as intermittent connection failures under load, not a clean error.

### Fixes

- **More backend instances/ports** — each is a separate dst tuple, multiplies the ceiling directly.
- **SNAT pool** (multiple LB source IPs) — also multiplies the ceiling.
- **L4/DSR passthrough** — LB never opens its own outbound connection per client, so the SNAT exhaustion problem doesn't exist at all. This is how the largest single-node benchmarks in §7 are typically rigged, and how many production L4 LBs (IPVS, ECMP-based) work.
- **TPROXY** — LB preserves original client IP/port when forwarding, backend sees real tuples directly.

---

## 9. Scaling WebSockets — Connection Count

- **Horizontal scaling** — more WS instances behind the LB, each with its own ~60K-per-LB-source-IP budget (§8). Main lever.
- **`SO_REUSEPORT`** — multiple processes/threads on the *same box* bind the same port; kernel load-balances `accept()` across them. Lets you use all cores without a userspace LB in front.
- **Kernel tuning** — raise `ulimit -n`/`fs.file-max`, widen `ip_local_port_range`, enable `tcp_tw_reuse`, right-size socket buffers per connection.
- **L4 passthrough / DSR** — avoids LB-side SNAT exhaustion entirely.
- **Runtime choice** — Erlang/Elixir (BEAM processes, ~2KB each) and Go (goroutines) beat thread-per-connection models on connections-per-GB. This is *why* Phoenix hit 2M on one box.
- **Sticky sessions** — WS is stateful; LB must route reconnects to the *same* instance (IP hash / cookie / consistent hashing). Required infra, not a capacity lever.
- **Geo-distributed edge gateways** — regional PoPs each get their own port/FD budget and cut latency.

---

## 10. Scaling WebSockets — Coordinating Across Instances

Once clients are spread across N instances, instance A doesn't hold the socket for a client on instance B — it physically can't push to it directly. This is the **fan-out problem**.

Analogy: each WS instance is a radio tower broadcasting to whoever's tuned in. To reach a listener on a *different* tower, you need a shared signal source (pub/sub) every tower listens to and rebroadcasts locally.

**Mechanism**: instance subscribes to broker → business logic publishes to a topic (`room:42`, `user:918`) → every instance holding a relevant connection receives it and writes to its local sockets.

### Broker comparison

| | Redis Pub/Sub | Redis Streams | Kafka | NATS (+ JetStream) |
|---|---|---|---|---|
| Delivery guarantee | Fire-and-forget, **lost if no subscriber listening** | At-least-once, replayable | At-least-once, replayable, durable log | Core = fire-and-forget; JetStream = at-least-once, replayable |
| Ordering | Best-effort | Per-stream ordered | Guaranteed within a partition | Per-subject ordered (JetStream) |
| Latency | Lowest | Low | Higher (durability tradeoff) | Very low |
| Ops weight | Light | Light | Heavy (ZooKeeper/KRaft, partitions) | Light-to-moderate |
| Best fit | Ephemeral broadcast (typing indicators, presence) | Need replay, keep it simple | Durability/audit trail, buffer missed messages | High-throughput low-latency fanout, less ops than Kafka |

**Ties back to §4**: Kafka's replay-from-offset is a common *implementation* of the app-specific "recover missed messages on reconnect" problem. Redis pub/sub can't do this — undelivered messages are gone forever.

### Supporting pieces

- **Presence/routing table** (Redis KV: `userId → instanceId`) — needed for targeted delivery instead of broadcasting everything to everyone. Needs TTL/heartbeat so a crashed instance's stale mapping expires.
- **Topic granularity** — instance-scoped (`instance:7`) for direct routing vs room-scoped (`room:42`) for group broadcast. Wrong granularity → message storms or missed deliveries.
- **Backpressure** — a slow client shouldn't block the broker or other clients. Fix: bounded per-connection outbound queue; drop or disconnect the slow client.
- **Partition key (Kafka)** — partition by `roomId`/`userId` to guarantee per-entity ordering while parallelizing.
- **Framework shortcuts** — Socket.IO's `redis-adapter` pre-builds this whole pattern; Phoenix/Elixir has it natively via BEAM's distributed process registry, no external broker needed.

