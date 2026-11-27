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

> 🎤 **Interview answer:** "I don't jump straight to drawing. I follow simple steps:
> 1. **Ask questions** – who uses it, how many users, is it more reads or more writes, and what must never fail (like payments).
> 2. **Pick 3–4 main features.**
> 3. **Draw the basic flow** – app → load balancer → API servers → cache → database, plus a queue and workers for background work.
> 4. **Design the data** – main tables and indexes.
> 5. **Design the main APIs.**
> 6. **Talk about scale and failures** – caching, more servers, retries, duplicates, monitoring.
>
> At each step I say why I chose something and what the downside is. And I connect it to systems I've actually built."

```
Client → Load balancer → API servers (many, stateless) → Redis cache → Database
                                   ↓
                                 Queue → Workers (emails, invoices, etc.)
```

---

## Core concepts

**Vertical vs horizontal scaling**

> 🎤 **Interview answer:**
> - "**Vertical scaling** = make one server bigger (more CPU and RAM). Simple, no code change. But there's a limit, and if that one server dies, everything is down.
> - **Horizontal scaling** = add **more servers** behind a load balancer. Can grow much more, and if one dies, others keep working. But the servers must be **stateless**: no user data saved in a server's memory.
>
> In practice, I scale the app by adding servers, and scale the database by making it bigger first, then adding read copies."

**Stateless servers**

> 🎤 **Interview answer:** "A stateless server **doesn't remember anything about a user** between requests. Everything needed comes with the request (like the JWT), or is stored in a shared place (Redis, DB, S3). So **any server can handle any request**. We can add or remove servers freely, and if one crashes, users don't lose anything. Keeping sessions or uploaded files on one server's disk breaks this."

**Load balancer** – spreads traffic across servers and checks their health (AWS ALB, Nginx).

**Caching layers** – browser → CDN → app memory → Redis → DB.

**Database scaling**
- Fix indexes and slow queries first.
- **Read replicas** – copies of the DB used only for reading. Writes go to the main DB. (Copies may be a few ms/seconds behind.)
- **Sharding** – split data across many DBs by a key (like userId).
- **Connection pooling.**

**CAP theorem (simple):**

> 🎤 **Interview answer:** "CAP says: when the network between servers breaks, a system must choose between:
> - **Consistency** – everyone sees the latest correct data, even if that means saying 'try again later'.
> - **Availability** – always give an answer, even if the data might be a bit old.
>
> I choose based on the business. For payments, wallet and stock → **consistency**, because a wrong answer costs money. For feeds, likes and view counts → **availability** is fine; slightly old data is OK."

**Consistency types:**

> 🎤 **Interview answer:**
> - "**Strong consistency** – a read always gets the latest data. Like reading from the main database.
> - **Eventual consistency** – the data may be old for a short time, but becomes correct soon. Read copies, caches and Kafka events work like this.
>
> In a design, I say clearly which reads must be fresh, like payment status, and send those to the main DB."

**Latency vs throughput:** latency = time for **one** request. Throughput = **how many** requests per second.

**Single point of failure (SPOF):** any part that, if it fails, takes the whole system down. Fix: more copies, more servers, multiple data centres.

**Observability:**

> 🎤 **Interview answer:** "Observability means being able to see what's happening inside the system. Three parts:
> - **Logs** – what happened. JSON logs with a request id.
> - **Metrics** – numbers: how fast, how many errors, CPU, consumer lag. In CloudWatch.
> - **Traces** – the path of **one** request across many services, using a request id.
>
> Plus **alerts**, so we find problems before users do. At KindSwap we watched the leader-election service in CloudWatch, with alerts for 'no leader' and error spikes."

---

## Monolith vs microservices

> 🎤 **Interview answer:** "A **monolith** is one codebase, deployed as one app. Simple to build, test and deploy, and transactions are easy with one database. Right for small teams and new products.
>
> **Microservices** split the system into **many small apps**, each with its own database, each deployed separately. Teams can work and deploy independently, and scale only the busy part. But it's much more complex: network calls can fail, data is only 'eventually' in sync, and you need more monitoring and DevOps.
>
> My honest view: start with a **well-organised monolith**, and split out a service only when one part clearly needs it, like a different scaling need or a separate team. At KindSwap, Referral and Staking were separate NestJS services because they had their own logic, payout flows and release cycles, and they talked through events on Redis pub/sub."

| Monolith | Microservices |
|---|---|
| One codebase, one deploy | Many small services, each deployed separately |
| Simple to build, test, debug | Each service can scale and deploy on its own |
| One DB, easy transactions | Each service has its own DB → cross-service transactions are hard |
| Whole app scales together | Scale only the busy service |
| One bug can crash everything | A failure can stay inside one service |
| Good for small teams, new products | Good for big teams |

**How services talk:**

