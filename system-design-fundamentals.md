# System Design Fundamentals — Lesson Notes

Pitched at someone with 4+ years of engineering experience. No "what is a CPU" hand-holding — instead, the mechanics, the trade-offs, and *why an interviewer or a senior engineer cares* about each topic.

---

## SECTION 0 — BACKGROUND

### 0. Computer Architecture (14m)

You already know CPU/RAM/disk. What matters for system design is the **latency hierarchy**, because almost every design decision (caching, replication, batching) is really an argument about where data physically lives relative to the CPU.

Rough numbers worth memorizing (order of magnitude, not exact):

| Operation | Latency |
|---|---|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~4 ns |
| Main memory (RAM) reference | ~100 ns |
| SSD random read | ~10–100 µs |
| HDD random read | ~1–10 ms |
| Round trip within same datacenter | ~0.5 ms |
| Round trip cross-region (e.g., US↔Europe) | ~100–150 ms |

The big takeaway: **network calls dominate everything else by orders of magnitude.** A cache hit in memory is ~100,000x faster than a network round trip to another region. This single fact justifies: in-memory caches, CDNs, read replicas placed near users, batching network calls, and avoiding N+1 query patterns.

Also relevant: disks are sequential-read friendly (this is why append-only logs, WALs, and LSM-trees in NoSQL databases exist — they turn random writes into sequential ones).

**Why it matters:** every "should I cache this?" or "should I denormalize this?" question is really a latency-hierarchy question in disguise.

---

### 1. Application Architecture (11m)

The progression you should be able to narrate:

1. **Monolith** — one codebase, one deployable, one database. Fine until team size or scale requires independent deployability.
2. **Monolith + separate DB read replicas** — scale reads before you scale architecture.
3. **Service-oriented / microservices** — split by business domain, each with its own datastore ideally. Buys you independent scaling and deployment, costs you network calls, distributed transactions, and operational complexity.
4. **Event-driven architecture** — services communicate via events/queues instead of direct calls, enabling async processing and loose coupling.

**Key interview signal:** don't default to microservices because it's trendy. The right answer is almost always "start simple, articulate *specifically* what forces the split" (team autonomy, differing scaling needs, differing tech stack requirements, blast-radius isolation).

**Client-server vs. peer-to-peer** is worth being able to state plainly: virtually all systems you'll design are client-server; P2P (BitTorrent, blockchain) comes up rarely but shows up when asked about file-sharing systems.

---

### 2. Design Requirements (26m)

This is the most important 26 minutes of the whole course, honestly — most system design interviews are lost or won in the first 5 minutes here.

**Functional requirements (FRs):** what the system *does*. Get these from the prompt via clarifying questions. Prioritize the top 3-4 core use cases — don't try to design for every feature.

**Non-functional requirements (NFRs):** the *-ilities* — this is where seniority shows.

- **Availability**: can the system tolerate node failure? (measured in "nines" — 99.9% = ~8.7 hrs downtime/year, 99.99% = ~52 min/year)
- **Consistency**: do all readers see the same data at the same time? (strong vs. eventual — ties directly into CAP, later)
- **Latency**: p50/p95/p99 targets. Always think in percentiles, never averages — averages hide the tail that actually hurts users.
- **Durability**: once written, can data be lost?
- **Scalability**: read-heavy vs write-heavy? Predictable load vs spiky? (this single question drives cache vs. queue vs. sharding decisions)

**Back-of-envelope estimation** — the skill that separates "sounds smart" from "is precise":
- QPS = DAU × actions/user/day ÷ 86,400
- Storage = records/day × avg size/record × retention period
- Bandwidth = QPS × avg payload size

The point isn't precision, it's **directional correctness** — is this 100 QPS (single server is fine) or 100K QPS (you need sharding and caching from day one)? Numbers should change your architecture decisions, not just decorate the whiteboard.

