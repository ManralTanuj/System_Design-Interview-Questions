# System Design Interview Q&A — .NET Developer (4–5 Years)

Each question includes a **model answer** written the way you'd actually say it in an interview, plus what the interviewer is probing for. Keep answers conversational — tie them to real experience where you can.

---

## Section 1 — Fundamentals & Architecture

### 1. Walk me through the lifecycle of an HTTP request in ASP.NET Core.

**Answer:** A request hits **Kestrel** (the built-in web server), often behind a reverse proxy like IIS or Nginx. Kestrel hands it into the **middleware pipeline** — an ordered chain where each piece can inspect/modify the request and response (e.g., exception handling, HTTPS redirection, authentication, authorization, then routing). **Routing** matches the URL to an endpoint. **Model binding** maps query/route/body data to action parameters, and **model validation** runs. Dependency injection resolves the controller and its dependencies. The action executes, returns an `IActionResult`, which gets serialized (usually to JSON) and written back through the pipeline in reverse order.

*Key point to land:* middleware order matters — auth must come before authorization, exception handling near the top.

### 2. Monolith vs microservices vs modular monolith — when would you NOT use microservices?

**Answer:** A **monolith** is one deployable unit — simple to build, test, and deploy, but hard to scale teams and selectively scale parts. **Microservices** split the system into independently deployable services, giving independent scaling and team autonomy, but you pay for it with network latency, distributed transactions, eventual consistency, and heavy operational overhead. A **modular monolith** is a middle ground — one deployable, but with strong internal module boundaries.

I would **not** recommend microservices for a small team, an early-stage product with unclear boundaries, or when the org lacks DevOps maturity. You inherit a distributed system's complexity before you've earned the benefits. I'd start with a modular monolith and extract services only when a clear scaling or team-boundary pain appears.

### 3. Explain DI lifetimes (Singleton, Scoped, Transient) and a bug each can cause.

**Answer:**
- **Singleton** — one instance for the app's lifetime. Bug: if it holds mutable state and isn't thread-safe, you get race conditions across concurrent requests.
- **Scoped** — one instance per request. Bug: the **captive dependency** problem — injecting a Scoped service into a Singleton accidentally promotes it to singleton lifetime, so it never gets refreshed and can leak request data across requests.
- **Transient** — a new instance every time it's requested. Bug: injecting a Transient `DbContext`-like resource where you actually wanted one shared per request, causing extra allocations or inconsistent state.

*Rule of thumb:* `DbContext` is Scoped; stateless helpers can be Transient or Singleton; anything mutable + Singleton must be thread-safe.

### 4. Design a layered architecture for a .NET web API — what goes in each layer?

**Answer:** I'd use a Clean/Onion-style layering:
- **API layer (Controllers/Minimal APIs)** — HTTP concerns only: routing, model binding, returning status codes. No business logic.
- **Application layer (Services / Use Cases)** — orchestrates business workflows, defines interfaces (e.g., `IOrderRepository`), holds DTOs and validation.
- **Domain layer** — entities, value objects, domain rules. No dependency on infrastructure.
- **Infrastructure layer** — EF Core, external API clients, file/blob storage, email — implementations of the interfaces the application layer defines.

The dependency rule: outer layers depend on inner layers, never the reverse. The domain knows nothing about EF Core or HTTP.

### 5. Sync vs async — why does async/await matter for scalability?

**Answer:** ASP.NET Core handles requests on a limited **thread pool**. With synchronous I/O (a DB call, an HTTP call), the thread sits **blocked** doing nothing while waiting. Under load, all threads get blocked waiting, new requests queue up, and throughput collapses — that's **thread-pool starvation**. With `async/await` on I/O-bound work, the thread is **released** back to the pool during the wait and picks up other requests; when the I/O completes, execution resumes on an available thread. So async doesn't make a single request faster — it lets the same number of threads serve far more concurrent requests.

*Caveat:* async helps **I/O-bound** work. For **CPU-bound** work it doesn't help and can hurt; don't fake it with `Task.Run` in web requests.

### 6. Design a URL shortener (bit.ly).