> 🎤 **Interview answer:**
> - "**Sync (REST / gRPC)** – service A calls B and **waits** for the answer. Simple, but if B is down, A fails too. So I add timeouts, retries and a circuit breaker.
> - **Async (Kafka / RabbitMQ / Redis events)** – A sends a message and **doesn't wait**. B handles it later. Services don't depend on each other, but the result comes a bit later.
>
> I use sync when I need the answer right now, like checking a user. I use async for background work, like notifications or giving referral rewards."

---

## Patterns to know

**Saga pattern**

> 🎤 **Interview answer:** "A saga is used when one business action needs **many services**, so we can't use one database transaction. Each service does its own step. If a later step fails, we run **undo steps** for the earlier ones.
>
> Example: create order → reserve stock → take payment. If payment fails → release the stock → cancel the order.
>
> Every step and every undo step must be safe to run twice, because messages can repeat."

**Circuit breaker**

> 🎤 **Interview answer:** "A circuit breaker protects us from a service that keeps failing. It works like an electric fuse:
> - **Closed** (normal) – calls go through.
> - If too many calls fail, it **opens** – for some time, we stop calling that service and return a fallback answer immediately. So requests don't pile up waiting.
> - After a while, it goes **half-open** – lets a few test calls through. If they work, it closes again. If not, it opens again.
>
> In Node, the `opossum` library does this. I'd use it around third-party calls like the SMS provider or payment status API."

**Retry with exponential backoff + jitter**

> 🎤 **Interview answer:** "When a call fails for a temporary reason, I retry it, waiting longer each time: 1 s, 2 s, 4 s. I also add a small **random** extra wait (jitter), so thousands of clients don't all retry at the exact same moment. I limit the number of retries, only retry temporary errors (timeouts, 5xx, 429), never 400s, and only retry actions that are safe to repeat. After the last failure, the job goes to a DLQ."

