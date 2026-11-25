# API Architecture Notes

## Layers (mental model)
Every API is a stack of independent choices. To understand any combination, fill in these slots:

| Layer | Question | Options |
|---|---|---|
| Transport | How do bytes move? | TCP, QUIC/UDP, stdio pipes, Unix socket, HTTP, WebSocket, IBM MQ, Kafka |
| Framing / correlation | Where does a message end, and which reply matches which request? | HTTP (built in), newline or `Content-Length` (stdio), MQ message boundary + `CorrelId`, JSON-RPC `id` |
| Encoding | How are bytes structured? | JSON, XML, Protobuf |
| Message protocol (style) | How are method, params, result and error expressed? | REST, JSON-RPC, SOAP, gRPC, GraphQL |
| Interaction pattern | Who talks when? | Request-response, notification, streaming, pub/sub, callback (webhook) |
| Contract | How is it described? | OpenAPI, WSDL, `.proto`, GraphQL SDL, JSON Schema (MCP tools) |

- **Tied vs free:**
  - **REST** is tied to HTTP (uses its verbs, status codes, caching).
  - **gRPC** is specified over HTTP/2 only and defaults to Protobuf.
  - **GraphQL** is transport-agnostic in theory, HTTP in practice.
  - **JSON-RPC and SOAP** are fully transport-agnostic.
- HTTP is a protocol that REST, gRPC and GraphQL use *as a transport*. WebRTC is a stack (ICE, DTLS, SRTP, SCTP), not one protocol.

### Common stacks

| System | Message protocol | Encoding | Transport | Pattern |
|---|---|---|---|---|
| Spring REST | REST | JSON | HTTP over TCP + TLS | Request-response |
| MCP (local) | JSON-RPC 2.0 | JSON | stdio | Request-response, notifications |
| MCP (remote) | JSON-RPC 2.0 | JSON | Streamable HTTP (+ SSE) | Plus streaming, server-to-client requests |
| SOAP over MQ | SOAP | XML | IBM MQ / JMS | Async request-reply |
| LSP | JSON-RPC 2.0 | JSON | stdio (`Content-Length` framing) | Request-response, notifications |
| Ethereum node | JSON-RPC 2.0 | JSON | HTTP, WebSocket, or IPC | Request-response, subscriptions |
| gRPC | gRPC | Protobuf | HTTP/2 | Unary + 3 streaming modes |

## HTTP versions
Each one fixes the main drawback of the one before.

| Version | Main feature | Drawback (fixed in next version) |
|---|---|---|
| HTTP/1.0 | Simple text request-response, one TCP connection per request | Connection setup cost on every request (fixed by 1.1 keep-alive) |
| HTTP/1.1 | Persistent connections, chunked transfer, Host header (virtual hosting) | Head-of-line blocking (one request at a time per connection), repeated verbose headers; browsers work around it with about 6 connections per host (fixed by 2) |
| HTTP/2 | Binary framing, multiplexed streams on one connection, header compression (HPACK); gRPC needs it | TCP-level head-of-line blocking (one lost packet stalls all streams), separate TCP + TLS handshakes (fixed by 3) |
| HTTP/3 | Runs on QUIC/UDP, independent streams, built-in TLS 1.3, 1-RTT handshake (0-RTT on resumption), connection migration | UDP can be blocked by firewalls, so clients fall back to HTTP/2 |

## REST vs RPC
- **Abstraction:** resource vs operation.
- **Interface:** uniform resource interface vs explicit procedure interface.
- **HTTP semantics:** REST uses them (verbs, status codes, caching). RPC mostly uses HTTP just to invoke operations, or doesn't use HTTP at all.
- **Real distinction:** action-oriented (`POST /createOrder`, params usually in the body) vs resource-oriented (`/orders/42`).
- **Method properties:** GET is safe, idempotent and cacheable. PUT and DELETE are idempotent. POST and PATCH are not guaranteed idempotent.
- **Richardson maturity:** L0 single endpoint, L1 resources, L2 verbs + status codes, L3 HATEOAS. Fielding's definition of REST requires L3, so most "REST" APIs (L2) are really HTTP APIs.
- **Single execute endpoint routed by payload** is L0 / RPC-style: fine for internal transactional flows, but no HTTP caching, and routing/metrics need payload inspection.

