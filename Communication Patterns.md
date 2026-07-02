# Communication Patterns in System Design

## Introduction

Covers the eight ways two parties exchange data in a system: REST, GraphQL, gRPC, WebSockets, SSE, WebRTC, Short Polling, Long Polling, and Webhooks. Some are protocols, some are architectural patterns built on top of HTTP — grouped here because they answer the same design question: how should this component talk to that one.

Structure:
1. **What it is** — analogy, technical definition, protocol used, use cases
2. **Pros vs Cons** — across scales, with system design considerations
3. **Failure Scenarios / Bottlenecks** — distinct from Table 2's cons
4. **Decision diagram** — quick reference for picking one

---

## Table 1: What It Is

| Topic | Analogy | What it is (technical points + protocol used) | Use cases |
|---|---|---|---|
| **REST** | Ordering off a fixed menu — you ask for "user #123" and get the whole dish, even if you only wanted one item. | • Resource-based, HTTP verbs (GET/POST/PUT/DELETE)<br>• Stateless — each request self-contained<br>• Fixed response shape per endpoint<br>• **Protocol:** HTTP/1.1 or HTTP/2, application layer over TCP | • Public-facing APIs<br>• CRUD (Create Read Update Delete) heavy applications<br>• Webhook receivers (a webhook payload lands on a REST endpoint)<br>• IoT (Internet of Things) device config/provisioning APIs |
| **GraphQL** | A buffet — you build your own plate in one trip instead of getting a preset dish. | • Single endpoint, client specifies exact fields via a query<br>• Server resolves each field through resolvers<br>• Schema is a strongly typed contract<br>• **Protocol:** Transport-agnostic, almost always HTTP (POST) | • Mobile clients needing nested data in one round trip<br>• BFF (Backend-For-Frontend) layer aggregating multiple microservices for one frontend team<br>• Rapidly-iterating product teams who don't want backend redeploys for every UI change |
| **gRPC** | Two people using a pre-agreed shorthand code — fast, but only understood by those in on it. | • Contract-first via `.proto` files<br>• Binary serialization (Protocol Buffers)<br>• Supports unary and streaming (client, server, bidirectional) calls<br>• **Protocol:** HTTP/2, binary payload over TCP | • Internal service mesh, service-to-service calls<br>• High-frequency browser dashboards via grpc-web (trading UIs)<br>• Bandwidth-constrained IoT devices where every byte matters |
| **WebSockets** | A phone call — once connected, either person can speak anytime, no redialing. | • Persistent, full-duplex connection<br>• Starts as HTTP, upgrades via handshake (101 Switching Protocols)<br>• Either side can push at any time<br>• **Protocol:** TCP, `ws://` or `wss://` after HTTP upgrade | • Chat applications, multiplayer games<br>• Real-time order-book streaming on trading platforms<br>• Collaborative editing / shared whiteboards (live cursors) |
| **SSE (Server-Sent Events)** | A radio broadcast — the server keeps talking, the client just listens. | • One-way, server → client stream<br>• Plain HTTP, `text/event-stream` content type<br>• Browser auto-reconnect is part of the spec<br>• **Protocol:** HTTP/1.1 or HTTP/2, persistent connection over TCP | • Live notifications, stock tickers<br>• Token-by-token streaming for LLM (Large Language Model) chat responses — this is how ChatGPT-style UIs stream text<br>• CI/CD (Continuous Integration/Continuous Deployment) build log tailing |
| **WebRTC (Web Real-Time Communication)** | Two people who want to talk directly pick up private walkie-talkies — they first call a mutual friend to swap frequencies, then talk directly without the friend relaying anything. | • Peer-to-peer media/data, browser-native<br>• ICE (Interactive Connectivity Establishment) handles NAT (Network Address Translation) traversal using STUN (Session Traversal Utilities for NAT) and TURN (Traversal Using Relays around NAT) servers<br>• Signaling (SDP offer/answer exchange) is NOT defined by the spec — you build it yourself, typically over WebSocket<br>• **Protocol:** UDP primarily, DTLS (Datagram Transport Layer Security) handshake, SRTP (Secure Real-time Transport Protocol) for encrypted media, SCTP (Stream Control Transmission Protocol) over DTLS for the data channel | • Video/voice calls (Zoom/Meet-style)<br>• Peer-to-peer file sharing without a relay server<br>• Screen sharing<br>• Low-latency multiplayer game state sync between peers |
| **Short Polling** | Calling someone every 5 minutes to ask "anything new?" — mostly wasted calls. | • Client asks at fixed intervals<br>• Stateless, plain HTTP request/response repeated<br>• **Protocol:** HTTP/1.1 or HTTP/2 | • Simple status checks<br>• Fallback for clients behind corporate proxies/firewalls that block WebSockets entirely<br>• Battery-conscious mobile background sync |
| **Long Polling** | Calling and asking them to stay on the line until they actually have something to say, then hanging up and redialing. | • Client asks, server holds the request open until data is ready or a timeout hits<br>• Client immediately re-requests after each response<br>• **Protocol:** HTTP/1.1 or HTTP/2, held-open request/response | • Older chat apps before WebSocket adoption<br>• Automatic transport fallback inside Socket.IO when WebSocket upgrade fails<br>• Delivery-status checks in older telecom/SMS systems |
| **Webhooks** | "Call me when it happens" — you leave your number, they call you back instead of you calling to check. | • Reverse of polling — receiver registers a URL, sender POSTs to it on event<br>• Server-to-server, event-triggered, asynchronous<br>• **Protocol:** HTTP POST, typically over HTTPS | • Payment confirmation (Stripe "charge succeeded")<br>• CI triggers (GitHub push events)<br>• Syncing state between two SaaS products (CRM ↔ billing tool) without either polling the other |