**Idempotency** – see [07 → Idempotency](07-apis-backend.md#q-what-is-idempotency-and-how-do-you-implement-it).

**Outbox pattern** – see [09 → Outbox](09-kafka-redis-messaging.md#outbox-pattern).

**CQRS (simple)**

> 🎤 **Interview answer:** "CQRS means using **one model for writing and a different model for reading**. Writes go to the normal database. Reads come from a copy designed for fast reading, like a summary table, a cache or Elasticsearch, kept updated through events. It's useful when reads are very different from writes, like heavy dashboards. But it adds complexity, so I'd use it only when clearly needed."

**Bulkhead**

> 🎤 **Interview answer:** "Bulkhead means **keeping resources separate**, like the separate compartments in a ship, so one leak doesn't sink everything. For example, a separate connection pool for the SMS provider. If SMS becomes slow, it can only use up its own pool, and payments and other APIs keep working."

**API Gateway / BFF**

> 🎤 **Interview answer:** "An API gateway is one front door for all services: routing, auth, rate limiting, logging. A **BFF (Backend-for-Frontend)** is a small backend made for one client type, like mobile or admin, that collects and shapes data exactly for that client."

**Event sourcing (basic)**

> 🎤 **Interview answer:** "With event sourcing, instead of saving only the current value, we save **every change** as an event, like 'deposited 500', 'withdrew 200'. The current balance is calculated by replaying all events. It gives a full history, which is great for money ledgers. But it's complex to build."

---

## Design a URL shortener

> 🎤 **Interview answer:** "It's very **read-heavy**: a link is created once but opened many times.
>
> **APIs:**
> - `POST /shorten` – takes a long URL, returns a short code.
> - `GET /:code` – redirects to the long URL.
>
> **Making the code:** take an auto-increment id and convert it to Base62 (a–z, A–Z, 0–9). 7 characters give about 3.5 trillion codes, with no clashes. A random code with a 'does it exist?' check also works.
>
> **Database:** a table with code (primary key), long URL, user, created date and expiry.
>
> **Speed:** since it's read-heavy, I check Redis first for the code, then the DB. Very popular links can be served by a CDN.
>
> **Click counts:** I don't write to the DB on every click. I send click events to a queue and count them in the background.
>
> **301 vs 302:** 301 is permanent and the browser caches it, so less load but we lose click tracking. 302 sends every click through us. If tracking matters, I use 302."

---

## Design a notification system

> 🎤 **Interview answer:** "This is close to what I built with Kafka at Appsinvo.
> 1. Business services **don't send notifications themselves**. They just send events like `order.placed` or `payment.success`.
> 2. A **notification service** reads these events, checks the user's settings (email / SMS / push, quiet hours) and picks the template.
> 3. It puts jobs into **separate queues per channel**: email, SMS, push. So a slow SMS provider doesn't delay emails.
> 4. Workers for each channel call the providers (SES, SMS gateway, FCM), with **retries and a dead-letter queue**.
> 5. Every notification is saved with a status (pending / sent / failed), and checked by `eventId` so nobody gets the same message twice.
> 6. I limit how many messages one user can get, and send big campaigns in batches."

Batch sending: see [02 → chunk](02-javascript-coding.md#q11-chunk-an-array).

---

## Design a rate limiter

> 🎤 **Interview answer:**
> - "**Where:** basic limits in Nginx or the API gateway, detailed limits in app middleware.
> - **Key (who to count):** IP for logged-out routes, userId or API key for logged-in routes. Much stricter limits on login and OTP.
> - **Algorithm:** token bucket (allows small bursts) or sliding window (more accurate).
> - **Storage:** Redis, so all servers share the same counts. `INCR` is safe even with many requests at once.
> - **Response:** 429 with a `Retry-After` header.
> - **If Redis is down:** decide to **allow** requests (keep the site up) or **block** them (safer). I'd allow for normal APIs, and block for sensitive ones like OTP."

Details: [03 → rate limiting](03-nodejs-express.md#10-api-rate-limiting).

---

## Design a chat system

> 🎤 **Interview answer:** "This is close to the chat I built for the matrimony app.
> - **Connection:** users connect with WebSockets (Socket.io). I check the JWT when they connect. Each user joins a room named after their user id.
> - **Sending:** when a user sends a message, the server checks it, **saves it in the DB first**, then sends it to the receiver's room. So nothing is lost even if delivery fails.
> - **Storage:** messages go in their own collection (not inside a conversation document), with an index on `conversationId + createdAt`. Old messages load page by page with cursor pagination.
> - **Many servers:** the Socket.io **Redis adapter** passes messages between servers, so a user on server A gets a message sent from server B. Plus sticky sessions.
> - **Offline users:** the message is already saved, so we send a push notification and deliver it when they come back online.
> - **Delivered / read ticks:** the app sends back 'delivered' and 'read' events, and we update the status.
> - **Online status and typing:** kept in Redis with a short expiry, not in the DB."

---

## Design a payment flow

> 🎤 **Interview answer:** "This is based on what I built for 5 payment gateways.
> 1. The user creates an order. Its status is `pending_payment`. **Prices are always calculated on the server**, never trusted from the app.
> 2. The app calls our payment API with an **Idempotency-Key**. We create an order on the gateway (Razorpay / Stripe) and return the checkout details.
> 3. The user pays on the **gateway's page**, so card details never touch our servers.
> 4. The gateway sends a **webhook**. This is our **source of truth**. We check the signature, find the payment by the gateway's payment id, and if it's already processed, we just return 200 (so duplicates are safe). Otherwise we mark it paid, update the order, and send a `payment.success` event for the invoice and notifications.
> 5. The app's 'verify' call after payment is only for showing the result. We never trust it alone.
> 6. A **background job** checks payments stuck in 'pending' with the gateway's status API, in case a webhook was missed.
>
> To support many gateways cleanly, I made a common `PaymentGateway` interface with `createOrder`, `verifyWebhook`, `refund` and `getStatus`, and one class per gateway. A factory picks the right one based on country or currency. Amounts are saved as whole numbers in paise / cents."

---

## Design e-commerce checkout

> 🎤 **Interview answer:** "The main problem is **overselling**: 10 items left, and 50 people click 'buy' at the same moment.
>
> The common bug: read the stock, check it in code, then save. Two requests can read '10' at the same time and both sell. This is a race condition.
>
> The simplest and best fix: **check and reduce in one single database operation**:
> - MongoDB: `updateOne({ _id, stock: { $gte: qty } }, { $inc: { stock: -qty } })`.
> - SQL: `UPDATE products SET stock = stock - qty WHERE id = ? AND stock >= qty`.
>
> If nothing was updated, it's out of stock. The database makes sure only the right number of buyers succeed.
>
> For orders with many items, I use a transaction, or lock the rows with `SELECT ... FOR UPDATE`. For a better experience during payment, I can **reserve** stock for 10 minutes, and a job releases expired reservations. For huge flash sales, Redis `DECRBY` can handle the rush first, and the DB is updated through a queue. I used the single-update approach in Byte Digital and Rider Perfume."

```js
// MongoDB – check and reduce in one step
const result = await Product.updateOne(
  { _id: productId, stock: { $gte: qty } },  // only if enough stock
  { $inc: { stock: -qty } }                  // reduce it
);
if (result.modifiedCount === 0) throw new AppError("Out of stock", 409);
```

---

## Design a "profiles near me" search

> 🎤 **Interview answer:** "This is what I built for the matrimony platform.
> - Each profile saves its location as a point `[longitude, latitude]`, with a `2dsphere` index.
> - The search uses `$geoNear` with the user's location and a max distance, plus filters like gender, age range (calculated from date of birth) and religion. It skips blocked users and the user themselves, and returns results page by page.
> - A compound index supports the common filters.
> - **Privacy:** we never return other users' exact location, only a rounded distance like '5 km away'.
> - **Speed:** popular searches can be cached for a short time, and the page size is limited."

---

[Home](README.md) | [← 09 Kafka, Redis & Messaging](09-kafka-redis-messaging.md) | Next: [11 Scenario Questions →](11-scenario-questions.md)