## JSON-RPC 2.0
- **Request:** `{"jsonrpc":"2.0","method":"…","params":…,"id":…}`. `params` is an array or an object.
- **Notification:** a request with no `id`. The server sends no response.
- **Batch:** an array of requests. Responses come back as an array, in any order, with notifications omitted.
- **Response:** exactly one of `result` or `error`, with the same `id`. An error has `code`, `message` and optional `data`.
- **Reserved error codes:** -32700 parse error, -32600 invalid request, -32601 method not found, -32602 invalid params, -32603 internal error, -32000 to -32099 server-defined.
- **No transport or framing defined** by the spec. Correlation is by `id`. Errors travel in the body, so over HTTP the status is usually 200.
- **Used by:** MCP, LSP, Ethereum nodes.

## MCP (Model Context Protocol)
- JSON-RPC 2.0 messages (UTF-8) between a client (inside a host app) and a server.
- **Stateful:** starts with an `initialize` request (protocol version + capability negotiation), then a `notifications/initialized` notification.
- **Bidirectional:** servers can send requests to clients (sampling, elicitation, roots), not only respond.
- **Server features:** tools (`tools/list`, `tools/call`), resources, prompts.
- **Transports** (latest spec revision I know of is 2025-11-25; check for newer):
  - **stdio:** client launches the server as a subprocess. Newline-delimited JSON-RPC over stdin/stdout (no embedded newlines). Only MCP messages on stdout, logs on stderr. For local servers.
  - **Streamable HTTP:** one endpoint (e.g. `/mcp`). The client POSTs every message. The server answers with JSON or an SSE stream (notifications and responses get `202 Accepted`). An optional GET opens an SSE stream for server-initiated messages (server may return 405). Session via `Mcp-Session-Id`, version via `MCP-Protocol-Version` header, resumability via SSE event IDs. For remote servers.
  - **Legacy HTTP+SSE** (spec 2024-11-05) is deprecated. Custom transports are allowed.
- SSE inside Streamable HTTP is just a streaming response mode. The transport is still HTTP, not WebSocket.
- **Security:** on HTTP, validate the `Origin` header and bind local servers to localhost. Authorization is OAuth-based. For stdio, take credentials from the environment.
- Request IDs must not be null.

## gRPC
- HTTP/2 + Protobuf (default codec; compact binary serialization, not compression). Specified over HTTP/2 only.
- Contract-first via `.proto` with generated stubs.
- **Modes:** unary, server streaming, client streaming, bidirectional.
- **Built in:** deadlines (propagate across calls), cancellation, metadata, interceptors, own status codes (distinct from HTTP status).
- **Evolution:** never reuse or renumber field tags; adding fields is backward compatible; mark removed fields as `reserved`.
- **Ops:**
  - Default max inbound message size is 4 MB in most implementations.
  - Long-lived HTTP/2 connections defeat L4 load balancers, so use an L7 or client-side load balancer.
- **Browsers** can't call it directly: use gRPC-Web (needs a proxy such as Envoy unless the server supports it) or Connect.
- **Best for:** internal service-to-service. **Weak for:** public APIs, debugging.

## GraphQL
- The client's query lists the exact fields it wants, and the response mirrors the query's shape. Typically a single endpoint with a typed schema.
- **Operations:** query, mutation, subscription (over WebSocket via `graphql-ws`, or SSE).
- Fixes over/under-fetching; good as a BFF over many services.
- **Errors:** partial success is possible (`data` and `errors` together). HTTP status is often 200 even on errors, though newer servers can return 4xx for malformed requests (GraphQL-over-HTTP spec).
- **Gotchas:** N+1 (use DataLoader), hard HTTP caching (persisted queries enable GET and CDN caching), needs depth/cost limits, disable introspection on public APIs.

## SOAP
- XML Envelope (optional Header + Body), WSDL contract, standardized Faults.
- An envelope format, not inherently RPC. WSDL binding style is document/literal (the norm) or rpc.
- **Extensions:** WS-Security (message-level signing/encryption), WS-ReliableMessaging, WS-AtomicTransaction.
- **Transport-agnostic** (HTTP, JMS, MQ, SMTP). SOAP over JMS has a W3C spec. On non-HTTP transports, reply routing and correlation come from the transport (`ReplyTo` / `CorrelId`) or WS-Addressing.
- Message-level security survives queues and intermediaries, while TLS protects only one hop. Fits banking; downside is verbosity.

## Messaging as transport (IBM MQ / JMS / Kafka)
- **IBM MQ:** queue broker. Point-to-point queues (plus pub/sub topics); a message is removed once consumed. **JMS** is the Java API, and MQ can be the JMS provider.
- **Kafka:** retained, replayable log; consumers track offsets. MQ suits commands and request-reply, Kafka suits event streams. Both are messaging backbones, not API styles.
- **Request-reply over MQ:**
  1. Requester puts the message (e.g. a SOAP envelope) on the request queue with `ReplyToQ` and a `MsgId`. In JMS: `JMSReplyTo`, `JMSMessageID`.
  2. Responder processes it and puts the reply on `ReplyToQ` with `CorrelId` = the request's `MsgId`. In JMS: `JMSCorrelationID`.
  3. Requester reads the reply by `CorrelId`.