**Answer:**
- **API:** `POST /shorten` (takes long URL → returns short code) and `GET /{code}` (redirects).
- **Code generation:** I'd use a **counter + base62 encoding** approach — an auto-increment ID encoded into base62 (`[a-zA-Z0-9]`) gives short, collision-free codes. Hashing the URL is an alternative but needs collision handling. For distributed ID generation, a range-based allocator or Snowflake-style IDs avoids a single bottleneck.
- **Storage:** key-value lookup (`code → long URL`). Read-heavy, so a KV store or indexed table works; add **Redis caching** in front since the same links get hit repeatedly.
- **Redirect:** return **301** for permanent (better caching) or **302** if you want to keep counting clicks server-side.
- **Extras:** analytics (click counts via async events), custom aliases, expiry.

### 7. How do you handle configuration and secrets across environments?

**Answer:** I layer configuration: `appsettings.json` for defaults, `appsettings.{Environment}.json` for per-environment overrides, then **environment variables** (which win), bound to strongly-typed classes via the `IOptions<T>` pattern. Secrets never go in source control — locally I use **User Secrets**, and in production a vault like **Azure Key Vault** or **AWS Secrets Manager**, injected at runtime. The `ASPNETCORE_ENVIRONMENT` variable drives which environment config loads.

---

## Section 2 — Databases & Data Modeling

### 8. SQL vs NoSQL — how do you decide?

**Answer:** I start from **access patterns and consistency needs**, not preference. I pick **SQL** when I need strong consistency, transactions across entities, complex joins/reporting, and the schema is relatively stable — e.g., orders, payments, accounting. I lean **NoSQL** when I need flexible/evolving schemas, massive horizontal scale, or my access is mostly key-based or document-shaped — e.g., user sessions, product catalogs, event logs. It's rarely all-or-nothing; many systems use SQL for core transactional data and a document/KV store for caching, sessions, or search.

### 9. A query is slow in SQL Server — how do you diagnose and fix it?

**Answer:** First I look at the **actual execution plan** for table scans, missing index warnings, and expensive operators (key lookups, sorts). Common causes and fixes: a **missing index** on a filter/join column; **`SELECT *`** pulling unneeded columns (project only what's needed, or use a covering index); **parameter sniffing** giving a bad cached plan (`OPTION (RECOMPILE)` or optimize-for); **stale statistics** (update them); and at the ORM level, an **N+1 pattern** from EF Core lazy loading. I also check whether the query should be paginated rather than returning everything.

### 10. The N+1 problem in EF Core — what is it and how do you fix it?

**Answer:** N+1 happens when you load a list of N parent rows (1 query), then accessing a navigation property on each triggers a separate query per item — N more queries. Usually it's **lazy loading** silently firing queries inside a loop. Fixes: use **`.Include()`** for eager loading, or better, **project with `.Select()`** into a DTO so EF generates a single optimized join and only pulls needed columns. For one-to-many that causes row explosion, **`AsSplitQuery()`** splits it into separate efficient queries.

### 11. Design the schema for an e-commerce order system.

**Answer:** Core tables:
- `Customers` (Id, name, email…)
- `Products` (Id, name, current price, …)
- `Inventory` (ProductId, quantity, rowversion for concurrency)
- `Orders` (Id, CustomerId, status, created date, total)
- `OrderItems` (Id, OrderId, ProductId, quantity, **unit price at time of order**)

Key decisions: store the **price at purchase time** on `OrderItems` because product prices change — the order must reflect what the customer actually paid. Use **foreign keys** for integrity, **soft deletes** (an `IsDeleted` flag) for products so historical orders stay valid, and a **concurrency token** on inventory so two simultaneous purchases can't oversell stock.

### 12. Zero-downtime database migrations in production?

**Answer:** I use the **expand-contract (parallel change) pattern**. Instead of a breaking change, I make backward-compatible steps: **expand** — add the new column/table (nullable, no breaking change), deploy code that writes to both old and new; backfill data; switch reads to the new structure; then **contract** — once nothing uses the old column, remove it in a later migration. Each step is deployable independently, so old and new app versions can both run during a rolling deploy. EF Core migrations handle the schema changes; I avoid destructive changes (renaming/dropping columns) in the same deploy as the code that depends on them.

### 13. Explain indexing and the cost of too many indexes.

**Answer:** An index is a sorted data structure (a **B-tree** in SQL Server) that lets the engine find rows without scanning the whole table. A **clustered index** defines the physical row order (one per table); **non-clustered indexes** are separate structures pointing back to rows. They make **reads** fast. The cost: every **write** (insert/update/delete) must also update every affected index, so too many indexes slow down writes and consume storage. The goal is indexing the columns you actually filter/join/sort on — and using **covering indexes** (including needed columns) for hot queries — not indexing everything.

### 14. Optimistic vs pessimistic concurrency — how do you implement optimistic in EF Core?

**Answer:** **Pessimistic** concurrency locks the row when read so others wait — safe but hurts throughput and risks deadlocks. **Optimistic** assumes conflicts are rare: you read freely, and only check for a conflict at save time. In EF Core I add a **`rowversion`/concurrency token** column; EF includes its original value in the `WHERE` clause of the `UPDATE`. If another transaction changed the row, zero rows match, EF throws a `DbUpdateConcurrencyException`, and I handle it — typically reload, merge, and retry, or surface a conflict to the user. I prefer optimistic for web apps where conflicts are uncommon.

### 15. Design read/write separation for a read-heavy database.

**Answer:** I'd set up a **primary (write) database with one or more read replicas**. Writes go to the primary; reads are routed to replicas to spread load. The trade-off is **replication lag** — replicas are eventually consistent, so a user might not immediately see their own write. I mitigate that with "read-your-writes" routing (send a user's reads to the primary briefly after they write) for sensitive flows. This pairs naturally with **CQRS**, where the write model and read model are separated, sometimes with denormalized read stores optimized for queries.