---

## Table 2: Pros vs Cons (All Scales) + System Design Considerations

| Topic | Pros | Cons | System design considerations |
|---|---|---|---|
| **REST** | • Stateless → scales horizontally with zero coordination<br>• CDN (Content Delivery Network) / edge caching works out of the box — advantage over GraphQL, which can't cache this easily since every query differs<br>• Simple to build/debug | • Over-fetching/under-fetching — fixed response shape<br>• N+1 problem for nested/related resources<br>• Chatty at scale — mobile clients making 10+ calls per screen | • Versioning strategy up front (URI vs header versioning)<br>• Pagination design for large collections<br>• Cache-Control/ETag strategy for CDN offload<br>• Stateless auth (JWT (JSON Web Token)) vs session store tradeoff |
| **GraphQL** | • One round trip for nested data, exact shape the client wants — advantage over REST, no over/under-fetching<br>• Schema acts as an enforced frontend-backend contract | • Harder to cache (single endpoint, varying queries)<br>• Client-controlled query complexity = unpredictable server load<br>• Steeper learning curve, resolver complexity | • Query depth/complexity limiting is mandatory from day one, not an afterthought<br>• Persisted queries needed for both caching and security<br>• Resolver batching (DataLoader pattern) must be architected in, not bolted on<br>• Schema evolves additively — no REST-style versioning |
| **gRPC** | • HTTP/2 multiplexing — many calls over one connection, low latency/bandwidth — advantage over REST for internal high-throughput calls<br>• Compile-time type safety, smaller payloads (binary vs JSON) | • Not browser-native, needs grpc-web + proxy<br>• Harder to debug (binary, not human-readable)<br>• L4 (Layer 4) load balancers don't handle multiplexed streams well | • Needs a service mesh (Envoy/Istio) for proper L7 (Layer 7) load balancing<br>• Proto contract governance across teams becomes a real process, not just a file<br>• Deadline propagation and retry budgets must be designed to avoid cascading failures |
| **WebSockets** | • True bidirectional, lowest latency for two-way real-time — advantage over SSE (one-way) and over polling (no wasted requests) | • Stateful — server holds per-connection state, harder to load balance (needs sticky sessions)<br>• Each open connection costs memory/file descriptors | • Horizontal scaling needs a shared pub/sub backbone (Redis/Kafka) so any node can reach any client<br>• Heartbeat/ping-pong required to detect dead connections<br>• Decide delivery guarantee — at-least-once needs client-side dedup on reconnect |
| **SSE** | • Simpler than WebSockets — plain HTTP, browser auto-reconnect built in — advantage over WebSockets when you only need one-way push<br>• Works through normal HTTP infra/proxies | • One-way only — need a separate channel for client→server<br>• Browser caps ~6 connections/domain on HTTP/1.1 | • Prefer HTTP/2 deployment to avoid the browser connection cap<br>• Use `Last-Event-ID` header to resume from the right point after reconnect<br>• Decide if push volume justifies a dedicated fan-out layer |
| **WebRTC** | • True peer-to-peer for media — no server relay needed once connected, advantage over WebSockets which always routes through a server<br>• Mandatory encryption (DTLS/SRTP) built into the protocol, not optional | • Complex signaling setup required — WebRTC doesn't define signaling, you build it separately<br>• NAT traversal isn't always successful — may need TURN relay fallback, which costs bandwidth/money<br>• Mesh topology doesn't scale past a small group without a dedicated media server | • Need a signaling server (commonly WebSocket-based) purely to exchange SDP (Session Description Protocol) offers/answers and ICE candidates<br>• Choose mesh vs SFU (Selective Forwarding Unit) vs MCU (Multipoint Control Unit) architecture based on expected participant count<br>• Budget TURN server cost — a real-world chunk of users (commonly cited ~10-20%) sit behind symmetric NATs that force relay<br>• Consider geographic distribution of media/TURN servers for latency |
| **Short Polling** | • Dead simple, works everywhere, no special infra — advantage over WS (WebSockets)/SSE/long polling in restrictive network environments | • Wastes most requests (mostly "nothing new")<br>• Added latency ≈ half the poll interval | • Poll interval is a direct staleness-vs-load tradeoff, tune deliberately<br>• Client-side backoff (widen interval when idle) reduces wasted load<br>• Backend stays stateless and trivially scales horizontally |
| **Long Polling** | • Fewer wasted requests and lower latency than short polling — advantage over short polling, server responds only when data's ready | • Ties up a server thread/connection per waiting client — same resource cost as WebSockets, plus reconnect overhead on top | • Server needs a budgeted thread/connection pool for held requests<br>• Usually best used as an automatic fallback tier (e.g. Socket.IO), not the primary design<br>• Timeout handling needed so proxies/load balancers don't kill held connections prematurely |
| **Webhooks** | • Zero polling waste on both sides — near-instant, event-triggered — advantage over all polling variants<br>• Sender holds no per-consumer connection state — advantage over WebSockets for the sender side | • Receiver must expose a public, reachable endpoint<br>• No built-in delivery guarantee — depends entirely on sender's retry policy | • Retry policy and idempotency keys on the receiver must be designed from day one<br>• Signature verification (HMAC (Hash-based Message Authentication Code)) is non-negotiable for security<br>• A queue/buffer on the receiver side absorbs bursts from the sender<br>• Provide a replay mechanism for consumers to recover missed events |