- **Kafka request-reply:** `correlationId` header plus a reply topic.
- **Gains:** durability (persistent messages), transactions, sender and receiver don't need to be up at the same time.
- **Costs:** higher latency. The caller must handle timeouts (message expiry), duplicates on retry (idempotency), and poison messages (backout queue / DLQ).
- **SOAP over MQ:** the envelope is the message body, and WS-Security protects it end to end across the queue.

## WebSocket
- Starts as an HTTP upgrade (`101 Switching Protocols`), then a persistent full-duplex connection over TCP. `wss://` is over TLS, `ws://` is plain.
- Also possible over HTTP/2 (RFC 8441) and HTTP/3 (RFC 9220).
- **Heartbeat:** ping/pong control frames. Reconnect with backoff on the client.
- **No message semantics,** so design your own protocol or use a subprotocol such as STOMP. Socket.IO is its own protocol on top of WebSocket, not interoperable with plain WebSocket.
- **Scaling:**
  - A connection stays on one server for its whole life, so sticky sessions are needed only for fallback transports (e.g. Socket.IO long polling).
  - The load balancer must support `Upgrade`, and its idle timeout must exceed your ping interval.
  - Cross-node fan-out needs a pub/sub backplane (Redis/Kafka).
- **Use:** chat, live tickers, collaboration, games.

## SSE
- One-way server to client over plain HTTP (`text/event-stream`), UTF-8 text only.
- Auto-reconnect with resume via `Last-Event-ID`.
- HTTP/1.1 caps about 6 connections per domain (HTTP/2 multiplexing fixes it).
- Browser `EventSource` supports only GET and no custom headers, so use cookies or a fetch-based client for auth.
- **Use:** notifications, live feeds, LLM token streaming. Simpler than WebSocket when only the server pushes.

## Webhooks
- Server-to-server HTTP callback on an event (the reverse of polling), e.g. payment-status callbacks.
- **Receiver:** verify the HMAC signature, check the signed timestamp to stop replays, return 2xx fast and process async, be idempotent (delivery is at-least-once; duplicates and reordering happen), dedupe by event ID, and re-fetch current state if ordering matters.
- **Sender:** retries with exponential backoff and a dead-letter queue.

## WebRTC
- Media is peer-to-peer (SRTP over UDP) but not always direct. Everything is encrypted (DTLS, which also keys SRTP). Data channels (SCTP over DTLS) carry game or app data too.
- **Signaling is not part of WebRTC:** you build it (usually WebSocket/HTTP) to exchange the SDP offer/answer and ICE candidates.
- **ICE** gathers host, STUN (server-reflexive) and TURN (relay) candidates, tests the pairs, and picks the best working one by priority (direct, then STUN, then TURN). It is not a sequential fallback.
- **STUN** discovers the public address. **TURN** relays when NAT blocks (costly; can fall back to TCP/TLS).
- **Scaling:** a mesh breaks down beyond about 4-6 peers; use an SFU (forwarding) or MCU (mixing).

## Interaction patterns

| Pattern | Direction | Connection | Use |
|---|---|---|---|
| Short polling | Client pulls | New request each time | Simple, wasteful |
| Long polling | Client pulls | Held open until data | Fallback for push |
| SSE | Server to client | One long HTTP response | Feeds, notifications |
| WebSocket | Both ways | Persistent TCP | Chat, trading, games |
| Webhook | Server to server | Per-event HTTP call | Event notification |
| Async request-response | Client to server | `202 Accepted` + `Location` status URL or callback | Long-running jobs |
| Queue request-reply | Both ways | Broker queues + `CorrelId` | Decoupled, guaranteed delivery |

- **Streaming:** gRPC streams at the API level. Kafka (replayable log) and MQ (queue) sit behind the API as messaging backbones, not API styles.

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
| AI agent/tool integration | MCP (JSON-RPC 2.0) |
| Async commands with guaranteed delivery | MQ (or Kafka for event streams) |

## Cross-cutting
- **Reliability:** idempotency keys on POST (essential for payments), timeouts, retries with backoff.
- **Evolution:** versioning (URI/header for REST, field-tag compatibility for Protobuf), cursor pagination over offset.
- **Errors:** consistent model; RFC 9457 (Problem Details) for REST.
- **Security and ops:** OAuth2/JWT for clients, mTLS for service-to-service, rate limiting, API gateway, correlation/trace IDs.