---

## Section 3 — Caching & Performance

### 16. Cache strategy for a read-heavy product catalog API.

**Answer:** Catalog data is read-millions/written-rarely — ideal for caching. I'd use the **cache-aside pattern**: on read, check the cache; on a miss, load from DB, store in cache with a TTL, return it. Because the app runs on multiple instances behind a load balancer, I'd use a **distributed cache (Redis)** so all instances share it. On a product update, I **invalidate** (or update) the relevant cache key so stale data doesn't linger. I'd set a sensible TTL as a safety net even with active invalidation.

### 17. `IMemoryCache` vs Redis — when do you need distributed?

**Answer:** `IMemoryCache` stores data **in the process's memory** — fast, but local to one instance. The moment you run multiple instances behind a load balancer, each has its own cache: low hit rates, inconsistent data, and you'd need sticky sessions to compensate (a smell). A **distributed cache like Redis** is shared across all instances, survives app restarts, and gives consistent cached state. Rule: single instance or truly per-instance data → `IMemoryCache`; multi-instance or shared state → Redis.

### 18. How do you handle cache invalidation, and why is it hard?

**Answer:** The hard part is that cached data can go **stale** the moment the source changes, and knowing exactly *what* to invalidate isn't always obvious (one update can affect many cached views). Strategies: **TTL** (simplest — accept staleness up to the expiry), **write-through** (update cache and DB together), **cache-aside with explicit invalidation** (delete the key on write), and **event-driven invalidation** (publish a "product changed" event that invalidates related keys across services). There's no perfect answer — it's a trade-off between freshness, complexity, and performance, which is why it's famously "one of the two hard problems."

### 19. Cache stampede / thundering herd — what is it and how do you mitigate it?

**Answer:** When a hot cache key **expires**, many concurrent requests all miss at once and stampede the database to rebuild it — a load spike that can take the DB down. Mitigations: a **lock / request coalescing** so only one request rebuilds the value while others wait for it; **staggered/jittered expiry** so keys don't all expire at the same instant; and **refresh-ahead**, where you proactively refresh a key before it expires so there's never a cold miss under load.

### 20. API response times spiked under load — how do you investigate?

**Answer:** I start with **data, not guesses** — APM/Application Insights to see *where* time goes: is it the database, an external dependency, CPU, or GC? Common culprits: a slow/blocking DB query or **connection pool exhaustion**; **sync-over-async** blocking thread-pool threads; missing caching causing repeated expensive work; **GC pressure** from excessive allocations; or a slow downstream dependency with no timeout. I'd correlate the spike with deploys, traffic patterns, and resource metrics, then drill into the worst offender first.

---

## Section 4 — Messaging, Queues & Async Workflows

### 21. When do you use a message queue instead of a direct API call?

