# API Architecture Notes

## Layers
- **Three different things:** protocol (HTTP, WebSocket, WebRTC), style (REST, gRPC, GraphQL, SOAP), pattern (request-response, webhook, SSE, streaming). A system combines one of each.
- **HTTP versions:** each one fixes the main drawback of the one before.

| Version | Main feature | Drawback (fixed in next version) |
|---|---|---|
| HTTP/1.0 | Simple text request-response, one TCP connection per request | Connection setup cost on every request (fixed by 1.1 keep-alive) |
| HTTP/1.1 | Persistent connections, chunked transfer, Host header (virtual hosting) | Head-of-line blocking (one request at a time per connection), repeated verbose headers; browsers work around it with about 6 connections (fixed by 2) |
| HTTP/2 | Binary framing, multiplexed streams on one connection, header compression (gRPC needs it) | TCP-level head-of-line blocking (one lost packet stalls all streams), slow TCP + TLS handshake (fixed by 3) |
| HTTP/3 | Runs on QUIC/UDP, independent streams, built-in TLS 1.3, faster handshake, connection migration | No later version; UDP can be blocked by firewalls, so clients fall back to HTTP/2 |

## REST vs RPC
- **Abstraction:** resource vs operation.
- **Interface:** uniform resource interface vs explicit procedure interface.
- **HTTP semantics:** REST uses them (verbs, status codes, caching). RPC mostly uses HTTP just to invoke operations.
- **Real distinction:** action-oriented (`POST /createOrder`, params usually in the body) vs resource-oriented (`/orders/42`; GET is safe and cacheable, PUT/DELETE are idempotent).
- **Richardson maturity:** L0 single endpoint, L1 resources, L2 verbs + status codes, L3 HATEOAS. Most "REST" APIs stop at L2.
- **Single execute endpoint routed by payload** is L0 / RPC-style: fine for internal transactional flows, but no HTTP caching, and routing/metrics need payload inspection.

## gRPC
- HTTP/2 + Protobuf (compact binary serialization, not compression), contract-first via `.proto` with generated stubs.
- **Modes:** unary, server streaming, client streaming, bidirectional.
- **Built in:** deadlines, metadata, interceptors, own status codes.
- **Evolution:** never reuse or renumber field tags; adding fields is backward compatible.
- Browsers can't call it directly (use gRPC-Web or Connect). **Best for:** internal service-to-service. **Weak for:** public APIs, debugging.

## GraphQL
- The client defines the response shape (fields), not the request payload. Typically a single endpoint with a typed schema.
- **Operations:** query, mutation, subscription (usually over WebSocket).
- Fixes over/under-fetching; good as a BFF over many services.
- **Gotchas:** N+1 (use DataLoader), hard HTTP caching, needs depth/cost limits, errors arrive in an `errors` array with HTTP 200.

## SOAP
- XML Envelope (Header + Body), WSDL contract, standardized Faults.
- **Extensions:** WS-Security (message-level signing/encryption), WS-ReliableMessaging, WS-AtomicTransaction.
- Transport-agnostic (HTTP, JMS, MQ, SMTP). Message-level security survives queues and intermediaries, while TLS protects only one hop. Fits banking; downside is verbosity.

## WebSocket
- Starts as an HTTP upgrade (`101 Switching Protocols`), then a persistent full-duplex connection over TCP (`wss://`).
- Heartbeat with ping/pong; reconnect with backoff. No message semantics, so design your own protocol (or use STOMP / Socket.IO).
- **Scaling:** stateful connections need sticky sessions or an L4 load balancer, plus a pub/sub backplane (Redis/Kafka) for fan-out.
- **Use:** chat, live tickers, collaboration, games.

## SSE
- One-way server to client over plain HTTP (`text/event-stream`), text only.
- Auto-reconnect with resume via `Last-Event-ID`. HTTP/1.1 caps about 6 connections per domain (HTTP/2 fixes it).
- **Use:** notifications, live feeds, LLM token streaming. Simpler than WebSocket when only the server pushes.

## Webhooks
- Server-to-server HTTP callback on an event (the reverse of polling), e.g. payment-status callbacks.
- **Receiver:** verify the HMAC signature, return 2xx fast and process async, be idempotent (delivery is at-least-once; duplicates and reordering happen).
- **Sender:** retries with exponential backoff and a dead-letter queue.

## WebRTC
- Media is peer-to-peer (SRTP over UDP, DTLS mandatory) but not always direct. Data channels (SCTP over DTLS) carry game or app data too.
- **Signaling is not part of WebRTC:** you build it (usually WebSocket/HTTP) to exchange the SDP offer/answer and ICE candidates.
- **ICE order:** direct, then STUN (discovers the public address), then TURN (relay when NAT blocks; costly; can fall back to TCP/TLS).
- **Scaling:** a mesh breaks down beyond about 4-6 peers; use an SFU (forwarding) or MCU (mixing).

## Interaction patterns

| Pattern | Direction | Connection | Use |
|---|---|---|---|
| Short polling | Client pulls | New request each time | Simple, wasteful |
| Long polling | Client pulls | Held open until data | Fallback for push |
| SSE | Server to client | One long HTTP response | Feeds, notifications |
| WebSocket | Both ways | Persistent TCP | Chat, trading, games |
| Webhook | Server to server | Per-event HTTP call | Event notification |
| Async request-response | Client to server | `202 Accepted` + status URL or callback | Long-running jobs |

- **Streaming:** gRPC streams at the API level; Kafka/MQ for event streams behind the API (a messaging backbone, not an API style).

## Choosing

| Need | Pick |
|---|---|
| Public/partner CRUD APIs | REST |
| Internal service-to-service, low latency | gRPC |
| Many clients with varied data needs | GraphQL |
| Legacy/B2B/regulated integration | SOAP |
| Bidirectional real-time | WebSocket |
| Server push only | SSE |
| Event notification between systems | Webhook |
| Peer media/data | WebRTC |

## Cross-cutting
- **Reliability:** idempotency keys on POST (essential for payments), timeouts, retries with backoff.
- **Evolution:** versioning (URI/header for REST, field-tag compatibility for Protobuf), cursor pagination over offset.
- **Security and ops:** OAuth2/JWT for clients, mTLS for service-to-service, rate limiting, API gateway, consistent error model, correlation/trace IDs.