---

## Table 3: Failure Scenarios / Bottlenecks


| Topic | Failure scenarios / Bottlenecks |
|---|---|
| **REST** | Retried POSTs can create duplicate resources without idempotency keys (same problem your Outbox pattern solves at the DB layer) |
| **GraphQL** | A single deep/nested query can trigger an N+1 resolver storm — thousands of DB calls from one client request, unless batched |
| **gRPC** | One dead multiplexed connection kills every stream riding on it at once — bigger blast radius than REST's independent connections; proxies not speaking HTTP/2 correctly can silently break calls |
| **WebSockets** | Server crash drops all its connections at once, forcing mass reconnects; idle-timeout proxies/load balancers can silently kill "healthy" connections |
| **SSE** | HTTP/1.1's 6-connections-per-domain cap breaks multi-tab usage of the same app (mitigated by HTTP/2) |
| **WebRTC** | ICE negotiation/gathering adds perceptible call-setup latency (often 1-3+ seconds); signaling server outage blocks new connections or renegotiation even if already-connected peers are unaffected; firewalls blocking all UDP force TURN-over-TCP, adding further latency; carrier-grade NAT can cause total connection failure even with STUN |
| **Short Polling** | At scale, request volume itself becomes the bottleneck — e.g. 100K clients polling every 5s = 20K req/sec, most returning nothing |
| **Long Polling** | The connect → wait → respond → reconnect cycle can consume more server resources in aggregate than one stable WebSocket connection would |
| **Webhooks** | Anyone who discovers your webhook URL can POST forged events unless signatures are verified; debugging is harder than polling — no "ask again," you wait or check the sender's delivery logs |

---

## Decision Diagram
This diagram covers common cases.

```
Need exact response shape, nested data in one call? ── yes ──► GraphQL
        │ no
        ▼
Internal service-to-service, high throughput? ── yes ──► gRPC
        │ no
        ▼
Peer-to-peer media/data, no server relay wanted? ── yes ──► WebRTC
        │ no
        ▼
Is this a server receiving an event from another
service/third party (not a client-facing need)? ── yes ──► Webhooks
        │ no
        ▼
Does the client need live updates from the server at all?
        │ no ──► REST (default)
        │ yes
        ▼
   Does the client also need to push data continuously (bidirectional)?
        │ yes ──► WebSockets
        │ no
        ▼
   Can the connection stay open (no restrictive proxy/firewall killing it)?
        │ yes ──► SSE
        │ no
        ▼
   Can it stay open briefly, just long enough to wait for one event?
        │ yes ──► Long Polling
        │ no  ──► Short Polling
```