**Answer:** I reach for a queue when I want to **decouple** producer and consumer, **level load** (absorb spikes and process at a steady rate), do **async work** the user shouldn't wait for, or need **reliability** through retries and a dead-letter queue. A direct API call is right when I need an **immediate, synchronous response** and the caller can't proceed without it. Example: a checkout *should* synchronously confirm payment, but *should* enqueue the confirmation email and analytics rather than block on them.

### 22. Send notifications without blocking the user request.

**Answer:** The request handler should do the minimum and return fast. So I **enqueue** a notification job (to Service Bus / RabbitMQ, or a DB-backed queue) and return immediately. A **background worker** (`BackgroundService` or a separate consumer) picks up the job and sends the email/SMS via the provider. I add **retries with exponential backoff** for transient failures and a **dead-letter queue** for messages that keep failing, so a flaky email provider never affects checkout latency or reliability.

### 23. Queue vs pub/sub topic — use case for each?

**Answer:** A **queue** is point-to-point: a message is consumed by exactly **one** consumer — good for work distribution, like an "process this order" job picked up by one worker. A **pub/sub topic** broadcasts each message to **many** subscribers independently — good for events where multiple systems react, like an `OrderPlaced` event that triggers inventory, email, and analytics services separately. Rule: command/work → queue; event/notification fan-out → topic.

### 24. Can you guarantee exactly-once processing?

**Answer:** Truly exactly-once delivery is effectively impossible in a distributed system — networks fail and acknowledgments get lost. What's practical is **at-least-once delivery + idempotent processing**, which gives *effectively* exactly-once results. The broker may deliver a message more than once, but the consumer detects duplicates (via a message ID / idempotency key it has already processed) and safely ignores repeats. So instead of chasing perfect delivery, I design consumers to be idempotent.

### 25. What is idempotency, and how do you make "create payment" idempotent?

**Answer:** Idempotent means doing the same operation multiple times has the same effect as doing it once. For "create payment," I require the client to send an **idempotency key** (e.g., a GUID per attempt). On the server, before charging, I check whether that key was already processed: if yes, I return the **stored original result** instead of charging again; if no, I process the payment and persist the result against that key (ideally in the same transaction). So a retry after a timeout never double-charges the customer.

### 26. Implement a nightly report-generation job in .NET.

**Answer:** For an in-process scheduled task, I'd use a **`BackgroundService` (`IHostedService`)** with a timer, or better, **Hangfire** or **Quartz.NET** for proper scheduling, retries, and a dashboard. For a multi-instance deployment I'd ensure only one instance runs the job (distributed lock or a leader), or offload scheduling to an external scheduler (cron / cloud scheduler) that triggers an endpoint or enqueues a job. Whatever the trigger, the job itself should be **idempotent**, **observable** (logged, monitored), and **resilient** (retry on transient failure, alert on hard failure).

---

## Section 5 — Scalability & Reliability

### 27. Vertical vs horizontal scaling — what makes an app horizontally scalable?

**Answer:** **Vertical scaling** = a bigger machine (more CPU/RAM) — simple but capped and a single point of failure. **Horizontal scaling** = more instances behind a load balancer — near-limitless and more resilient, but the app must be designed for it. The key requirement is **statelessness**: no user state stored in instance memory or local disk, because any request can land on any instance. State goes to shared stores (Redis, DB, blob storage). Operations should be idempotent, and instances must be interchangeable.

### 28. How do you make an ASP.NET Core app stateless behind a load balancer?

**Answer:** Move anything per-user out of process memory. **Session state** goes to a distributed store like **Redis** (or DB) instead of in-memory. Don't rely on local files — use blob/object storage. Avoid in-memory caches as a source of truth across instances (use a distributed cache). Avoid sticky sessions as a crutch — they reduce the load balancer's flexibility and hide statefulness. Once any instance can handle any request identically, you can scale out freely and survive instance restarts.

### 29. Design a rate limiter for a public API.

**Answer:** I'd use a **token bucket** or **sliding window** algorithm. In .NET 7+, there's **built-in rate limiting middleware** supporting fixed window, sliding window, token bucket, and concurrency limiters. I'd key limits **per API key or per user** (and maybe per IP for anonymous traffic), store counters in a **distributed store (Redis)** so limits hold across instances, and return **HTTP 429 Too Many Requests** with a `Retry-After` header when exceeded. Tiered limits (free vs paid) map to different bucket sizes per key.

