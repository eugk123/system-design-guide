# Networking, Load Balancing & API Design

The "front half" of every system: how a request gets from a client to your servers, how
traffic is spread, and how clients talk to your service.

## The request path

```
Client → DNS resolve → (CDN edge) → Load Balancer → Reverse Proxy → App servers → ...
```

### DNS
- Maps `api.example.com` → IP(s). Cached at multiple layers (browser, OS, resolver) with a TTL.
- **GeoDNS / latency-based routing** returns different IPs by client location → routes users to
  the nearest region. First lever for global latency.
- **Anycast**: one IP announced from many locations; network routes to nearest. Used by CDNs
  and DNS providers.
- DNS is also a coarse failover/load-balancing tool (round-robin A records), but TTL caching
  makes it slow to react — don't rely on it for fast failover.

### CDN (Content Delivery Network)
- Geographically distributed edge caches. Serve static assets (images, JS, video) close to users.
- **Pull** (origin-pull): CDN fetches from origin on first miss, caches it. **Push**: you
  upload to the CDN proactively.
- Cache key = URL (+ headers). Control with `Cache-Control`, `ETag`, TTLs. Invalidate via
  purge or versioned URLs (`/img/logo.v2.png`).
- Massive win for read-heavy media. Always mention CDN when serving blobs.

## Load balancing

Spreads requests across healthy servers; provides a stable entry point; enables horizontal scale.

### L4 vs L7
- **L4 (transport)**: routes by IP/port, no payload inspection. Fast, protocol-agnostic.
  Can't do content-based routing.
- **L7 (application)**: understands HTTP. Routes by path/host/header/cookie, does TLS
  termination, sticky sessions, request rewriting. More flexible, slightly more overhead.

### Algorithms
- **Round robin** / weighted round robin — simple, ignores load.
- **Least connections** — send to the server with fewest active connections; better for
  uneven request durations.
- **Least response time** — factors in latency.
- **Consistent hashing** — same key → same server; minimizes reshuffling when servers
  change. Critical for caches and sharded stateful services (see below).
- **IP hash / sticky sessions** — pin a client to a server (needed only if state is local;
  prefer stateless servers + shared session store instead).

### Health checks & redundancy
- LB polls backends (`/healthz`); removes unhealthy ones from rotation.
- The LB itself must not be a SPOF: active-passive pair with a floating/virtual IP, or
  multiple LBs behind DNS / anycast.

### Consistent hashing (important)
Naive `hash(key) % N` remaps almost everything when `N` changes (a server is added/removed),
causing a cache-wide miss storm. **Consistent hashing** places servers and keys on a hash
ring; a key maps to the next server clockwise. Adding/removing a node only moves keys in one
arc → ~1/N keys move. Use **virtual nodes** (each physical node gets many ring positions) to
even out distribution and handle heterogeneous capacity. This underpins distributed caches,
Cassandra/DynamoDB partitioning, and sharded routing.

## Stateless vs stateful servers
- **Stateless app servers** are the default: any server can handle any request; scale by
  adding more behind the LB; a dead server loses nothing. Put session/user state in a shared
  store (Redis) or a signed token (JWT).
- **Stateful services** (e.g. a websocket gateway holding live connections) need sticky
  routing and careful failover. Keep state minimal and externalized where possible.

## Protocols
- **HTTP/1.1**: one request per connection at a time (head-of-line blocking); keep-alive reuses connections.
- **HTTP/2**: multiplexed streams over one connection, header compression, server push. Lower latency.
- **HTTP/3 (QUIC)**: over UDP, eliminates TCP head-of-line blocking, faster connection setup. Good on lossy mobile networks.
- **WebSocket**: full-duplex persistent connection. Use for chat, live updates, presence.
- **gRPC**: HTTP/2 + Protocol Buffers; binary, strongly-typed, streaming. Great for internal
  service-to-service RPC. Less browser-friendly than REST/JSON.
- **TCP vs UDP**: TCP = reliable, ordered, connection-oriented (most things). UDP =
  fire-and-forget, low overhead (video/voice, gaming, DNS, where loss is tolerable).

## API design styles

### REST
- Resources + HTTP verbs. `GET` (read, safe, cacheable), `POST` (create), `PUT`/`PATCH`
  (update), `DELETE`. Stateless. Status codes carry meaning (2xx/4xx/5xx).
- **Idempotency**: `GET/PUT/DELETE` should be idempotent; `POST` is not — protect it with an
  **idempotency key** so retries don't double-charge/double-create.
- **Pagination**: prefer **cursor-based** (opaque token) over offset/limit for large or
  changing datasets — offset gets slow and skips/duplicates rows as data shifts.
- **Versioning**: `/v1/...` in the path, or via header. Plan for it from day one.

### gRPC — internal RPC where latency/throughput matter (typed contracts, streaming, binary).
### GraphQL — client-specified queries; one endpoint returns exactly the fields requested.
  Solves over-/under-fetching for rich clients (mobile). Costs: complex caching, risk of
  expensive nested queries (need depth/complexity limits), N+1 resolution.

### Choosing
- Public, broad, cache-friendly, simple → **REST**.
- Internal, high-throughput, polyglot services → **gRPC**.
- Diverse clients needing flexible, aggregated reads → **GraphQL** (often as a gateway over
  REST/gRPC services).

## Cross-cutting API concerns
- **Auth**: API keys, OAuth2 / OIDC, signed tokens (JWT). Validate at the edge/gateway.
- **Rate limiting** (see `problems/02-rate-limiter.md`) — protect against abuse and overload.
- **Idempotency keys** for unsafe writes.
- **Backward compatibility** — additive changes only within a version; never repurpose a field.
- **An API gateway** consolidates auth, rate limiting, routing, TLS, and observability in
  front of many services.

## Interview signals
- You terminate TLS at an L7 LB/gateway and keep app servers stateless.
- You reach for consistent hashing when discussing caches or sharding, and can explain *why*.
- You put static/media behind a CDN and reason about cache TTL/invalidation.
- You design idempotent writes and cursor pagination without being prompted.
- You pick a protocol by use case (WebSocket for chat, gRPC for internal, REST for public).