**Framework to always follow:**
1. Clarify FRs (2-3 core ones, don't scope-creep)
2. Clarify NFRs (pick the 2-3 that actually matter for *this* system — a chat app cares about latency + availability, a bank ledger cares about consistency + durability)
3. Napkin math only if it changes a decision
4. Then design.

---

## SECTION 1 — NETWORKING

### 3. Networking Basics (16m)

**IP address**: identifies a machine on a network. **Port**: identifies a process/service on that machine. Together, `IP:port` = a socket.

**OSI model** — you don't need all 7 layers memorized, but know these three cold:
- **Layer 3 (Network)**: IP — routing packets between networks.
- **Layer 4 (Transport)**: TCP/UDP — reliability, ordering, flow control.
- **Layer 7 (Application)**: HTTP, gRPC, WebSocket, DNS — the protocols you actually write code against.

**Latency vs. bandwidth vs. throughput** — commonly conflated:
- **Latency**: time for one packet to travel A→B.
- **Bandwidth**: max theoretical data rate of the pipe.
- **Throughput**: actual achieved data rate (bandwidth is the ceiling, throughput is reality, often lower due to congestion/retries).

**Why it matters:** when someone says "the API is slow," the fix is completely different depending on whether it's a latency problem (physical distance, TCP handshake overhead → use CDN, keep connections alive, move region) or a throughput problem (payload too big, too much data → compress, paginate, cache).

---

### 4. TCP and UDP (10m)

**TCP** — connection-oriented, reliable, ordered, congestion-controlled.
- 3-way handshake (SYN, SYN-ACK, ACK) before any data flows — this costs a full round trip before you've sent a single byte.
- Guarantees delivery + ordering via ACKs and retransmission.
- Used for: HTTP/HTTPS, most APIs, anything where you can't tolerate data loss.

**UDP** — connectionless, no guarantees, no handshake, no ordering.
- Fire and forget. Lower latency because there's no handshake and no retransmission waits.
- Used for: video/audio streaming, live gaming, DNS queries — anywhere a dropped/late packet is worse to wait/retransmit for than to just skip.

**The interview-relevant judgment call:** "should this be TCP or UDP?" really means "can this system tolerate occasional data loss in exchange for lower latency?" Live video call → yes (UDP, a dropped frame is fine). Financial transaction → no (TCP, must be reliable).

Also know: **QUIC** (used in HTTP/3) is UDP-based but adds reliability + built-in TLS + eliminates head-of-line blocking that TCP has at the transport layer — worth name-dropping if asked about modern low-latency HTTP.

---

### 5. DNS (10m)

DNS translates human-readable domain names → IP addresses. The resolution chain:

`Browser cache → OS cache → Router cache → ISP resolver → Root nameserver → TLD nameserver (.com) → Authoritative nameserver → IP returned`

Each layer caches based on **TTL** to avoid re-walking this chain every request (which would be a huge latency hit given the round trips involved).

**System-design-relevant DNS tricks:**
- **DNS-based load balancing / GeoDNS**: return different IPs based on the requester's location → routes users to the nearest datacenter. This is a first line of geographic routing before the request ever hits your load balancer.
- **Weighted round robin DNS**: split traffic across IPs by percentage — used for canary/blue-green rollouts at the DNS level.
- **Low TTLs** let you fail over quickly (change the IP and it propagates fast) but increase load on DNS servers and add latency from more frequent lookups. **High TTLs** are cheaper but slow to fail over. This is a real trade-off you should be able to articulate.

---

## SECTION 2 — APIs

### 6. HTTP (23m)

You know HTTP verbs — focus on the parts people get fuzzy on:

**Statelessness**: each HTTP request carries all context needed (no server-side session memory required) — this is *why* HTTP servers can be scaled horizontally trivially. Session state, if needed, has to be pushed to a cookie/token or an external store (Redis) — never in-process memory, or you break horizontal scaling.

**Status codes that actually matter in interviews:**
- `429 Too Many Requests` — rate limiting
- `503 Service Unavailable` — backpressure/overload signal, should trigger client-side backoff
- `409 Conflict` — optimistic concurrency control failures
- `304 Not Modified` — caching validation

**Headers worth knowing:** `Cache-Control`, `ETag`/`If-None-Match` (conditional requests for caching), `Idempotency-Key` (client-generated key so retried POSTs don't double-execute — critical for payments).

**HTTP/1.1 vs HTTP/2 vs HTTP/3:**
- HTTP/1.1: one request per TCP connection at a time (head-of-line blocking) — browsers work around this by opening multiple connections.
- HTTP/2: multiplexes multiple requests over one TCP connection, adds header compression — big latency win.
- HTTP/3: runs over QUIC (UDP) instead of TCP — removes TCP-level head-of-line blocking entirely, faster connection setup.

**Idempotency** is the single most interview-tested HTTP concept: GET/PUT/DELETE should be idempotent (same call twice = same result), POST is not by default. When designing "process a payment" or "create an order" APIs, you should proactively mention idempotency keys to prevent duplicate charges on retry.

---

### 7. WebSockets (9m)

HTTP is request-response: client always initiates. Some use cases need the **server** to push data to the client without being asked (chat, live notifications, collaborative editing, live dashboards).

Older workaround: **polling** (client asks "anything new?" every N seconds — wastes requests, adds latency up to the poll interval) or **long polling** (server holds the request open until it has data — better, still has overhead of re-establishing connections).

**WebSockets**: a single persistent, full-duplex TCP connection, established via an HTTP handshake (`Upgrade: websocket` header) then upgraded — after that, both sides can push messages anytime with minimal overhead.

**Trade-offs to state explicitly:**
- Cost: each open WebSocket holds server resources (memory, a thread/connection slot) for potentially a long idle time — this changes your capacity planning completely vs. stateless HTTP (you now need to think about "how many concurrent connections can one server hold," and sticky routing/load balancing gets harder because a client's state now lives on one specific server).
- You lose easy horizontal scaling of connections unless you add a pub/sub layer (e.g., Redis Pub/Sub) so any server instance can broadcast to a client connected to a *different* instance.

**When to reach for it:** truly bidirectional, low-latency, high-frequency updates (chat, multiplayer games, live collaboration). For "mostly one-way, occasional push" (e.g., order status updates), **Server-Sent Events (SSE)** is often a simpler fit — one-directional server→client over plain HTTP, easier to scale.

---

### 8. API Paradigms (29m)

**REST** — resource-oriented, verbs (GET/POST/PUT/DELETE) over nouns (URIs). Stateless, cacheable, widely understood, but often results in over-fetching/under-fetching (you get the whole resource even if you needed one field, or you need N calls to stitch data together).

**GraphQL** — client specifies exactly the shape of data it wants in one query, server resolves it. Solves over/under-fetching, great for client teams with varied data needs (mobile vs. web needing different subsets). Costs: harder to cache (queries are arbitrary, not fixed URLs), harder to rate-limit (a "cheap" looking query can trigger an expensive resolver chain), added complexity on the server (resolvers, schema stitching).

**gRPC** — binary protocol (protobuf) over HTTP/2, contract-first (`.proto` files generate client/server stubs). Much lower latency and smaller payloads than JSON/REST, supports streaming natively (client-stream, server-stream, bidirectional). Downsides: not human-readable/debuggable in a browser easily, requires codegen tooling, less ideal for public-facing APIs (browser support is awkward without a proxy).

**When to use which (this is the actual interview answer):**
- Public API, need wide compatibility, cacheability → REST
- Internal service-to-service, need speed and strict typing → gRPC
- Client needs flexible queries over a complex graph of data (e.g., social network feed combining posts+users+comments) → GraphQL
- Real-time bidirectional → WebSockets (not really an "API paradigm" in the REST/GraphQL/gRPC sense, but comes up in the same conversation)

**Message formats**: JSON (human-readable, larger, universal) vs. Protocol Buffers/Avro/Thrift (binary, smaller, faster to (de)serialize, requires schema). Mention this trade-off when discussing internal high-throughput services.

---

### 9. API Design (21m)

Practical rules that read as "senior" in an interview:

- **Resource naming**: nouns, plural, hierarchical (`/users/{id}/orders/{orderId}`), no verbs in the path (`/getUser` is a REST smell).
- **Pagination**: never return unbounded lists.
  - **Offset-based** (`?page=3&limit=20`): simple, but breaks if items are inserted/deleted mid-pagination (items shift), and gets slow for large offsets (DB still has to scan/skip).
  - **Cursor-based** (`?after=<opaque_id>`): stable under concurrent writes, scales better (index seek instead of scan), the standard choice for infinite-scroll feeds.
- **Versioning**: `/v1/` in the URL (simple, visible) vs. header-based versioning (cleaner URLs, less discoverable). Always plan for versioning from day 1 — breaking API changes without it are painful.
- **Rate limiting**: token bucket / leaky bucket algorithms; communicate via `429` + `Retry-After` header.
- **Idempotency** (again, because it keeps coming up): idempotency keys for any non-idempotent mutating operation that might be retried.
- **Filtering/sorting**: via query params, but for anything expensive, consider pushing to a search-optimized store (Elasticsearch) rather than ad-hoc filtering on the primary DB.

**Interview move that scores well:** when asked to "design the API for X," don't just list endpoints — briefly justify pagination strategy and idempotency for write endpoints unprompted. That's the signal that separates someone who's shipped real APIs from someone reciting REST conventions.

---

## SECTION 3 — CACHING BASICS

### 10. Caching (21m)

Caching is fundamentally exploiting the latency hierarchy from Lesson 0 — keep hot data closer (memory) instead of re-fetching/re-computing from something slower (disk, DB, network).

**Where caches live** (know all of these, they show up in different system designs):
- **Client-side** (browser cache, mobile app cache)
- **CDN** (edge, geographically distributed — next lesson)
- **Load balancer / reverse proxy cache**
- **Application-level / in-memory cache** (Redis, Memcached) — the one you'll design with most often
- **Database query cache / buffer pool**

**Cache-aside (lazy loading)**: app checks cache, on miss reads DB and populates cache. Most common pattern. Risk: **thundering herd** — if a hot key expires, many concurrent requests all miss simultaneously and hammer the DB at once. Mitigate with request coalescing/locking or staggered TTLs (add jitter).

**Write-through**: write to cache and DB synchronously. Cache always consistent, but adds write latency.

**Write-back (write-behind)**: write to cache immediately, asynchronously flush to DB later. Fast writes, but risk of data loss if the cache node dies before flush.

**Write-around**: write directly to DB, skip cache (used when written data isn't likely to be read again soon — avoids polluting cache with cold data).

**Eviction policies:**
- **LRU** (evict least recently used) — good general default.
- **LFU** (evict least frequently used) — better when access frequency matters more than recency.
- **TTL-based expiry** — simplest, used alongside LRU/LFU in practice.

**Cache invalidation** — famously "one of the two hard problems in CS." Key strategies: TTL expiry (simplest, accept some staleness), explicit invalidation on write (delete/update the cache key when the underlying data changes — requires discipline to not miss a write path), and versioned/namespaced keys (bump a version number instead of hunting down every key to delete).

**Interview signal:** always state *what staleness is acceptable* for the data in question before picking a strategy. A social media like-count can be a few seconds stale; an account balance generally cannot.

---

### 11. CDNs (11m)

A CDN is caching applied geographically: copies of content live on edge servers physically close to users, so requests don't have to round-trip to your origin datacenter.

**Push CDN**: you proactively upload content to edge nodes (good for static assets that rarely change — you control exactly what's cached and when).

**Pull CDN**: edge node fetches from origin on first request (cache miss), caches it, serves subsequent requests from cache until TTL expires (good for large/less-predictable content — less upfront management).

**What goes on a CDN:** static assets (images, JS/CSS bundles, video), and increasingly, cacheable API responses via edge caching (some CDNs now support serving JSON API responses at the edge, or even running compute at the edge — "edge functions").

**Cache invalidation for CDNs** is the same hard problem as Lesson 10 but geographically distributed and slower to propagate — this is why static assets are typically **versioned in the filename/URL** (`app.a1b2c3.js`) instead of relying on invalidation: a new deploy just gets a new URL, so there's nothing to invalidate, and old cached versions naturally age out.

**Why it matters in a design:** any system serving media (video platform, image-heavy social app) should mention CDN placement early — it removes a huge chunk of load and latency from your origin servers and is usually a "free" architectural win to call out.

---

## SECTION 4 — PROXIES

### 12. Proxies and Load Balancing (14m)

**Forward proxy**: sits in front of *clients*, hides client identity from the server (e.g., corporate proxy, VPN). The server doesn't know which specific client it's talking to.

**Reverse proxy**: sits in front of *servers*, hides server identity/topology from the client (e.g., Nginx, load balancer). The client just sees one endpoint; the proxy routes internally. Also handles SSL termination, compression, request routing.

**Load balancer** is a reverse proxy with a routing algorithm:
- **Round robin**: simple, even distribution, ignores server load/capacity differences.
- **Least connections**: routes to the server currently handling the fewest active connections — better for uneven request durations.
- **Weighted** versions of the above — accounts for heterogeneous server capacity.
- **IP hash / consistent hashing**: same client always routed to the same server — needed when you want **session affinity/sticky sessions** (e.g., WebSocket connections from Lesson 7, or in-memory session state).

**L4 vs L7 load balancing:**
- **L4** (transport layer): routes based on IP/port only, doesn't inspect the request content — faster, less flexible.
- **L7** (application layer): inspects HTTP headers/paths/cookies, can route `/api/video/*` differently than `/api/users/*` — more flexible, slightly more overhead.

**Health checks**: LBs periodically ping backend servers and stop routing to unhealthy ones — this is your first line of automatic failure tolerance, worth mentioning whenever you draw a load balancer box.

**Where to place LBs:** typically at multiple layers — one in front of your app servers, potentially another in front of your DB read replicas.

---

### 13. Consistent Hashing (15m)

**The problem it solves:** naive hashing (`hash(key) % N` servers) means adding/removing *one* server reshuffles almost every key's target server — catastrophic for a distributed cache or sharded DB, since it triggers a mass cache-miss storm or data migration.

**How consistent hashing works:** map both servers and keys onto a conceptual ring (hash space, e.g., 0 to 2^32-1). A key is assigned to the first server found walking clockwise from the key's hash position. When a server is added/removed, **only the keys between it and its predecessor on the ring** need to move — not the whole keyspace.

**Virtual nodes**: each physical server is mapped to *multiple* points on the ring (not just one). This solves two problems: (1) even distribution — without virtual nodes, ring gaps can be uneven, overloading some servers; (2) when a server dies, its load is spread across many other servers instead of dumping entirely onto its one ring-neighbor.

**Where this actually shows up in designs:** sharded caches (Memcached client-side hashing), distributed databases (Cassandra, DynamoDB use it internally for partitioning), and any "which shard/node owns this key" problem where you want minimal data movement on scale-up/down events.

**Interview signal:** if you're designing anything horizontally partitioned (cache cluster, sharded DB, distributed queue), proactively mention consistent hashing as the mechanism that lets you scale in/out without a full rebalance — that's usually the "aha, they get it" moment.

---

## SECTION 5 — STORAGE

### 14. SQL (19m)

**ACID** — the guarantees a relational DB gives you per transaction:
- **Atomicity**: all-or-nothing execution.
- **Consistency**: DB moves from one valid state to another (constraints/foreign keys always hold).
- **Isolation**: concurrent transactions don't see each other's intermediate state (levels: Read Uncommitted → Read Committed → Repeatable Read → Serializable, each trading concurrency for stricter correctness).
- **Durability**: once committed, survives a crash (via write-ahead logging).

**Indexes**: B-tree indexes turn O(n) scans into O(log n) lookups — but every index adds write overhead (each insert/update must also update the index) and storage. Composite indexes matter in the order columns are declared (leftmost-prefix rule).

**Normalization vs. denormalization:**
- Normalized (3NF): no redundant data, strong consistency, but reads require JOINs across tables.
- Denormalized: duplicate data to avoid JOINs, faster reads, but writes must update multiple places (risk of inconsistency) — common trade made deliberately for read-heavy systems.

**When SQL is the right call:** data has clear relationships, you need multi-row/multi-table transactions (money movement, inventory + order together), strong consistency matters, and query patterns aren't fully known upfront (SQL's flexible querying via JOINs is a real advantage over NoSQL's fixed access patterns).

**Scaling SQL:** vertical scaling first (bigger box), then read replicas (offload reads), then sharding (harder — loses easy cross-shard JOINs and transactions) — the natural next lesson.

---

### 15. NoSQL (18m)

Not "no SQL," more like "not only SQL" — a family of models that trade relational flexibility for horizontal scalability and/or specific access patterns.

**Key-value stores** (Redis, DynamoDB): simplest model, O(1) lookups by key, great for caching/session storage/simple lookups. No query flexibility beyond the key.

**Document stores** (MongoDB): JSON-like documents, flexible schema, good when your data is naturally nested/hierarchical and you mostly fetch whole documents by ID (product catalogs, user profiles).

**Wide-column stores** (Cassandra, HBase): rows can have different columns, optimized for huge write throughput and range scans over a partition key + sort key — classic fit for time-series data (event logs, sensor data, activity feeds).

**Graph databases** (Neo4j): nodes + edges, optimized for traversal queries (friend-of-friend, recommendation paths) that would require expensive recursive JOINs in SQL.

**The core trade-off vs. SQL:** you generally give up flexible ad-hoc querying and often give up strong multi-record transactions, in exchange for horizontal scalability and higher write throughput. Schema is typically defined by *access pattern first* (design your table around "what queries will I run" — the opposite of SQL's normalize-first approach).

**Interview signal:** don't say "NoSQL is faster" — that's imprecise and gets pushed on. Say: "NoSQL trades query flexibility and (often) strong consistency for horizontal write scalability and predictable low-latency access on known query patterns." Then justify the specific NoSQL type by the actual access pattern in the prompt.

---

### 16. Replication and Sharding (17m)

Two different axes of scaling that get conflated — keep them distinct.

**Replication** = copies of the *same* data on multiple nodes. Purpose: availability (failover) and read scaling (route reads to replicas).
- **Leader-follower (primary-replica)**: one node accepts writes, replicates to followers; followers serve reads. Simple, but leader is a single point of write-bottleneck and a failover event (leader dies) requires promoting a follower — with a window of unavailability or data loss depending on replication lag.
- **Multi-leader**: multiple nodes accept writes, replicate to each other. Enables writes in multiple regions, but now you have to handle write conflicts (two leaders got different writes to the same record) — needs conflict resolution (last-write-wins, vector clocks, CRDTs).
- **Synchronous vs. asynchronous replication**: sync = leader waits for replica ack before confirming write (strong consistency, higher write latency, replica failure blocks writes); async = leader confirms immediately, replicates in background (lower latency, but replica can lag → stale reads, and potential data loss if leader dies before replicating).

**Sharding (partitioning)** = splitting *different* data across multiple nodes, each holding a subset. Purpose: scale writes and total data volume beyond what one machine can hold.
- **Shard key choice is the whole game**: pick a key that (a) distributes load evenly (avoid hotspots — e.g., don't shard by date if all traffic is "today") and (b) matches your query patterns (queries needing data from a single shard are cheap; cross-shard queries/joins are expensive or require a scatter-gather + merge).
- Consistent hashing (Lesson 13) is the usual mechanism for mapping keys → shards while minimizing reshuffling on scale events.
- **Costs**: cross-shard transactions and joins become hard/slow, rebalancing is operationally heavy, and you often need a routing layer to know which shard owns a given key.

**Typical combined pattern in real systems:** shard for write scale, replicate each shard for read scale + availability. Worth drawing this explicitly in a design — "3 shards, each with 1 leader + 2 read replicas" is a concrete, defensible architecture statement.

---

### 17. CAP Theorem (12m)

States: in the presence of a **network partition (P)** — which *will* happen in any distributed system — you must choose between **Consistency (C)** and **Availability (A)**. You cannot have all three simultaneously during a partition.

- **CP (consistent, not available during partition)**: nodes that can't confirm they have the latest data refuse to respond rather than risk serving stale data. Example fit: bank balances, inventory counts where overselling is unacceptable.
- **AP (available, not consistent during partition)**: nodes keep serving even if they might be stale, and reconcile later. Example fit: social media likes/view counts, shopping cart items, DNS — anywhere brief staleness is an acceptable cost for uptime.

**Important nuance that separates senior answers from textbook answers:** CAP only applies *during an actual network partition* — most of the time your system isn't partitioned, and the real day-to-day trade-off engineers make is better described by **PACELC**: if Partitioned, choose A or C (as CAP says); Else (no partition), choose between Latency or Consistency anyway, because even synchronous replication for strong consistency costs latency.

**How this shows up in an interview:** don't just say "we chose AP." Say *which specific data* needs which guarantee — most real systems are a mix (e.g., an e-commerce site: inventory count needs CP-like guarantees to prevent overselling, but the product catalog/reviews can be AP). Blanket-labeling an entire system as CP or AP is a red flag; per-data-type reasoning is the strong signal.

---

### 18. Object Storage (6m)

Object storage (S3, GCS, Azure Blob) stores data as flat, immutable **objects** (blob + metadata + unique key) in a **bucket** — no folder hierarchy really, no file-system semantics, no partial in-place edits (you replace the whole object, you don't seek-and-modify like a file).

**Why not just use a regular DB or filesystem for large files?** Object storage is built for massive horizontal scale, is extremely durable (S3 quotes 11 nines via automatic multi-AZ replication), and is far cheaper per GB than block/file storage or a database — but it has higher per-request latency and no support for partial updates or strong transactional semantics.

**Use it for:** images, videos, backups, logs, static site assets, data lake files (Parquet/CSV for analytics) — i.e., large blobs accessed by key, not things you query by arbitrary field or update incrementally.

**Common pattern:** store the *metadata* (owner, title, upload time, permissions) in your regular DB, and store just a reference/URL to the actual blob in object storage. Also commonly paired with a CDN in front of it (Lesson 11) for public-facing content like profile pictures or video files.

---

## SECTION 6 — BIG DATA

### 19. Message Queues (8m)

A message queue decouples **producers** (services emitting work/events) from **consumers** (services processing them), with the queue buffering in between. This is the core building block of **asynchronous processing**.

**Why decouple at all:** if your API directly calls a slow downstream service synchronously (e.g., "resize this video" inline in the upload request), the client waits for the whole chain, and a slow/down downstream service takes your whole API down with it. Put a queue between them: the API just enqueues the job and returns immediately; a worker pool consumes at its own pace. This also smooths out traffic spikes — the queue absorbs a burst, workers drain it steadily instead of the backend being hit with the full spike.

**Queue vs. Pub/Sub (Topic):**
- **Queue** (e.g., SQS, RabbitMQ): each message consumed by exactly *one* consumer — good for distributing discrete units of work across a worker pool.
- **Pub/Sub / Topic** (e.g., Kafka, SNS): each message delivered to *every* subscriber — good for fanning an event out to multiple independent downstream systems (e.g., "order placed" event triggers email service, analytics service, and inventory service, all independently).

**Delivery guarantees** — know these three terms cold:
- **At-most-once**: message might be lost, never duplicated.
- **At-least-once**: message never lost, but might be delivered/processed twice — **this means your consumers must be idempotent** (same theme as HTTP idempotency in Lesson 6/9).
- **Exactly-once**: the ideal, genuinely hard to guarantee end-to-end in practice — usually achieved by combining at-least-once delivery with idempotent consumers, rather than a true single-delivery guarantee.

**Interview signal:** whenever you introduce a queue, proactively say "consumers need to be idempotent since most queues guarantee at-least-once delivery" — that one sentence signals real production experience.

---

### 20. MapReduce (12m)

A programming model for processing huge datasets in parallel across many machines, popularized by Google, implemented by Hadoop, and conceptually still underlying Spark (even though Spark improved on it significantly).

**Map phase**: input data is split into chunks, each processed independently in parallel, emitting intermediate **key-value pairs**. (e.g., word count: map each line of text → emit `(word, 1)` for every word.)

**Shuffle phase**: intermediate pairs are grouped by key and routed to reducers — all values for the same key end up on the same reducer node. This is the expensive, network-heavy part (data movement across the cluster) and usually the actual bottleneck in a MapReduce job.

**Reduce phase**: each reducer aggregates all values for its assigned keys into a final result. (word count: sum all the `1`s for each word → final counts.)

**Why this matters even though most people use Spark/managed services now:** the *mental model* — split, process in parallel, group by key, aggregate — underlies virtually all distributed data processing (Spark, Flink, even distributed SQL query engines). If asked to design an analytics pipeline ("compute daily active users from raw event logs"), describing it in map/shuffle/reduce terms demonstrates you understand how the parallelism and the bottleneck (shuffle/network I/O) actually work, rather than just naming a tool.

**Batch vs. stream processing**, worth contrasting since it comes up in the same breath: MapReduce/Spark batch jobs process a bounded dataset (e.g., "yesterday's logs") on a schedule; stream processing (Kafka Streams, Flink in streaming mode) processes unbounded data continuously as it arrives, trading some accuracy/completeness (windowing, late-arriving data) for near-real-time results. Real systems increasingly use both — batch for accurate historical reports, streaming for real-time dashboards, often reconciled with each other (the "lambda architecture" pattern).

---

## How These Connect (the part textbooks skip)

A quick mental map of how this whole course composes into one design, since these lessons feel siloed individually:

**Client → DNS (5) → CDN (11, for static assets) → Load Balancer (12) → App servers (1) speaking HTTP/gRPC/WebSocket (6-9) → Cache (10) → Database, replicated + sharded via consistent hashing (13, 16) → Object storage for blobs (18) → Message queue (19) for async work → Batch/stream jobs (20) for analytics, feeding back into the cache or a search index.**

Every one of these is a lever you pull for a specific NFR from Lesson 2: latency → cache/CDN; availability → replication/load balancing/health checks; consistency → CAP-aware data placement; scale → sharding/queues/async processing. When you get a system design prompt, the exercise is really: "given these specific NFRs, which levers do I pull, and why not the others?"
