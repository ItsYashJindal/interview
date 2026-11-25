[Home](README.md) | [← 09 Kafka, Redis & Messaging](09-kafka-redis-messaging.md) | Next: [11 Scenario Questions →](11-scenario-questions.md)

# 10 – System Design Basics (added – asked in most 4+ year interviews)

At 4–5 years you won't usually get a full "design Twitter" round, but you will get "how would you design X" or "how will this scale" questions. Use the simple framework below.

## Contents
- [How to answer a design question (framework)](#how-to-answer-a-design-question)
- [Core concepts](#core-concepts)
- [Monolith vs microservices](#monolith-vs-microservices)
- [Patterns to know](#patterns-to-know)
- **Small designs**
  - [Design a URL shortener](#design-a-url-shortener)
  - [Design a notification system](#design-a-notification-system)
  - [Design a rate limiter](#design-a-rate-limiter)
  - [Design a chat system](#design-a-chat-system)
  - [Design a payment flow](#design-a-payment-flow)
  - [Design e-commerce checkout (stock without overselling)](#design-e-commerce-checkout)
  - [Design a "profiles near me" search](#design-a-profiles-near-me-search)

---

## How to answer a design question

> 🎤 **Interview answer:** "When I get a design question, I don't jump straight to the diagram. I start by clarifying requirements: who the users are, the expected scale in users and requests per second, whether it's read-heavy or write-heavy, and what must never fail, like payments. Then I list 3 or 4 core features, draw the high-level flow, client to load balancer to stateless API servers to cache and database, with a queue and workers for async work, then define the main data model with indexes and the key APIs. Finally I talk about scaling and failure handling: caching, replicas, retries, idempotency, monitoring. I explain the trade-offs of each choice as I go. Wherever possible I connect it to systems I've actually built."

1. **Ask questions (2 min):** Who uses it? How many users / requests per second? Read-heavy or write-heavy? What must never fail?
2. **List the main features** (keep 3–4).
3. **Draw the high-level flow:** Client → Load balancer → API servers → Cache → DB, + queue + workers.
4. **Design the data** – main tables/collections and indexes.
5. **Design the main APIs** – 3–4 endpoints.
6. **Handle scale and failure** – caching, queue, replicas, retries, idempotency.
7. **Mention trade-offs** – "I chose X because…, the downside is…".

---

## Core concepts

**Vertical vs horizontal scaling**

> 🎤 **Interview answer:** "Vertical scaling means giving one machine more CPU and RAM. It's simple and needs no code changes, but there's a hardware limit and it stays a single point of failure. Horizontal scaling means adding more machines behind a load balancer. It scales much further and gives redundancy, but it requires stateless application servers: sessions in Redis or JWT, files in S3, shared state like rate-limit counters in Redis. In practice I scale the app tier horizontally, with PM2 cluster plus multiple instances, and scale the database vertically first, then with read replicas."

- Vertical – bigger machine (more CPU/RAM). Simple, has a limit, single point of failure.
- Horizontal – more machines behind a load balancer. Needs **stateless** servers (sessions in Redis / JWT, files in S3).

**Stateless servers** – no user data kept in server memory between requests, so any server can handle any request.

> 🎤 **Interview answer:** "A stateless server doesn't keep any user-specific data in its memory between requests. Everything needed comes with the request, like the JWT, or lives in shared stores like Redis, the database or S3. Then any instance can handle any request, so we can add or remove instances freely, deploy without losing sessions, and survive instance crashes. Things like in-memory sessions, local file uploads or in-process caches of user state break this, so I avoid them."

**Load balancer** – spreads traffic, health checks (AWS ALB, Nginx).

**Caching layers** – browser → CDN → app memory → Redis → DB.

**Database scaling**
- Indexes and query fixes first.
- **Read replicas** – send reads to replicas, writes to primary (replica lag: data may be a few ms/sec old).
- **Sharding / partitioning** – split data across DBs by a key (userId).
- **Connection pooling**.

**CAP theorem (simple):** in a distributed system during a network split you must choose **Consistency** (everyone sees the same latest data, may reject requests) or **Availability** (always respond, data may be old). Payments → consistency; feeds/likes → availability.

> 🎤 **Interview answer:** "CAP says that in a distributed system, when a network partition happens, you have to choose between consistency, where every read sees the latest write, possibly by rejecting requests, and availability, where every request gets a response, possibly with stale data. Partitions can't be avoided, so the real choice is C or A during a failure. I pick based on the business: for payments, wallet balances and stock, I prefer consistency, because a wrong answer costs money. For feeds, likes, view counts or profile searches, availability with eventual consistency is fine."

**Consistency types:** strong (read sees latest write) vs eventual (will be correct after a short time – e.g. read replicas, caches, events).

> 🎤 **Interview answer:** "Strong consistency means a read always returns the latest committed write, like reading from the primary database. Eventual consistency means replicas, caches or downstream services may be stale for a short time but converge to the right value. Read replicas, Redis caches and event-driven updates through Kafka are all eventually consistent. So in designs I state which reads must be strongly consistent, like checking payment status, and route them to the primary, and which can tolerate slight staleness."

**Latency vs throughput:** latency = time for one request; throughput = requests handled per second.

**Single point of failure (SPOF):** any component that, if it fails, takes the whole system down. Remove with replicas, multiple instances, multi-AZ.

**Observability:** logs (what happened), metrics (how much / how fast – CloudWatch), traces (path of one request across services – request id / OpenTelemetry), alerts.

> 🎤 **Interview answer:** "Observability has three pillars. Logs tell me what happened: structured JSON logs with a request id. Metrics tell me how much and how fast: latency percentiles, error rates, CPU, consumer lag, in CloudWatch. Traces show the path of one request across services, using a correlation id or OpenTelemetry. On top of these, alerts tell us about problems before users do. At KindSwap, for example, we monitored the leader-election service in CloudWatch with alarms for no leader or error spikes."

---

## Monolith vs microservices

> 🎤 **Interview answer:** "A monolith is one codebase and one deployment. It's simpler to build, test, debug and deploy, transactions are easy with one database, and it's the right choice for small teams and early products. Microservices split the system into independently deployable services, each owning its data. Teams can deploy and scale independently, and failures can be isolated. But you pay with distributed complexity: network calls that fail, eventual consistency, distributed transactions that need sagas, and more monitoring and DevOps work.
>
> My honest view is to start with a well-structured modular monolith and extract a service when a part has a clearly different scaling need, team or release cycle. At KindSwap, Referral and Staking were separate NestJS services because they had their own domain logic, payout workflows and release cycles, and they communicated with events over Redis pub/sub."

| Monolith | Microservices |
|---|---|
| One codebase, one deploy | Many small services, each deployed separately |
| Simple to build, test, debug | Each team/service can scale and deploy independently |
| One DB, easy transactions | Each service owns its DB → distributed transactions are hard |
| Whole app scales together | Scale only the busy service |
| One bug can crash everything | Failure can be isolated (with timeouts/circuit breakers) |
| Good for small teams, early products | Good for large teams, clear domains |

**Honest answer:** start with a well-structured monolith (modules), split into services when a part has a different scaling need or team. At KindSwap, Referral and Staking were separate services because they had separate logic, payouts and release cycles.

**How services talk:**

> 🎤 **Interview answer:** "There are two styles. Synchronous calls, REST or gRPC, are simple and give an immediate answer, but the caller waits and fails if the callee is down, so you need timeouts, retries and circuit breakers. Asynchronous messaging with Kafka, RabbitMQ or Redis events decouples services in time: the producer doesn't wait or even know the consumers, but the result is eventually consistent. I use sync when the caller needs the answer right now, like validating a user, and async for side effects and cross-service workflows, like notifications or crediting referral rewards."

- **Sync** – REST/gRPC (simple, but caller waits and fails if callee is down).
- **Async** – Kafka/RabbitMQ/Redis events (decoupled, eventual consistency).

---

## Patterns to know

**Saga pattern** – a transaction across services done as a series of local steps; if a step fails, run **compensating** actions. Example: create order → reserve stock → take payment. Payment fails → release stock → cancel order.

> 🎤 **Interview answer:** "A saga handles a business transaction that spans multiple services, where we can't use one database transaction. It's broken into a sequence of local transactions, each in its own service, and if a step fails, compensating actions undo the previous steps. For example: create order, reserve stock, take payment. If payment fails, we release the stock and cancel the order. It can be choreographed, where services react to each other's events, or orchestrated, where a central coordinator tells each service what to do. Every step and compensation must be idempotent, since messages can be redelivered."

**Circuit breaker** – if a downstream service keeps failing, stop calling it for some time and return a fallback quickly (library: `opossum`). States: closed (normal) → open (fail fast) → half-open (test a few calls).

> 🎤 **Interview answer:** "A circuit breaker protects us from a failing dependency. Normally it's closed and calls go through. If failures or timeouts cross a threshold, it opens, and for a cool-down period calls fail immediately or return a fallback, instead of piling up and exhausting our connections. After that it goes half-open and lets a few test calls through. If they succeed it closes again, otherwise it reopens. In Node, the `opossum` library implements this. I'd use it around third-party calls like SMS providers or payment status APIs."

**Retry with exponential backoff + jitter** – retry failed calls after 1 s, 2 s, 4 s (+ random) – only for safe/idempotent operations.

> 🎤 **Interview answer:** "For transient failures, I retry with exponentially increasing delays, like 1, 2 and 4 seconds, plus random jitter so thousands of clients don't retry at exactly the same moment and overwhelm the recovering service. I cap the number of retries, only retry errors that can recover, like timeouts, 5xx or 429, never 400s, and only retry operations that are idempotent or protected by an idempotency key. After the final failure, the work goes to a DLQ or gets marked failed for follow-up."

**Idempotency** – see [07 → Idempotency](07-apis-backend.md#q-what-is-idempotency-and-how-do-you-implement-it).

**Outbox pattern** – see [09 → Outbox](09-kafka-redis-messaging.md#outbox-pattern).

**CQRS (simple)** – separate models for writes and reads (e.g. write to Postgres, read from a denormalised view/Elasticsearch).

> 🎤 **Interview answer:** "CQRS, Command Query Responsibility Segregation, means using separate models for writes and reads. Writes go to a normalised transactional store, and reads are served from a model optimised for queries, like a denormalised table, a cache or Elasticsearch, kept in sync through events. It helps when reads and writes have very different shapes or loads, like complex dashboards over transactional data. The cost is extra complexity and eventual consistency, so I'd only use it where it clearly pays off."

**Bulkhead** – separate pools/resources so one slow dependency doesn't use all connections.

> 🎤 **Interview answer:** "The bulkhead pattern, named after ship compartments, isolates resources so one failing part can't sink everything. For example, separate connection pools or worker queues per dependency, so if the SMS provider becomes slow, it can only use up its own pool, and payments and other APIs keep working."

**API Gateway / BFF** – one entry point; Backend-for-Frontend for different clients.

> 🎤 **Interview answer:** "An API gateway is the single entry point that handles routing, auth, rate limiting and logging for all services. A backend-for-frontend is a variation where each client type, like web, mobile or admin, gets its own thin backend layer that aggregates and shapes data exactly for that client, so the core services stay generic."

**Event sourcing (basic)** – store every change as an event and rebuild state from events.

> 🎤 **Interview answer:** "With event sourcing, instead of storing only the current state, we store every change as an immutable event, like deposited 500 or withdrew 200, and rebuild the current state by replaying the events. It gives a complete audit trail and the ability to rebuild read models, which is valuable for financial ledgers. The downsides are complexity, event schema versioning, and the need for snapshots to keep the rebuild fast."

---

## Design a URL shortener

> 🎤 **Interview answer:** "First I'd clarify the scale, but it's clearly very read-heavy: one write and many redirects. There are two APIs: POST `/shorten`, which returns a short code, and GET `/:code`, which redirects. For code generation, I'd take an auto-increment id, or a distributed id generator, and Base62-encode it. Seven characters give around 3.5 trillion combinations with no collisions. A random code with a uniqueness check also works. The table is code as the primary key, plus long URL, user, created and expiry dates.
>
> Since it's read-heavy, redirects hit Redis first, with the database as a fallback, and a CDN for very hot links. For analytics, I wouldn't write to the database on every click. I'd push click events to a queue and aggregate them asynchronously. On redirect type: 301 is permanent and cached by browsers, which means less load but lost analytics, while 302 sends every click through us, so if analytics matters I choose 302."

- **APIs:** `POST /shorten { url }` → `{ code }`; `GET /:code` → 301/302 redirect.
- **Code generation:** auto-increment id → Base62 encode (`a-zA-Z0-9`, 7 chars ≈ 3.5 trillion), or random + check for collision.
- **DB:** `urls(code PK, long_url, user_id, created_at, expires_at)`.
- **Read-heavy** → cache `code → url` in Redis; CDN for hot links.
- **Analytics:** send click events to a queue instead of writing to DB on every redirect.
- 301 (permanent, browser caches – fewer hits, less analytics) vs 302 (temporary – every click reaches you).

---

## Design a notification system

> 🎤 **Interview answer:** "This is close to what I built with Kafka at Appsinvo. Business services don't send notifications themselves. They publish events like `order.placed` or `payment.success`. A notification service consumes those, looks up the user's preferences, like channels and quiet hours, and the template, and pushes jobs onto channel-specific queues for email, SMS and push. That way a slow SMS provider doesn't delay emails. Channel workers call the providers, like SES, the SMS gateway or FCM, with retries, backoff and a dead-letter queue.
>
> Every notification is stored with a status, pending, sent or failed, for tracking, and de-duplicated by eventId so users never get the same message twice. I'd also rate-limit per user and send large campaigns in batches, so we don't hit provider limits."

(Close to what you built with Kafka.)
1. Services publish events: `order.placed`, `payment.success`.
2. **Notification service** consumes, looks up user preferences (email/SMS/push, quiet hours) and templates.
3. Pushes jobs to channel-specific queues: email, SMS, push.
4. Channel workers call providers (SES/SendGrid, SMS gateway, FCM) with **retries + DLQ**.
5. Store notification status (`pending/sent/failed`) for tracking and dedupe with `eventId`.
6. Rate-limit per user (don't send 50 SMS), batch large campaigns ([02 → chunk](02-javascript-coding.md#q11-chunk-an-array)).

---

## Design a rate limiter

> 🎤 **Interview answer:** "I'd put coarse limits at the edge, in Nginx or the API gateway, and fine-grained limits in application middleware. The key depends on the use case: IP for anonymous routes, userId or API key for authenticated ones, and often per route, with much stricter limits on login and OTP. For the algorithm I'd pick a token bucket, which allows short bursts, or a sliding window for accuracy. The state lives in Redis, so all instances share counters, using atomic INCR or a Lua script. When the limit is hit, we return 429 with a Retry-After header and rate-limit headers. One design decision to state clearly: if Redis is down, do we fail open and allow requests, keeping the site up, or fail closed? I'd fail open for general APIs and fail closed for sensitive ones like OTP."

- Where: API gateway / Nginx and/or middleware.
- Key: IP, userId, or API key + route.
- Algorithm: token bucket or sliding window ([03 → rate limiting](03-nodejs-express.md#10-api-rate-limiting)).
- Store: Redis (shared across servers, atomic `INCR`/Lua).
- Response: 429 + `Retry-After`, headers `X-RateLimit-Remaining`.
- If Redis is down: decide fail-open (allow, keep site up) or fail-closed (block, safer for login/OTP).

---

## Design a chat system

> 🎤 **Interview answer:** "This is close to the chat I built for the matrimony app. Clients connect over WebSockets with Socket.io, and the JWT is verified in the handshake middleware. Each user joins a room named after their user id. When a user sends a message, the server validates it, saves it in a messages collection first, and then emits it to the receiver's room, so nothing is lost if delivery fails. Messages live in their own collection, indexed on conversationId and createdAt, never inside a conversation document, and history loads with cursor pagination.
>
> To run multiple servers, I use the Socket.io Redis adapter, so an emit on one server reaches users connected to another, plus sticky sessions. For offline users, the message is already in the database, so we send a push notification and deliver on reconnect. Delivered and read receipts are ack events that update the status. Online presence and typing indicators go in Redis with short TTLs, not in the database."

(Close to your Socket.io + JWT chat.)
- **Connection:** WebSocket (Socket.io), JWT verified in the handshake.
- **Send:** client emits `message` → server validates → saves in DB (`messages: conversationId, senderId, text, createdAt, status`) → emits to the receiver's room (`userId` room).
- **Multiple servers:** Socket.io **Redis adapter** so a message reaches a user connected to another server; sticky sessions on the load balancer.
- **Offline users:** message is in DB; send push notification; deliver on reconnect.
- **Message status:** sent → delivered → read (ack events).
- **History:** cursor pagination by `createdAt` with index `{ conversationId: 1, createdAt: -1 }`.
- **Online status / typing:** Redis with TTL, not DB.
- **Scale:** messages in a separate collection, not inside conversation document.

---

## Design a payment flow

> 🎤 **Interview answer:** "This is based on what I built for 5 gateways. The client creates an order, which starts in `pending_payment`, with prices always calculated on the server. Then the client calls the payment endpoint with an Idempotency-Key. The backend creates an order on the gateway and returns the checkout details, and the user pays on the gateway's page, so card data never touches our servers.
>
> The source of truth is the webhook. We verify the HMAC signature, look up the payment by the gateway's payment id, and if it's already processed we just return 200, which makes it idempotent with a unique index. Otherwise we mark it paid, update the order and publish a `payment.success` event for invoices and notifications. The client's verify call after redirect is only for UX, never trusted alone. A reconciliation job checks payments stuck in pending against the gateway's status API, in case a webhook was missed.
>
> To support multiple gateways cleanly, I used a common PaymentGateway interface, with createOrder, verifyWebhook, refund and getStatus, and one adapter class per gateway, chosen by a factory based on country or currency. Amounts are stored as integers in the smallest unit."

(Close to your 5 gateways.)
1. Client → `POST /orders` → order created with `status = pending_payment`.
2. `POST /payments` with **Idempotency-Key** → backend creates a gateway order (Razorpay/Stripe) → returns gateway order id / checkout session to client.
3. User pays on gateway UI.
4. Gateway sends **webhook** → verify **signature** → find payment by gateway id → if already processed, ignore (idempotent) → mark `paid` → update order → publish `payment.success` event.
5. Client also calls a "verify" endpoint after redirect (don't trust only the client).
6. **Reconciliation job** – check pending payments older than X minutes with gateway's status API (in case webhook was missed).
7. **Gateway abstraction:** a common interface `createOrder / verifyWebhook / refund / getStatus` with one class per gateway (strategy/adapter pattern), chosen by country/currency/config.
8. Store amounts in smallest unit (paise/cents) as integers.

---

## Design e-commerce checkout

> 🎤 **Interview answer:** "The core problem is overselling: 10 units left and 50 people click buy at the same moment. The classic bug is reading the stock, checking it in code and then saving, which is a race condition. The simplest and most effective fix is an atomic conditional update: in MongoDB, `updateOne` with `stock >= qty` in the filter and `$inc` of minus qty; in SQL, `UPDATE ... SET stock = stock - qty WHERE id = ? AND stock >= qty`. If no row was modified, it's out of stock. For orders with multiple items, I wrap it in a transaction, or use `SELECT FOR UPDATE` row locks.
>
> For a better user experience during payment, I'd reserve stock for about 10 minutes and have a job release expired reservations. For extreme flash sales, Redis DECRBY can gate the requests quickly, with the database synced through a queue. I used the atomic update approach for stock recalculation on bookings at Byte Digital and in Rider Perfume."

**Problem:** 10 units left, 50 users click "buy" at the same time → don't oversell.
Options:
- **Atomic conditional update** (simplest, very effective):
  - Mongo: `updateOne({ _id, stock: { $gte: qty } }, { $inc: { stock: -qty } })` → if `modifiedCount === 0`, out of stock.
  - SQL: `UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1` → check affected rows.
- **Row lock** `SELECT ... FOR UPDATE` inside a transaction.
- **Reservation** – reserve stock for 10 minutes during payment; a job releases expired reservations.
- **Redis** `DECRBY` for flash sales, then sync to DB through a queue.

(In Byte Digital I did "stock recalculation on each booking" – same idea.)

---

## Design a "profiles near me" search

> 🎤 **Interview answer:** "This is what I built for the matrimony platform. Each profile stores its location as a GeoJSON Point, in longitude, latitude order, with a 2dsphere index. The search uses a `$geoNear` aggregation as the first stage, with the user's coordinates and a max distance, plus a query filter for preferences like gender, age range calculated from date of birth, and religion, excluding blocked users and the user themselves, then pagination. A compound index supports the common filters. For privacy, we never return other users' exact coordinates, only a rounded distance like '5 km away'. For performance, popular searches can be cached briefly and the page size is capped."

(Your matrimony project.)
- Store location as GeoJSON Point `[lng, lat]` with a **2dsphere index**.
- Query with `$near` / `$geoNear` + filters (gender, age range from dob, religion) + pagination.
- Compound index on common filters; `$geoNear` must be first in aggregation and can take a `query` filter.
- Don't return exact coordinates of other users (privacy) – return distance rounded ("5 km away").
- Cache popular searches briefly; limit results per page.

---

[Home](README.md) | [← 09 Kafka, Redis & Messaging](09-kafka-redis-messaging.md) | Next: [11 Scenario Questions →](11-scenario-questions.md)