### 30. Circuit Breaker and Retry patterns — how does Polly help?

**Answer:** **Retry** re-attempts a transient failure, ideally with **exponential backoff and jitter** so you don't hammer a struggling service in sync. **Circuit Breaker** tracks failures and, after a threshold, "opens" — failing fast for a cool-down period instead of waiting on a dead dependency — then "half-opens" to test recovery. This prevents **cascading failures** where one slow service exhausts your threads and takes you down too. **Polly** is the .NET resilience library that provides these policies (retry, circuit breaker, timeout, bulkhead, fallback) and composes them, integrated cleanly via `HttpClientFactory`.

### 31. Health checks and graceful shutdown for a containerized .NET service?

**Answer:** I expose **health check endpoints** using `IHealthCheck`: a **liveness** probe (is the process alive? restart if not) and a **readiness** probe (is it ready to take traffic? check DB/dependencies — if not ready, the load balancer stops routing to it). For **graceful shutdown**, ASP.NET Core listens for SIGTERM and runs a shutdown sequence: stop accepting new requests, **drain in-flight requests** within a timeout, flush/dispose resources. I hook `IHostApplicationLifetime` for custom cleanup. This means deploys and scale-downs don't drop active requests.

### 32. A downstream dependency is slow/unreliable — how do you protect your service?

**Answer:** Layered defenses: a **timeout** so a hung call doesn't block a thread indefinitely; **retry with backoff** for transient blips; a **circuit breaker** to fail fast when it's clearly down; **bulkhead isolation** so calls to that one dependency can't consume all your threads/connections; **caching** to serve stale-but-acceptable data when it's unavailable; and a **graceful fallback / degraded response** so the user gets a reduced experience rather than an error. The principle: a failure in one dependency must not cascade into a failure of my whole service.

---

## Section 6 — APIs, Security & Observability

### 33. How do you version a REST API and what are the trade-offs?

**Answer:** Three common approaches: **URL path** (`/v1/products`) — explicit, easy to route and cache, but pollutes the URL; **query string** (`?api-version=1.0`) — flexible but easy to miss; **header-based** (`api-version` or content negotiation) — keeps URLs clean and is "purer" REST, but harder to test and discover. In practice I usually go **URL path versioning** for clarity, combined with the `Asp.Versioning` library. The real discipline is a **deprecation policy**: support old versions for a defined window, communicate sunset dates, and avoid breaking changes within a version.

### 34. Authentication vs authorization — secure a .NET API with JWT.

**Answer:** **Authentication** = who you are; **authorization** = what you're allowed to do. With **JWT**: the user authenticates (against an identity provider / auth server), which issues a **signed access token** containing claims (user id, roles). The client sends it as a `Bearer` token. The API uses JWT middleware to **validate the signature, issuer, audience, and expiry** on each request — no DB lookup needed because it's self-contained. **Authorization** then uses `[Authorize]` with role or **policy-based** checks against the claims. Access tokens are short-lived; a longer-lived **refresh token** gets new access tokens without re-login.

### 35. What logging and monitoring would you put in production?

**Answer:** **Structured logging** (e.g., Serilog) so logs are queryable JSON, not plain strings, shipped to a central store. Every request carries a **correlation ID** so I can trace one request's logs end to end. **Distributed tracing** via **OpenTelemetry** to follow requests across services. **Metrics** (request rates, latency percentiles, error rates, resource usage) on dashboards. **Health checks** wired to the orchestrator. And **alerting** on the things that matter — error spikes, latency SLO breaches, queue backlogs — so we find problems before users report them. The goal is the three pillars: logs, metrics, traces.

### 36. Trace a single request across multiple microservices?

**Answer:** I propagate a **correlation/trace ID** generated at the edge (or by the first service) and pass it in a header on every downstream call. Using **distributed tracing** (OpenTelemetry, the W3C `traceparent` standard), each service records spans tied to that trace ID, so a tool like Jaeger/Zipkin/Application Insights shows the full call tree with timings. All logs include the same ID, so log aggregation lets me pull every line for one request across all services. That's how you debug "where did the 3 seconds go" in a distributed flow.

### 37. What is CORS and when have you configured it?

