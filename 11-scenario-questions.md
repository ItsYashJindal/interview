[Home](README.md) | [← 10 System Design](10-system-design.md) | Next: [12 Project Questions →](12-project-questions.md)

# 11 – Real-World Scenario Questions

These separate someone with real experience from someone who only studied. **Always add one line from your own work** ("In my Appsinvo project, I…").

Answer pattern: **Understand → Measure → Fix → Prevent**.

## Contents
**Your list**
1. [Handle 10,000 API requests simultaneously](#1-how-would-you-handle-10000-api-requests-simultaneously)
2. [Improve performance of a slow Node.js API](#2-how-do-you-improve-the-performance-of-a-slow-nodejs-api)
3. [Prevent duplicate API requests](#3-how-do-you-prevent-duplicate-api-requests)
4. [Implement JWT authentication](#4-how-would-you-implement-jwt-authentication)
5. [Handle errors in Express.js](#5-how-do-you-handle-errors-in-expressjs)
6. [Optimise a slow MongoDB query](#6-how-would-you-optimise-a-slow-mongodb-query)
7. [Manage database transactions](#7-how-do-you-manage-database-transactions)
8. [Pagination for millions of records](#8-how-would-you-implement-pagination-for-millions-of-records)
9. [Secure an API against SQL/NoSQL injection](#9-how-do-you-secure-an-api-against-sql-injection-and-nosql-injection)
10. [Debug a production issue](#10-how-would-you-debug-a-production-issue)
11. [Async function throws an error](#11-what-happens-when-an-async-function-throws-an-error)
12. [Integrate a third-party API](#12-how-would-you-integrate-a-third-party-api)

**Added (commonly asked)**
- [E1. Server CPU is at 100%](#e1-server-cpu-is-at-100)
- [E2. Memory keeps increasing (memory leak)](#e2-memory-keeps-increasing)
- [E3. Cron job runs on every instance](#e3-cron-job-runs-on-every-instance)
- [E4. Send 1 lakh emails / notifications](#e4-send-1-lakh-emails-or-notifications)
- [E5. Upload large files](#e5-upload-large-files)
- [E6. Third-party API is slow or down](#e6-third-party-api-is-slow-or-down)
- [E7. Two users update the same record (race condition)](#e7-two-users-update-the-same-record)
- [E8. Zero-downtime deployment](#e8-zero-downtime-deployment)
- [E9. Redis goes down](#e9-redis-goes-down)
- [E10. Long-running task (report generation)](#e10-long-running-task-like-report-generation)
- [E11. Export lakhs of rows to CSV](#e11-export-lakhs-of-rows-to-csv)
- [E12. Data sync between two systems fails midway](#e12-data-sync-between-two-systems-fails-midway)
- [E13. Someone is brute-forcing the login](#e13-someone-is-brute-forcing-the-login)
- [E14. Works locally, fails in production](#e14-works-locally-fails-in-production)
- [E15. Logout from all devices](#e15-logout-from-all-devices)
- [E16. Database goes down](#e16-database-goes-down)
- [E17. Handling time zones](#e17-handling-time-zones)
- [E18. Code review – what do you check?](#e18-code-review--what-do-you-check)

---

### 1. How would you handle 10,000 API requests simultaneously?

> 🎤 **Interview answer:** "First I'd clarify whether it's 10,000 requests per second or 10,000 concurrent connections, and whether they're mostly reads or writes, because the answer changes. Node itself is good at concurrency because I/O is non-blocking, so the first check is that no route blocks the event loop with heavy CPU or sync work.
>
> Then I scale in layers. Use all CPU cores with PM2 cluster mode, and scale horizontally with multiple stateless instances behind a load balancer, with JWT auth, sessions in Redis and files in S3. Cache hot reads in Redis, and in a CDN for public data, so most requests never reach the database. On the database side: proper indexes, a correctly sized connection pool and read replicas. Anything slow, like emails or invoices, goes to a queue, and the API returns immediately. Then I protect the system with rate limiting, payload limits, timeouts and circuit breakers, set up auto-scaling, load-test with k6 or autocannon beforehand, and monitor p95 and p99 latency, error rate and event loop lag.
>
> From my experience at Appsinvo, moving notifications and order receipts to Kafka took them off the request path, and indexes plus caching kept the database load low under traffic."

First ask: 10,000 per second or 10,000 open at once? Reads or writes?

1. **Node handles concurrency well** because I/O is non-blocking – make sure no route does heavy CPU work or sync calls on the main thread.
2. **Use all CPU cores** – PM2 cluster mode / multiple containers.
3. **Scale horizontally** – more instances behind a load balancer (Nginx/ALB). Keep servers stateless (JWT, sessions in Redis, files in S3).
4. **Cache** frequent reads in Redis (and CDN for static/public data) so most requests don't hit the DB.
5. **Database** – proper indexes, connection pool sized correctly, read replicas for reads.
6. **Move slow work to a queue** (Kafka/SQS) – email, invoice, notifications – return quickly (201/202).
7. **Protect the system** – rate limiting, request body size limits, timeouts on every outgoing call, circuit breaker.
8. **Auto-scaling** based on CPU/request count; **load test** before (k6, Artillery, autocannon).
9. **Monitor** latency (p95/p99), error rate, event loop lag, DB connections.

*From my work:* moving notifications and order receipts to Kafka took them off the request path, and caching + indexes kept DB load low.

---

### 2. How do you improve the performance of a slow Node.js API?

> 🎤 **Interview answer:** "My rule is measure first, then fix the biggest bottleneck. I add timing logs for each step or use APM to see where the time goes: database, external calls or CPU. Most often it's the database, so I run `explain`, add the right compound indexes for the filters and sort, use projection and `.lean()` to fetch only the needed fields, remove N+1 queries, and push filtering and pagination into the query instead of doing it in JavaScript. Then I parallelise independent calls with `Promise.all`, cache data that doesn't change often in Redis, reduce the payload with gzip, move heavy work to queues or worker threads, add timeouts to slow third parties, and check for event loop blockers like a huge JSON.parse. Then I measure again.
>
> A real example: at Appsinvo, an admin user-listing API took about 4.5 seconds. With explain I found it was doing a collection scan, returning full user documents, and filtering some data in Node after loading it. I added compound indexes matching the filters and sort, returned only the fields the list needed with lean, and moved all filtering and pagination into the query. It came down to about 550 milliseconds, roughly 88% faster."

**Measure first, then fix the biggest problem.**
1. **Find where time goes** – logs with timing per step, APM (New Relic/Datadog/CloudWatch), `console.time`, DB slow query log, `explain()`.
2. **Database (most common cause)**
   - Add correct indexes (compound for filter + sort).
   - Select only needed fields (projection), `.lean()` in Mongoose.
   - Remove N+1 queries (use `$in`, JOIN, populate wisely).
   - Paginate, and do filtering in the DB, not in JS.
3. **Run independent calls in parallel** – `Promise.all` instead of sequential awaits.
4. **Cache** results that don't change often (Redis).
5. **Reduce payload** – fewer fields, gzip compression.
6. **Move heavy work off the request** – queue, worker threads for CPU tasks.
7. **Check external calls** – slow third-party APIs → timeouts, caching, async.
8. **Check event loop blocking** – big `JSON.parse`, sync crypto, large loops.
9. Re-measure and compare.

*From my work:* user-listing API went from **4.5 s to 550 ms** – added compound indexes for the filter fields, returned only required fields (smaller payload), and moved filtering into the query. See [13 Resume](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).

---

### 3. How do you prevent duplicate API requests?

> 🎤 **Interview answer:** "Duplicates come from double clicks, network retries or client bugs. On the frontend we disable the button and debounce, but that's only UX. The real protection has to be on the backend.
>
> The main tool is an idempotency key: the client sends a unique `Idempotency-Key` header for each logical action. The server does an atomic `SET key processing NX` in Redis with a TTL. If the key is new, we process the request and store the response against the key. If the key already has a response, we return that same response. If it's still processing, we return 409. As a second layer, I put unique constraints in the database, like a unique gateway payment id or user plus order reference, so even if two requests slip through, the second insert fails safely. For short critical sections, like applying a coupon, I use a Redis lock, and for webhooks and events, I store processed event ids.
>
> I used exactly this approach across the 5 payment gateways at Appsinvo, so retries never caused double payments."

Duplicates come from double clicks, network retries, or client bugs.

**Frontend:** disable the button after click, debounce.
**Backend (the real protection):**
1. **Idempotency key** – client sends a unique `Idempotency-Key` header per action. Server stores it in Redis with `SET key "processing" NX EX 86400`:
   - Key is new → process, store the response.
   - Key exists with a response → return the saved response.
   - Key exists, still processing → 409.
2. **Unique constraints in DB** – unique index on `(userId, orderRef)` or gateway `paymentId` → second insert fails safely.
3. **Distributed lock** for a short critical section (e.g. "apply coupon for this user").
4. **Webhook/event dedupe** – store processed `eventId`s.
5. **Rate limit** sensitive endpoints.

*From my work:* used idempotency keys across 5 payment gateways so retries didn't cause double payments.

---

### 4. How would you implement JWT authentication?

> 🎤 **Interview answer:** "On registration I validate the input, hash the password with bcrypt and save the user. On login I verify with `bcrypt.compare` and issue two tokens: a short-lived access token, around 15 minutes, with the user id and role, and a long-lived refresh token, around 7 days, whose hash I store in the database or Redis along with device info. The refresh token goes in an httpOnly, Secure, SameSite cookie, and the access token is returned in the body and kept in memory by the client.
>
> An auth middleware reads the Bearer token, verifies it, sets `req.user`, or returns 401, and an authorization middleware checks roles and returns 403. The refresh endpoint validates the refresh token against the database, issues a new access token and rotates the refresh token. If an old refresh token is reused, I treat it as theft and revoke all sessions. Logout deletes the refresh token, and for immediate revocation I can blacklist the access token's jti in Redis until it expires. Plus the basics: secrets in a secrets manager, rate-limited login, no sensitive data in the payload, and HTTPS only."

1. **Register** – validate input → hash password with bcrypt → save user.
2. **Login** – find user → `bcrypt.compare` → create:
   - **Access token** (15 min) with `sub: userId, role`.
   - **Refresh token** (7 days) – store its hash in DB/Redis with device info.
3. Send refresh token in **httpOnly, Secure, SameSite** cookie; access token in response body (kept in memory by the client).
4. **Auth middleware** – read `Authorization: Bearer <token>`, `jwt.verify`, set `req.user`, else 401.
5. **Authorization middleware** – check role/permission, else 403.
6. **Refresh endpoint** – verify refresh token + check it exists in DB and not revoked → issue new access token + **rotate** refresh token. If a used refresh token is reused → revoke all sessions (theft).
7. **Logout** – delete refresh token; optionally blacklist access token `jti` in Redis until it expires.
8. **Security** – strong secrets in env/Secrets Manager, rate-limit login, don't put sensitive data in payload, HTTPS only.

Code in [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens).

---

### 5. How do you handle errors in Express.js?

> 🎤 **Interview answer:** "I centralise it. Services throw a custom `AppError` with a status code and an `isOperational` flag. Async route handlers are wrapped with an asyncHandler, or we use Express 5, so rejected promises reach the error middleware. Validation middleware with zod or Joi returns clean 400s. A 404 handler catches unknown routes. Then one global error middleware at the end logs the full error with the request id and returns a consistent JSON response, hiding stack traces and internal messages in production.
>
> I also map known library errors: a Mongo duplicate key error, code 11000, becomes 409; a CastError becomes 400; a JWT TokenExpiredError becomes 401. At process level, I handle `unhandledRejection` and `uncaughtException` by logging and letting PM2 restart the process cleanly. And we alert on error-rate spikes."

1. Custom `AppError` class with `statusCode` and `isOperational`.
2. Async wrapper (`asyncHandler`) or Express 5 so rejected promises reach the error handler.
3. Validation middleware (Joi/zod/class-validator) → 400 with clear messages.
4. 404 handler for unknown routes.
5. **One global error middleware** `(err, req, res, next)` at the end – logs full error with request id, sends a clean response, hides stack traces in production.
6. Map known library errors: Mongo duplicate key `11000` → 409, `CastError` → 400, JWT `TokenExpiredError` → 401.
7. `process.on("unhandledRejection")` and `uncaughtException` → log and restart (PM2).
8. Alerts on error-rate spikes.

Code in [03 → Error handling](03-nodejs-express.md#5-error-handling).

---

### 6. How would you optimise a slow MongoDB query?

> 🎤 **Interview answer:** "First I identify it, with the MongoDB profiler using a slowms threshold, the Atlas performance advisor, or our logs. Then I run `explain('executionStats')` and look for COLLSCAN, a big gap between docs examined and docs returned, or an in-memory SORT stage. The usual fix is a compound index following the ESR rule, equality then sort then range, plus projection and lean to return less data. I also avoid patterns that can't use indexes well, like unanchored regex, `$where`, and `$ne` or `$nin` on large sets. In aggregations, I put `$match` and `$sort` first, project early and limit before `$lookup`. For deep pages I switch from skip to cursor pagination. If the query is still heavy, I reconsider the schema, embed data that's read together, or store computed counters, and cache frequent results in Redis. Finally, I check that the working set and indexes fit in RAM. That's exactly the process I followed for the 4.5-second-to-550-millisecond optimisation."

1. Find it – MongoDB profiler (`db.setProfilingLevel(1, { slowms: 100 })`), Atlas performance advisor, logs.
2. `explain("executionStats")` – check `COLLSCAN` vs `IXSCAN`, `totalDocsExamined` vs `nReturned`, in-memory `SORT`.
3. Add / fix index using the **ESR rule** (Equality, Sort, Range).
4. Projection – return only needed fields; `.lean()`.
5. Avoid `$regex` without a prefix anchor, `$where`, `$nin`/`$ne` on large sets – they can't use indexes well.
6. Aggregation – `$match` and `$sort` first, `$project` early, `$limit` before `$lookup` when possible.
7. Replace `skip` on deep pages with cursor pagination.
8. Fix schema if needed – embed data read together; store computed counters instead of counting each time.
9. Cache results in Redis if the query is frequent.
10. Check server: working set fits in RAM, index sizes.

---

### 7. How do you manage database transactions?

> 🎤 **Interview answer:** "I use a transaction only when multiple writes must succeed or fail together, like a money transfer, order plus stock plus payment record, or in the Air India project, creating a GRN and updating the received quantity on the purchase order. In SQL that's BEGIN and COMMIT, or Prisma's `$transaction`, which is what I used there. In MongoDB, it's a session with `withTransaction`, which needs a replica set. But since single-document updates are already atomic in MongoDB, I design to use those where possible.
>
> My rules: keep transactions short, never call external APIs inside them, because locks are held while waiting. Handle concurrency with `SELECT FOR UPDATE` or optimistic locking with a version column. Retry on deadlocks and transient write conflicts. And across microservices, where one database transaction isn't possible, I use a saga with compensating actions, plus the outbox pattern for reliable events."

1. Use a transaction only when **multiple writes must succeed or fail together** (money transfer, order + stock + payment record, GRN + PO update).
2. SQL: `BEGIN … COMMIT / ROLLBACK`; Prisma `$transaction`, TypeORM `dataSource.transaction`.
3. MongoDB: sessions + `withTransaction` (needs replica set). Single-document updates are already atomic, so design to use them when possible.
4. **Keep transactions short** – no external API calls inside (locks are held while waiting).
5. Choose isolation level; use `SELECT … FOR UPDATE` or optimistic locking (`version` column) for concurrent updates.
6. Handle **deadlocks / write conflicts with retry** (Mongo `withTransaction` retries transient errors).
7. **Across services** you can't use one DB transaction → **Saga** with compensating actions + **Outbox** for events ([10 → Patterns](10-system-design.md#patterns-to-know)).

Code in [04 → Transactions](04-mongodb-sql.md#8-transactions).

---

### 8. How would you implement pagination for millions of records?

> 🎤 **Interview answer:** "I wouldn't use offset or skip for this, because the database still has to read and throw away all the skipped rows, so deep pages become very slow, and results shift when new data arrives. I'd use cursor, or keyset, pagination: sort by `created_at` and `id` together so the order is unique, and for the next page ask for rows less than the last seen `(created_at, id)` with a limit. With an index matching that sort, every page is equally fast, even page 50,000.
>
> The API returns a `nextCursor`, base64-encoded, and `hasMore`, which I get by fetching limit plus one. I cap the limit at about 100. Exact total counts on millions of rows are expensive, so I cache them, show an estimate, or use a 'load more' UI. If the use case is complex search with many filters, I'd bring in Elasticsearch or OpenSearch. And if the actual need is an export, I don't paginate at all. I stream the data."

1. **Don't use `OFFSET/skip` for deep pages** – the DB still reads all skipped rows; page 50,000 becomes very slow.
2. Use **cursor (keyset) pagination**: `WHERE (created_at, id) < (:lastCreatedAt, :lastId) ORDER BY created_at DESC, id DESC LIMIT 20`.
3. Add an **index** matching the sort (`created_at DESC, id DESC`) plus filter columns.
4. Return `nextCursor` (encode as base64 string) and `hasMore` (fetch `limit + 1`).
5. Set a **max limit** (e.g. 100).
6. **Total count** is expensive – cache it, show approximate count, or skip it ("load more").
7. For search with many filters on huge data → a search engine (Elasticsearch/OpenSearch).
8. For exports, don't paginate the API – **stream** the data ([E11](#e11-export-lakhs-of-rows-to-csv)).

Code in [07 → Pagination](07-apis-backend.md#4-pagination).

---

### 9. How do you secure an API against SQL injection and NoSQL injection?

> 🎤 **Interview answer:** "SQL injection happens when user input is concatenated into a query string, so input like `' OR 1=1 --` changes the query logic. The fix is always parameterised queries, which ORMs like Prisma and TypeORM do by default. The risk comes back with raw queries, so I'm careful with things like `$queryRawUnsafe`.
>
> NoSQL injection in MongoDB is different: the attacker sends an object instead of a string, like `password: { $ne: null }`, and if we pass `req.body` straight into `findOne`, it matches without the real password. The fixes are strict type validation with zod, Joi or class-validator DTOs, so email and password must be strings, sanitising keys that start with `$` using express-mongo-sanitize or Mongoose's `sanitizeFilter`, never passing raw request objects into queries, and avoiding `$where`.
>
> On top of that: a least-privilege database user, whitelisting allowed sort and filter fields, and error messages that never expose the query."

**SQL injection** – attacker puts SQL in input: `' OR 1=1 --`.
```js
// ❌ Never
db.query(`SELECT * FROM users WHERE email = '${email}'`);
// ✅ Parameterised query
db.query("SELECT * FROM users WHERE email = $1", [email]);
```
ORMs (Prisma/TypeORM) parameterise by default – be careful with raw queries (`$queryRawUnsafe`, string-built `query()`).

**NoSQL injection** – attacker sends an object instead of a string:
```json
{ "email": "admin@x.com", "password": { "$ne": null } }
```
`User.findOne({ email, password })` would match without the real password!
Fixes:
- **Validate types** – email and password must be strings (Joi/zod/class-validator DTO).
- Sanitize – remove keys starting with `$` or containing `.` (`express-mongo-sanitize`); Mongoose `sanitizeFilter: true`.
- Never pass `req.body` / `req.query` directly into a query.
- Avoid `$where` and server-side JS.

**General:** least-privilege DB user (app user can't drop tables), whitelist sort/filter fields (`?sort=` only allowed values), error messages that don't expose queries, WAF.

---

### 10. How would you debug a production issue?

> 🎤 **Interview answer:** "First I assess the impact: who's affected, since when, and which endpoints, and I inform the team if it's serious. Then I stop the bleeding before finding the root cause: roll back the last deploy, turn off a feature flag, scale up or restart. Then I investigate with data. I look at dashboards for error rate, latency, CPU, memory, DB, Redis and queue lag, and search logs by request id, user id or error message in CloudWatch Logs Insights. The key question is what changed: a deploy, a config or env change, a data change, a third-party outage or a traffic spike. Then I reproduce it in staging with the same request or data, fix it, add a test for it, deploy, and watch the metrics. Afterwards we do a short blameless post-mortem: root cause, why monitoring didn't catch it earlier, and what prevents it next time.
>
> A real example: at KindSwap, users were receiving duplicate crypto price updates. CloudWatch logs showed the same update coming from different instance ids, because all 4 to 5 instances were publishing. I fixed it with Redis-based leader election, so only one instance publishes, with automatic failover."

1. **Understand impact** – who is affected, since when, which endpoint. If serious, inform the team.
2. **Stop the bleeding** – rollback the last deploy, toggle a feature flag, scale up, or restart if needed.
3. **Check monitoring** – error rate, latency, CPU/memory, DB, Redis, queue lag (CloudWatch dashboards/alarms).
4. **Check logs** – search by request id / user id / error message (CloudWatch Logs Insights, PM2 logs).
5. **Find what changed** – recent deploy, config/env change, data change, third-party outage, traffic spike.
6. **Reproduce** in staging/local with the same data or request.
7. **Fix**, add a test for it, deploy, and watch metrics.
8. **Post-mortem** – root cause, why monitoring didn't catch it earlier, steps to prevent (alerts, tests, validation).

Have a real story ready, e.g.: *"Duplicate price updates in KindSwap because all 4–5 instances were publishing. I found it from CloudWatch logs showing the same update from different instance ids. Fixed it with Redis leader election."*

---

### 11. What happens when an async function throws an error?

> 🎤 **Interview answer:** "An async function always returns a promise, so when it throws, nothing crashes at that line. The returned promise is rejected with that error. If the caller awaits it inside try/catch, or attaches `.catch`, it's handled. If nobody handles it, Node emits `unhandledRejection`, and since Node 15 the default behaviour is to crash the process.
>
> A few practical gotchas. In Express 4, a rejected promise in a route isn't caught, so the request hangs, which is why we use an asyncHandler or Express 5. Forgetting `await` inside a try block means the error escapes the catch. In `Promise.all`, one rejection rejects the whole thing. And an error thrown inside a setTimeout callback isn't caught by the outer try/catch, because it runs on a different call stack later."

An `async` function always returns a Promise. If it throws, the promise is **rejected** with that error. Nothing crashes at the `throw` line itself.

```js
async function getUser() { throw new Error("DB down"); }

getUser();                         // rejected promise, nobody handles it → "unhandledRejection"
await getUser();                   // throws at this line → catch with try/catch
getUser().catch(err => log(err));  // handled
```
- If nobody handles it → Node emits `unhandledRejection`. **Since Node 15 the default is to crash the process.**
- In **Express 4** the request hangs because Express doesn't catch rejected promises → use `asyncHandler` / `express-async-errors` or upgrade to Express 5.
- In `Promise.all`, one rejection rejects the whole thing (other promises keep running but their results are ignored).
- An error thrown inside a `setTimeout` callback inside an async function is **not** caught by the outer try/catch (different call stack).
- **Forgetting `await`** inside try/catch means the error is not caught: `try { save(); } catch {}` ❌ → `try { await save(); } catch {}` ✅.

---

### 12. How would you integrate a third-party API?

> 🎤 **Interview answer:** "I've done this a lot: 5 payment gateways, SMS providers and the SAP integration at Air India. First I read the docs for auth, rate limits, sandbox, error codes, webhooks and whether they support idempotency. Then I wrap the vendor in an adapter module behind our own interface. For example, all gateways implemented the same createOrder, verifyWebhook, refund and getStatus methods, so business code never depends on one vendor, switching is easy, and tests can mock it.
>
> For reliability: secrets in a secrets manager, a timeout on every call, retries with backoff only for idempotent requests, a circuit breaker if the vendor is often down, validating the response shape, and respecting their rate limits, queueing if necessary. For webhooks: verify the signature, respond fast, process async and dedupe. I log requests and responses with a correlation id, masking sensitive data. And there's always a fallback: mark the operation as pending and let a reconciliation job sync the status later. Everything is tested in the sandbox first."

(You did 5 payment gateways, SMS, SAP.)
1. **Read docs** – auth method, rate limits, sandbox, webhooks, error codes, idempotency support.
2. **Wrapper/adapter module** – one service (`RazorpayClient`) hides the vendor details. The rest of the code uses a common interface. Easy to switch vendors and mock in tests.
3. **Secrets** in env/Secrets Manager.
4. **Timeouts** on every call, **retries with backoff** only for safe/idempotent requests, **circuit breaker** if it's often down.
5. **Validate responses** – don't trust the shape blindly.
6. **Webhooks** – verify signature, respond fast, process async, dedupe.
7. **Log** request/response (mask sensitive data) with a correlation id.
8. **Rate limits** – respect them; queue requests if needed.
9. **Fallback** – queue for later, show "pending" status, reconciliation job.
10. **Test** in sandbox; write unit tests with mocked client.

---

## Added scenarios

### E1. Server CPU is at 100%

> 🎤 **Interview answer:** "First I check which process is using the CPU, with top, htop or `pm2 monit`, and whether it matches a traffic increase. If it's just more traffic, I scale out. If it's the code, I profile it: `--cpu-prof` or clinic.js flame graphs show the hot function. Typical causes are a heavy loop, a big JSON parse or stringify, a catastrophic regex, synchronous crypto, or image or PDF processing on the main thread. The fix depends on the cause: optimise the algorithm, cache the result, or move CPU-heavy work to worker threads or a separate background service through a queue."

- Check which process (`top`/`htop`, `pm2 monit`).
- Is it traffic (scale out) or code (a hot loop, big JSON parse, regex, sync crypto, image processing)?
- Profile with `node --prof` / `--cpu-prof` / clinic.js flame graph.
- Fix: move CPU work to worker threads or a separate service/queue, cache results, optimise the algorithm.

### E2. Memory keeps increasing

> 🎤 **Interview answer:** "First I confirm it's a real leak, memory steadily rising over hours in PM2 or CloudWatch and not dropping after GC, rather than normal usage. Then I take two or three heap snapshots with `--inspect` and Chrome DevTools, spaced over time under load, and compare which objects keep growing and what's retaining them. The common causes are in-memory caches or maps with no size limit, event listeners added on every request, intervals that are never cleared, closures holding large objects, and loading huge result sets fully into memory instead of streaming. As a short-term safety net, PM2's `max_memory_restart` keeps the service up, but the real fix is removing the leak, for example adding an LRU limit or TTL, or moving the cache to Redis."

- Confirm with `process.memoryUsage()`, PM2/CloudWatch memory graph over hours.
- Take 2–3 heap snapshots (`--inspect` + Chrome DevTools) and compare what grows.
- Common causes: in-memory cache/map without limit, event listeners added per request, intervals not cleared, closures holding big objects, large arrays loaded fully (use streams/pagination).
- Short-term: PM2 `max_memory_restart`. Long-term: fix the leak.

### E3. Cron job runs on every instance

> 🎤 **Interview answer:** "With 4 instances, a midnight cron runs 4 times, which means 4 emails to every user or, worse, 4 payouts. I've dealt with exactly this kind of multi-instance problem. Options: acquire a Redis lock with SET NX and a TTL before running the job, so only one instance gets it; or use leader election so only the leader runs scheduled jobs, which is what I built at KindSwap for the price publisher; or run crons in one dedicated worker process; or use a managed scheduler like EventBridge triggering a Lambda or a single queue message. On top of that, the job itself should be idempotent, for example by tracking which payouts are already done, so even an accidental double run is harmless."

With 4 instances, a cron at midnight runs 4 times (4 emails, 4 payouts!).
Fixes: Redis lock (`SET job:daily NX PX`) before running; leader election (only leader runs crons – what I built); a separate single worker for crons; or a managed scheduler (EventBridge → Lambda / one queue message). Also make the job idempotent.

### E4. Send 1 lakh emails or notifications

> 🎤 **Interview answer:** "I'd never send them inside the API request. The API creates a campaign record and returns 202 Accepted immediately. A producer then walks through the target users with a database cursor, in batches, and pushes jobs to a queue, Kafka, SQS or BullMQ. Workers consume with a concurrency limit that matches the provider's rate limit, with retries, backoff and a DLQ for failures. Each recipient is marked as sent, so if a batch is retried, no one gets a duplicate. We track progress and failures on the campaign, and respect user preferences and unsubscribes. That's the same design as the Kafka notification pipeline I built at Appsinvo."

- Don't send in the API request. Create a "campaign" record, return 202.
- Producer splits users into batches (cursor through DB) and pushes jobs to a queue (Kafka/SQS/BullMQ).
- Workers send with concurrency limits matching the provider's rate limit; retries + DLQ; mark each as sent (dedupe if retried).
- Track progress and failures; respect user preferences/unsubscribe.

### E5. Upload large files

> 🎤 **Interview answer:** "I avoid passing large files through Node's memory at all. The backend generates an S3 pre-signed URL after validating the file type and size, and the client uploads directly to S3. For very large files, I use S3 multipart upload, which supports parallel parts and resume. If the file really must go through our server, I stream it with busboy straight to S3 instead of buffering. Post-processing, like virus scanning, resizing or parsing, is triggered by an S3 event to a Lambda or a queue, and I save only the S3 key in the database."

- Don't pass big files through Node memory. Use **S3 pre-signed URLs** – client uploads directly to S3; for very large files use **multipart upload**.
- If it must pass through the server, **stream** it (`busboy`/multer stream → S3).
- Validate file type and size, scan if needed, process after upload with an S3 event → Lambda/queue.

### E6. Third-party API is slow or down

> 🎤 **Interview answer:** "I design so a third-party failure doesn't become our failure. Every call has a timeout. Idempotent calls retry with exponential backoff and jitter. A circuit breaker stops hammering a failing service and fails fast. Where possible I serve the last good cached response, or accept the request, queue the work and show the user a 'pending' status, then complete it later with a reconciliation job. We alert on it, and for critical dependencies, like OTP SMS, we keep a fallback provider ready to switch to."

Timeouts, retries with backoff + jitter (only for idempotent calls), circuit breaker, cache last good response, queue and process later, show "pending" to the user, alert, and have a fallback provider if critical (e.g. second SMS gateway).

### E7. Two users update the same record

> 🎤 **Interview answer:** "This is a race condition, and the fix depends on the case. For counters and stock, I use atomic updates: `$inc`, or an UPDATE with a condition like `stock >= 1`, so the database does the check and the write in one step. For editing a document, I use optimistic locking with a version field: the update only succeeds if the version hasn't changed, otherwise we retry or return 409, so nobody silently overwrites someone else's change. Mongoose and TypeORM both support this. When conflicts are frequent, I use pessimistic locking with `SELECT FOR UPDATE` inside a transaction. For operations across multiple resources, a Redis distributed lock. In the GRN dashboard, for example, a conditional update prevents two users from receiving more than the PO quantity at the same time."

- Atomic updates: `$inc`, `UPDATE … SET stock = stock - 1 WHERE stock >= 1`.
- **Optimistic locking** – `version` field; update only if version matches, else retry/409. (Mongoose has `optimisticConcurrency`; TypeORM `@VersionColumn`.)
- **Pessimistic locking** – `SELECT … FOR UPDATE` in a transaction.
- Distributed lock in Redis for cross-resource operations.

### E8. Zero-downtime deployment

> 🎤 **Interview answer:** "On EC2 with PM2, `pm2 reload` restarts the cluster workers one at a time, so some are always serving. With multiple instances or containers, I do a rolling deployment behind the load balancer: new instances must pass the health check before old ones are removed. Blue-green is another option, with instant rollback. Two things make it truly zero-downtime. First, graceful shutdown: on SIGTERM, stop accepting new connections, finish in-flight requests and close connections. Second, backward-compatible database migrations using expand and contract: add the new column first, deploy code that uses it, backfill, and remove the old column only in a later release, so old and new code can run at the same time."

- PM2 `pm2 reload` (restarts cluster workers one by one), or rolling deploy behind a load balancer (new instances pass health check before old ones are removed), or blue-green.
- Graceful shutdown on SIGTERM (finish in-flight requests).
- **Backward-compatible DB migrations** – add a column first, deploy code that uses it, remove old column later (expand → migrate → contract).

### E9. Redis goes down

> 🎤 **Interview answer:** "It depends on what Redis is used for. As a cache, the app should degrade, not die: Redis calls are wrapped with short timeouts and try/catch, and fall back to the database. It's slower, so I'd watch DB load closely. For rate limiting, we decide in advance between fail-open, to keep the site available, and fail-closed, for sensitive routes like OTP. For things like sessions, locks or leader election, an outage really hurts. In leader election, for example, no instance can become leader, so publishing pauses, which is a safe failure. That's why in production we use managed Redis with replicas and automatic failover, like ElastiCache Multi-AZ or Sentinel."

- Cache: app should **fall back to DB** (wrap Redis calls in try/catch, short timeouts) – slower but working. Watch DB load.
- Rate limiter: decide fail-open or fail-closed.
- Sessions/locks/leader election: affected – use managed Redis with replicas and auto-failover (ElastiCache Multi-AZ / Sentinel).

### E10. Long-running task like report generation

> 🎤 **Interview answer:** "I make it asynchronous. `POST /reports` validates the request, creates a job record and immediately returns 202 Accepted with a job id. A background worker, consuming from a queue, generates the report, uploads it to S3 and marks the job as done. The client either polls `GET /reports/:jobId`, or gets notified through a WebSocket event, push or email with a pre-signed download link. That keeps API requests short, survives restarts because the job is in a queue, and lets us retry failures."

`POST /reports` → create job, return **202 Accepted + jobId** → worker builds report → stores file in S3 → status `done` → client polls `GET /reports/:jobId` or gets a notification/websocket event with a download link (pre-signed URL).

### E11. Export lakhs of rows to CSV

> 🎤 **Interview answer:** "I stream it instead of loading everything into memory. I open a database cursor, like Mongoose's `find().cursor()` or a Postgres cursor, pass the rows through a transform stream that converts them to CSV lines, and pipe the result into the HTTP response with a Content-Disposition attachment header, using `pipeline` so backpressure and errors are handled. Memory stays flat no matter how many rows there are, and the download starts immediately. If the export is very large or slow, I turn it into a background job that writes the file to S3 and sends a link."

Stream: DB cursor (`Model.find().cursor()` / Postgres cursor) → transform to CSV → `res` with `Content-Disposition: attachment`, using `pipeline` for backpressure. For very big exports, do it as a background job (E10).

### E12. Data sync between two systems fails midway

> 🎤 **Interview answer:** "This is exactly what I handle in the Air India SAP–WMS integration. First, validation before sending, so bad records fail early with a clear reason instead of breaking the other system. Second, persist first, then send: every record is saved in our PostgreSQL database with a status, received, validated, sent, acknowledged or failed, and a unique business reference like the PO or GRN number, so nothing is lost if a call fails midway. Third, the receiving side is idempotent, using upserts by business reference, so re-sending is always safe. Fourth, failed records are retried automatically with backoff, and after the maximum attempts they're marked failed and we alert. Fifth, a dashboard shows failed records with the error reason and a retry option. And a periodic reconciliation job compares records between the two systems to catch anything that slipped through."

(Your SAP ↔ WMS integration.)
- Validate data before sending.
- Log every record with status (`pending / sent / failed`) and a unique reference (PO number, GRN number).
- Make the receiving side **idempotent** (upsert by reference) so re-sending is safe.
- Retry failed records automatically with backoff; after N tries mark failed and alert.
- Dashboard / report of failed records with a "retry" option.
- Periodic **reconciliation** job: compare counts/records in both systems.

### E13. Someone is brute-forcing the login

> 🎤 **Interview answer:** "I'd apply layered defences. Rate limiting both by IP and by account, because attackers rotate IPs. Progressive delays or a temporary lockout after several failed attempts, and a CAPTCHA after a few failures. The same generic error for a wrong email or a wrong password, so we don't reveal which accounts exist. Offering 2FA or OTP. Logging and alerting on unusual patterns, like many failures across accounts from one IP range. At the edge, a WAF or Nginx rules can block abusive IPs. In Rider Perfume, I applied this kind of rate limiting and attempt limits to the OTP login."

Rate limit by IP and by account, exponential delay / temporary lockout after N failures, CAPTCHA after a few failures, alert on unusual patterns, 2FA/OTP, same error message for wrong email or password (don't reveal which), log attempts.

### E14. Works locally, fails in production

> 🎤 **Interview answer:** "I compare the two environments systematically. Usually it's one of these: missing or different environment variables and secrets, a different Node version, different data, file path case sensitivity, since Linux is case-sensitive and Windows isn't, CORS allowed origins, network rules or security groups blocking a connection, memory limits, timeouts at Nginx or the load balancer, a stale build in dist, or time zone differences. I check the production logs for the exact error, compare configs, and try to reproduce in staging, which should mirror production as closely as possible."

Check env variables/secrets, Node version, different data, file paths/case sensitivity (Linux), CORS/allowed origins, network/security groups, memory limits, timeouts at Nginx/load balancer, build output (`dist`) up to date, time zone. Compare logs and configs.

### E15. Logout from all devices

> 🎤 **Interview answer:** "I store refresh tokens per device in the database, so 'logout from all devices' deletes all of the user's refresh tokens, and no device can get a new access token. For access tokens that are still valid, I keep a `tokenVersion` on the user record, cached in Redis, and include it in each JWT. 'Logout all' increments the version, and the auth middleware rejects any token whose version doesn't match. So every existing session ends immediately. The same mechanism is triggered on password change."

Store refresh tokens per device in DB; delete all for the user. For access tokens, keep a `tokenVersion` in the user record (also cached in Redis) and include it in the JWT; increase it on "logout all" so old tokens fail the middleware check.

### E16. Database goes down

> 🎤 **Interview answer:** "In production the database runs as a replica set or with Multi-AZ, so most failures are handled automatically by failover within seconds. On the app side, the driver reconnects with backoff, the readiness health check reports not-ready so the load balancer stops routing traffic, and APIs return a fast 503 instead of hanging. Where possible, writes can be queued and processed after recovery. We alert immediately. And backups only count if restores are tested, so we keep point-in-time recovery enabled and practise restoring."

Replica set / Multi-AZ failover handles most cases automatically. App side: connection retry with backoff, health check returns not-ready, return 503 instead of hanging, queue writes if possible, alert. Backups + tested restore.

### E17. Handling time zones

> 🎤 **Interview answer:** "I store all timestamps in UTC in the database, with `timestamptz` in Postgres, and MongoDB Dates are UTC anyway. APIs exchange ISO-8601 strings with the offset. Conversion to the user's local time zone happens only at the edges, in the UI, or when generating user-facing documents like invoices. Cron jobs are scheduled with an explicit time zone, for example IST for Indian business-day jobs, and date-only business fields, like a delivery date, are handled carefully so they don't shift a day when converted."

Store all dates in **UTC** in DB (`timestamptz` in Postgres, `Date` in Mongo is UTC). Convert to user's time zone only in the UI. Run cron jobs with explicit time zones. Use ISO-8601 strings in APIs.

### E18. Code review – what do you check?

> 🎤 **Interview answer:** "When I reviewed code for the 6 junior developers I mentored at Appsinvo, I checked, in order of importance: correctness and edge cases, like empty input, nulls and concurrency; security, meaning auth and ownership checks, input validation and injection risks; error handling; and database usage: correct indexes, no N+1 queries, pagination on lists, transactions where needed. Then maintainability: naming, readability, duplication and following our folder structure, then tests for the new logic, and no secrets or leftover console logs. Just as important is how I give feedback: politely, explaining why, with an example or a link, and separating must-fix issues from suggestions. And I appreciate good work, so reviews become a learning tool, not a gate."

(You reviewed code for 6 juniors.)
Correctness and edge cases, input validation and security (auth checks, injection), error handling, DB queries and indexes (N+1, missing pagination), naming and readability, duplication, tests added, no secrets/console logs, consistent structure. Give feedback politely with reasons and examples.

---

[Home](README.md) | [← 10 System Design](10-system-design.md) | Next: [12 Project Questions →](12-project-questions.md)