**Answer:** **CORS (Cross-Origin Resource Sharing)** is a browser security mechanism. By the **same-origin policy**, a browser blocks JavaScript on `site-a.com` from calling an API on `api-b.com` unless the API explicitly allows it. I hit this whenever a frontend (e.g., a React app on a different domain/port) calls my API. The browser may send a **preflight `OPTIONS`** request; the API must respond with `Access-Control-Allow-Origin` and related headers. In ASP.NET Core I configure a CORS policy (allowed origins, methods, headers) and apply it in the middleware pipeline — scoping allowed origins tightly rather than using a wildcard in production.

---

## Section 7 — Scenario / Design Prompts (Stretch)

### 38. Design a file upload service for large files (up to 5 GB).

**Answer:** Don't stream multi-GB files through the app server — it ties up resources and risks timeouts. Instead:
- **Direct-to-storage with pre-signed URLs:** the client asks my API for a short-lived **pre-signed URL** (Azure Blob SAS / S3 presigned), then uploads **directly** to blob storage, bypassing my server.
- **Chunked / resumable uploads:** split the file into parts so a dropped connection resumes from the last chunk instead of restarting; storage SDKs support block/multipart uploads.
- After upload, blob storage fires an **event** that enqueues post-processing (virus scan, validation, thumbnail/transcode) done by a background worker.
- Track upload status in a DB so the UI can show progress and the user is notified on completion.

### 39. Design "recently viewed products" for a logged-in user.

**Answer:** This is per-user, frequently written, capped, and tolerant of slight staleness — perfect for **Redis**. I'd use a key per user like `recent:{userId}` as a **Redis list or sorted set** (sorted set scored by timestamp lets me keep order and dedupe). On each product view, push the product id and **trim to the last N** (say 20). Reads are a single fast Redis call. I'd set a **TTL** so inactive users' data expires. I don't need it in the relational DB — it's not critical data, and losing it is harmless, which is exactly why a fast cache store fits.

### 40. Import a 1-million-row CSV without timing out the request.

**Answer:** **Accept-and-queue.** The upload endpoint stores the file (blob storage), creates a job record with status `Pending`, enqueues a message, and **returns immediately** with a job id — no synchronous processing. A **background worker** then streams the file (don't load it all into memory), validates and processes in **batches** using **bulk insert** (`SqlBulkCopy` / EF Core bulk extensions) rather than row-by-row. It updates progress and handles **partial failures** — collect bad rows into an error report rather than failing the whole import. On completion it updates job status and **notifies** the user (SignalR/email). The user polls or gets pushed the status and can download an error report.

### 41. Design a rate-limited public API with API keys, usage tracking, and tiered limits.

**Answer:**
- **API keys:** issue a key per client, store a hashed version with its **tier** (free/pro/enterprise) and limits.
- **Auth middleware:** validate the key on each request, attach the tier to the request context.
- **Rate limiting:** per-key counters in **Redis** using a **sliding window** or token bucket; the tier determines bucket size. Exceed → **429** with `Retry-After`.
- **Usage tracking:** increment per-key counters (requests, maybe by endpoint) asynchronously — write to Redis and periodically flush aggregates to a DB for billing/analytics without slowing requests.
- **Extras:** quotas (monthly caps), a dashboard for clients to see usage, and key rotation/revocation.

### 42. Add full-text search to an existing product catalog.

**Answer:** It depends on requirements. For modest needs, **SQL Server Full-Text Search** is simplest — no new infrastructure, queries the existing DB. For real search (typo tolerance, relevance ranking, faceting, autocomplete, scale), I'd use a dedicated engine: **Elasticsearch/OpenSearch** or **Azure Cognitive Search**. The catch is **keeping the index in sync** with the DB: I'd push updates via an **event-driven pipeline** (publish a "product changed" event → indexer updates the search engine) or a periodic sync job, accepting eventual consistency. The search engine owns ranking/relevance; the DB stays the source of truth.

---

## How to Use This

- Don't recite these verbatim — **rephrase in your own words** and **anchor each to something you've built**. "We hit exactly this N+1 issue in our reporting module and fixed it with projection" beats a textbook definition every time.
- For scenario questions (38–42), **think out loud**: requirements → API → data model → scale → failure modes. The reasoning is what's scored.
- It's fine — and mature — to say *"I'd start simple and add complexity when metrics justify it."*

Want me to turn any of these into a deeper dive with diagrams (e.g., the URL shortener or the CSV import flow)?
