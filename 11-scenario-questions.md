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

> 🎤 **Interview answer:** "First I'd ask: is it 10,000 per second, or 10,000 users connected at the same time? And is it mostly reading or writing? The answer changes the design.
>
> Node is already good at handling many requests at once, because it doesn't wait for I/O. So first I make sure **no route blocks the main thread** with heavy CPU work.
>
> Then I scale step by step:
> 1. **Use all CPU cores** with PM2 cluster mode.
> 2. **Add more servers** behind a load balancer. Servers must be stateless: JWT for auth, sessions in Redis, files in S3.
> 3. **Cache** popular data in Redis, so most requests never reach the DB.
> 4. **Database:** proper indexes, right connection pool size, read copies.
> 5. **Move slow work to a queue** – emails, invoices – and reply immediately.
> 6. **Protect the system:** rate limiting, request size limits, timeouts.
> 7. **Load test** before going live (k6, autocannon), and **monitor** response time and errors.
>
> From my work at Appsinvo: moving notifications and order receipts to Kafka took them out of the API request, and indexes plus caching kept the DB load low."

---

### 2. How do you improve the performance of a slow Node.js API?

> 🎤 **Interview answer:** "My rule: **measure first, then fix the biggest problem.**
>
> I add timing logs for each step, or use an APM tool, to see where the time goes: database, external API, or CPU.
>
> Most of the time it's the **database**. So I:
> - run `explain()` and add the right indexes,
> - return only the needed fields (projection, `.lean()`),
> - remove N+1 queries,
> - do filtering and pagination **in the DB**, not in JavaScript.
>
> Then: run independent calls together with `Promise.all`, cache data that rarely changes in Redis, compress responses with gzip, move heavy work to a queue, and add timeouts to slow third-party calls. Then I measure again.
>
> **Real example:** at Appsinvo, an admin user-listing API took about 4.5 seconds. Using `explain`, I found it was scanning the whole collection, returning full user documents, and filtering some data in Node after loading everything. I added compound indexes for the filters and sort, returned only the fields the list needed, and moved all filtering into the query. It came down to about **550 ms**, around 88% faster."

See [13 Resume](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).

---

### 3. How do you prevent duplicate API requests?

> 🎤 **Interview answer:** "Duplicates come from double clicks, network retries, or app bugs. On the frontend, we disable the button after the click. But that's only for user experience. **The real protection must be on the backend.**
>
> 1. **Idempotency key** – the app sends a unique `Idempotency-Key` header for each action. The server tries to save it in Redis with `SET key processing NX` ('only if it doesn't exist').
>    - New key → process the request and save the response with the key.
>    - Key already has a response → return the same saved response.
>    - Key still processing → return 409.
> 2. **Unique rule in the DB** – like a unique gateway payment id. Even if two requests get through, the second insert fails safely.
> 3. **Redis lock** for short critical actions, like applying a coupon.
> 4. **Save processed event ids** for webhooks and Kafka events.
>
> I used exactly this across 5 payment gateways at Appsinvo, so retries never caused double payments."

---

### 4. How would you implement JWT authentication?

> 🎤 **Interview answer:**
> 1. "**Register** – validate input, hash the password with bcrypt, save the user.
> 2. **Login** – check the password with `bcrypt.compare`, then create two tokens:
>    - **Access token** – short (15 min), has user id and role.
>    - **Refresh token** – long (7 days). I save its hash in the DB or Redis, with device info.
> 3. The refresh token goes in an **httpOnly cookie** (JavaScript can't read it). The access token goes in the response body, and the app keeps it in memory.
> 4. **Auth middleware** – reads the `Bearer` token, verifies it, sets `req.user`. If not valid → 401.
> 5. **Role middleware** – checks the role. If not allowed → 403.
> 6. **Refresh API** – checks the refresh token against the DB, gives a new access token and a **new** refresh token (rotation). If an old refresh token is used again, it's probably stolen, so I log out all sessions.
> 7. **Logout** – delete the refresh token.
> 8. **Basics** – secrets in a secrets manager, rate-limited login, no private data in the token, HTTPS only."

Code in [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens).

---

### 5. How do you handle errors in Express.js?

> 🎤 **Interview answer:** "I handle all errors in **one central place**:
> 1. Services throw my own `AppError` with a status code.
> 2. Async routes are wrapped with `asyncHandler` (or Express 5), so errors reach the error middleware.
> 3. Validation middleware (zod / Joi) returns a clean 400.
> 4. A 404 handler for unknown routes.
> 5. **One global error middleware at the end** – logs the full error with the request id, and sends a clean JSON response. No stack traces in production.
> 6. I convert known library errors: MongoDB duplicate key (code 11000) → 409, invalid id (CastError) → 400, expired JWT → 401.
> 7. For anything missed, `unhandledRejection` and `uncaughtException` handlers log the error, and PM2 restarts the app.
> 8. Alerts when errors suddenly increase."

Code in [03 → Error handling](03-nodejs-express.md#5-error-handling).

---

### 6. How would you optimise a slow MongoDB query?

> 🎤 **Interview answer:**
> 1. "**Find it** – MongoDB profiler, Atlas performance advisor, or our logs.
> 2. **Run `explain('executionStats')`** – look for COLLSCAN (reading every document), a big difference between documents checked and documents returned, or sorting in memory.
> 3. **Add the right compound index**, using the ESR rule: Equality fields first, then Sort, then Range.
> 4. **Return only needed fields** with projection and `.lean()`.
> 5. **Avoid queries that can't use indexes well**, like `$where`, or `$ne` / `$nin` on large data.
> 6. **In aggregations**, put `$match` and `$sort` first, and `$limit` before `$lookup`.
> 7. **For deep pages**, use cursor pagination instead of `skip`.
> 8. **If still slow**, change the schema: embed data that's read together, or save counters instead of counting every time.
> 9. **Cache** frequent results in Redis.
>
> This is exactly how I did the 4.5 s → 550 ms optimisation."

---

### 7. How do you manage database transactions?

> 🎤 **Interview answer:** "I use a transaction only when **many writes must all succeed or all fail together**. For example a money transfer, or in the Air India project, creating a GRN and updating the received quantity on the purchase order.
> - In SQL: `BEGIN ... COMMIT`, or Prisma's `$transaction`, which I used there.
> - In MongoDB: a session with `withTransaction`. But a single-document update is already all-or-nothing, so I try to design around that.
>
> My rules:
> - **Keep transactions short.** Never call an external API inside one, because rows stay locked while waiting.
> - For two people changing the same row: `SELECT ... FOR UPDATE`, or a `version` column.
> - Retry if the DB reports a deadlock or conflict.
> - **Across microservices**, one DB transaction isn't possible, so I use a **saga** with undo steps, and the **outbox pattern** for reliable events."

Code in [04 → Transactions](04-mongodb-sql.md#8-transactions). Patterns in [10 → Patterns](10-system-design.md#patterns-to-know).

---

### 8. How would you implement pagination for millions of records?

> 🎤 **Interview answer:** "I would **not** use `skip` / `OFFSET`. For page 50,000, the database still reads and throws away all the earlier rows, so it gets very slow. Also, results shift when new data is added.
>
> I'd use **cursor pagination**: sort by `created_at` and `id` together (so the order is always unique), and for the next page, ask for rows **after the last one the user saw**. With an index on that sort, every page is equally fast, even page 50,000.
>
> - The API returns `nextCursor` and `hasMore`. I find `hasMore` by fetching one extra row.
> - Max limit about 100.
> - Exact total count on millions of rows is slow, so I cache it, show an estimate, or use a 'Load more' button.
> - For complex search with many filters, I'd use Elasticsearch.
> - If the real need is an export, I don't paginate. I **stream** the data."

```sql
SELECT * FROM orders
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Code in [07 → Pagination](07-apis-backend.md#4-pagination). Export: [E11](#e11-export-lakhs-of-rows-to-csv).

---

### 9. How do you secure an API against SQL injection and NoSQL injection?

> 🎤 **Interview answer:** "**SQL injection** happens when user input is pasted directly into the SQL text. Then input like `' OR 1=1 --` changes what the query does. The fix: **always use parameterised queries**, where the input is sent separately from the SQL. ORMs like Prisma and TypeORM do this by default. I'm careful only with raw queries.
>
> **NoSQL injection** in MongoDB is different: the attacker sends an **object** instead of a string, like `password: { $ne: null }` ('password is not null'). If we pass `req.body` straight into `findOne`, it matches **without the real password**. Fixes:
> - **Check types** – email and password must be strings (zod / Joi / DTO).
> - **Remove keys that start with `$`** (`express-mongo-sanitize`, or Mongoose `sanitizeFilter`).
> - **Never pass `req.body` directly** into a query.
>
> Also: a DB user with limited permissions, only allow known sort / filter fields, and error messages that never show the query."

**SQL injection:**
```js
// ❌ Never – input is pasted into the SQL
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Parameterised – input is sent separately
db.query("SELECT * FROM users WHERE email = $1", [email]);
```

**NoSQL injection** – the attacker sends:
```json
{ "email": "admin@x.com", "password": { "$ne": null } }
```
`User.findOne({ email, password })` would match without the real password!

```js
// ✅ Fix – make sure they are plain strings
if (typeof email !== "string" || typeof password !== "string") {
  return res.status(400).json({ message: "Invalid input" });
}
```

---

### 10. How would you debug a production issue?

> 🎤 **Interview answer:**
> 1. "**Check the impact** – who is affected, since when, which API. If serious, tell the team.
> 2. **Stop the damage first** – roll back the last deploy, turn off a feature flag, add servers, or restart.
> 3. **Look at dashboards** – error rate, response time, CPU, memory, DB, Redis, queue lag.
> 4. **Search the logs** – by request id, user id or error message (CloudWatch Logs Insights).
> 5. **Find what changed** – a deploy, a config change, data change, third-party outage, or a traffic spike.
> 6. **Reproduce** it in staging with the same request or data.
> 7. **Fix it**, add a test, deploy, and watch the metrics.
> 8. **Short post-mortem** – root cause, why we didn't catch it earlier, and how to prevent it.
>
> **Real example:** at KindSwap, users were getting duplicate crypto price updates. CloudWatch logs showed the same update coming from different server ids, because all 4–5 servers were publishing. I fixed it with Redis leader election, so only one server publishes, with automatic failover."

---

### 11. What happens when an async function throws an error?

> 🎤 **Interview answer:** "An async function always returns a promise. So when it throws, **nothing crashes at that line**. Instead, the promise it returned becomes **rejected** (failed) with that error.
> - If the caller uses `await` inside try/catch, or `.catch()`, the error is handled.
> - If **nobody** handles it, Node raises `unhandledRejection`, and in Node 15+ **the app crashes**.
>
> Common mistakes:
> - In Express 4, an error in an async route isn't caught, so the request hangs. That's why we use `asyncHandler` or Express 5.
> - **Forgetting `await`** inside try/catch – then the catch never sees the error.
> - In `Promise.all`, one failure fails the whole thing.
> - An error thrown inside a `setTimeout` callback is **not** caught by the outer try/catch, because it runs later, separately."

```js
async function getUser() { throw new Error("DB down"); }

getUser();                         // nobody handles it → unhandledRejection → crash
await getUser();                   // throws here → catch with try/catch
getUser().catch(err => log(err));  // handled

// Forgetting await
try { save(); } catch {}           // ❌ error is NOT caught
try { await save(); } catch {}     // ✅ caught
```

---

### 12. How would you integrate a third-party API?

> 🎤 **Interview answer:** "I've done this a lot: 5 payment gateways, SMS providers, and SAP at Air India.
> 1. **Read the docs** – auth, rate limits, sandbox, error codes, webhooks.
> 2. **Wrap the vendor in one module** behind our own interface. For example, all payment gateways had the same methods: `createOrder`, `verifyWebhook`, `refund`, `getStatus`. The business code never talks to a vendor directly, so switching vendors is easy and tests can use a fake.
> 3. **Keys in a secrets manager.**
> 4. **Timeout on every call. Retry** with increasing waits, but only for actions that are safe to repeat. A **circuit breaker** if the vendor is often down.
> 5. **Check the response** – don't blindly trust its shape.
> 6. **Webhooks** – verify the signature, reply fast, process in the background, handle duplicates.
> 7. **Log** requests and responses with a request id, hiding private data.
> 8. **Respect their rate limits**, queue requests if needed.
> 9. **Fallback** – mark it 'pending' and let a background job fix the status later.
> 10. **Test** in their sandbox first."

---

## Added scenarios

### E1. Server CPU is at 100%

> 🎤 **Interview answer:** "First I check **which process** is using the CPU (`top`, `htop`, `pm2 monit`), and whether traffic has gone up.
> - If it's just **more traffic** → add more servers.
> - If it's **the code** → I use a profiler (`--cpu-prof` or clinic.js) to find the slow function. Usual causes: a heavy loop, a huge `JSON.parse` / `JSON.stringify`, sync crypto, or image / PDF work on the main thread.
>
> Fix: make the code faster, cache the result, or move the heavy work to a worker thread or a background service through a queue."

### E2. Memory keeps increasing

> 🎤 **Interview answer:** "First I confirm it's a real leak: memory keeps going up for hours in PM2 or CloudWatch and never comes down.
>
> Then I take 2–3 heap snapshots (`--inspect` + Chrome DevTools) a few minutes apart, and compare which objects keep growing.
>
> Common causes:
> - an in-memory cache or map with no size limit,
> - event listeners added on every request and never removed,
> - `setInterval` never cleared,
> - loading huge data fully into memory instead of streaming.
>
> Short-term safety: PM2 `max_memory_restart` restarts the app at a memory limit. Real fix: remove the leak, like adding a size limit or moving the cache to Redis."

### E3. Cron job runs on every instance

> 🎤 **Interview answer:** "With 4 servers, a midnight cron runs **4 times**. That means 4 emails to every user, or worse, 4 payouts! Fixes:
> - **Redis lock** – before running, take a lock with `SET NX` + expiry. Only one server gets it.
> - **Leader election** – only the leader runs crons. This is what I built at KindSwap.
> - **One separate worker** just for crons.
> - **A managed scheduler**, like AWS EventBridge triggering one Lambda.
>
> Also, the job itself should be safe to run twice. For example, save which payouts are already done, so an accidental second run does nothing."

### E4. Send 1 lakh emails or notifications

> 🎤 **Interview answer:** "Never inside the API request.
> 1. The API creates a 'campaign' record and **returns 202 immediately**.
> 2. A producer goes through the users **in batches** and puts jobs in a queue (Kafka, SQS or BullMQ).
> 3. Workers send them, at a speed that matches the provider's limit, with retries and a DLQ for failures.
> 4. Each user is marked as 'sent', so a retry never sends a duplicate.
> 5. We track progress and failures, and skip users who unsubscribed.
>
> It's the same design as the Kafka notification system I built at Appsinvo."

### E5. Upload large files

> 🎤 **Interview answer:** "I don't let big files pass through Node's memory.
> - The backend checks the file type and size, then creates an **S3 pre-signed URL**, and the app uploads **directly to S3**.
> - For very big files, I use **S3 multipart upload**: the file is sent in parts, in parallel, and can resume if it fails.
> - If a file really must go through our server, I **stream** it to S3 instead of loading it all in memory.
> - Work after upload (resize, scan, parse) is triggered by an S3 event to a Lambda or queue. I save only the S3 key in the DB."

### E6. Third-party API is slow or down

> 🎤 **Interview answer:** "I make sure **their problem doesn't become our problem**:
> - A **timeout** on every call.
> - **Retry** with increasing waits, only for safe-to-repeat calls.
> - A **circuit breaker** – stop calling them for a while if they keep failing, and fail fast.
> - Serve the **last good cached** answer if possible.
> - Or **accept the request, queue the work**, show the user 'pending', and finish it later.
> - **Alert** the team.
> - For critical things like OTP SMS, keep a **second provider** ready to switch to."

### E7. Two users update the same record

> 🎤 **Interview answer:** "This is a race condition. The fix depends on the case:
> - **Counters and stock** – one single update that checks and changes together: `$inc`, or `UPDATE ... WHERE stock >= 1`. The DB makes it safe.
> - **Editing a document** – a `version` field. The update works only if the version is still the same. If not, someone else changed it first, so I retry or return 409. No one silently overwrites another person's change.
> - **When clashes are frequent** – lock the row with `SELECT ... FOR UPDATE` inside a transaction.
> - **Across many resources** – a Redis lock.
>
> In the GRN dashboard, a conditional update stops two users from receiving more than the PO quantity at the same time."

### E8. Zero-downtime deployment

> 🎤 **Interview answer:**
> - "With PM2: `pm2 reload` restarts the app copies **one by one**, so some are always running.
> - With many servers: a **rolling deploy** – new servers must pass the health check before old ones are removed. Or **blue-green** – run the new version next to the old one and switch traffic, with instant rollback.
>
> Two more things are needed:
> 1. **Graceful shutdown** – on SIGTERM, stop taking new requests, finish running ones, close connections.
> 2. **Safe DB changes** – old and new code run at the same time during a deploy, so changes must work for both. First add the new column, then deploy code that uses it, fill the data, and remove the old column only in a later release."

### E9. Redis goes down

> 🎤 **Interview answer:** "It depends on what Redis is used for:
> - **Cache** – the app should still work. I wrap Redis calls in try/catch with short timeouts, and **fall back to the DB**. Slower, so I watch DB load.
> - **Rate limiting** – decide in advance: allow requests (keep the site up) or block them (safer for OTP).
> - **Sessions, locks, leader election** – these really suffer. For leader election, no server can become leader, so publishing pauses. That's a safe failure.
>
> That's why in production we use managed Redis with copies and automatic failover (ElastiCache Multi-AZ)."

### E10. Long-running task like report generation

> 🎤 **Interview answer:** "I make it a **background job**:
> 1. `POST /reports` checks the request, creates a job, and **immediately returns 202** with a job id.
> 2. A worker takes the job from a queue, builds the report, uploads it to S3, and marks the job 'done'.
> 3. The app either checks `GET /reports/:jobId` from time to time, or gets a notification (WebSocket, push or email) with a download link.
>
> This keeps API requests short, and the job isn't lost if a server restarts."

### E11. Export lakhs of rows to CSV

> 🎤 **Interview answer:** "I **stream** it instead of loading everything into memory. I read rows from the DB **one by one with a cursor**, convert each row into a CSV line, and send it straight into the HTTP response, using `pipeline`. Memory stays low no matter how many rows there are, and the download starts immediately. If the export is very big, I make it a background job that saves the file to S3 and sends a link."

```js
const { pipeline } = require("stream/promises");
const { Transform } = require("stream");

app.get("/export", async (req, res, next) => {
  res.setHeader("Content-Type", "text/csv");
  res.setHeader("Content-Disposition", "attachment; filename=orders.csv");
  res.write("id,amount,status\n");

  const toCsv = new Transform({
    objectMode: true,
    transform(order, _enc, done) {
      done(null, `${order._id},${order.amount},${order.status}\n`);
    },
  });

  try {
    await pipeline(Order.find().lean().cursor(), toCsv, res);
  } catch (err) {
    next(err);
  }
});
```

### E12. Data sync between two systems fails midway

> 🎤 **Interview answer:** "This is exactly what I handle in the Air India SAP–WMS integration.
> 1. **Validate before sending**, so bad records fail early with a clear reason.
> 2. **Save first, then send** – every record is saved in our PostgreSQL with a status (received → validated → sent → acknowledged / failed) and its business number (PO or GRN number). So nothing is lost if a call fails midway.
> 3. **Receiving side is safe to repeat** – it uses 'upsert by PO / GRN number', so sending the same record again is fine.
> 4. **Automatic retry** for failed records, waiting longer each time. After the max tries, mark it failed and alert.
> 5. **A dashboard** shows failed records with the reason and a 'Retry' button.
> 6. **A regular check job** compares records between both systems, to catch anything missed."

### E13. Someone is brute-forcing the login

> 🎤 **Interview answer:** "I use many layers:
> - **Rate limit by IP and by account** (attackers change IPs).
> - **Lock the account for a short time**, or add a growing delay, after several wrong tries. **CAPTCHA** after a few failures.
> - **Same error message** for wrong email and wrong password, so attackers can't find out which accounts exist.
> - **2FA / OTP.**
> - **Log and alert** on unusual patterns, like many failures from one IP range.
> - **Block bad IPs** at Nginx or a firewall.
>
> In Rider Perfume I applied this kind of rate limit and attempt limit to the OTP login."

### E14. Works locally, fails in production

> 🎤 **Interview answer:** "I compare the two environments step by step. It's usually one of these:
> - missing or different **environment variables** / secrets,
> - a different **Node version**,
> - **different data**,
> - **file name case** – Linux treats `User.js` and `user.js` as different files, Windows doesn't,
> - **CORS** allowed domains,
> - **network / firewall rules** blocking a connection,
> - **memory limits** or **timeouts** at Nginx or the load balancer,
> - an **old build** in `dist`,
> - **time zone** differences.
>
> I check the production logs for the exact error and try to reproduce it in staging."

### E15. Logout from all devices

> 🎤 **Interview answer:** "I save refresh tokens **per device** in the DB. 'Logout from all devices' deletes all of them, so no device can get a new access token.
>
> For access tokens that are still valid, I keep a `tokenVersion` number on the user, and put it inside every JWT. 'Logout all' **increases** the number. The auth middleware rejects any token whose number doesn't match. So every session ends immediately. I do the same on password change."

### E16. Database goes down

> 🎤 **Interview answer:** "In production, the database runs with copies (replica set / Multi-AZ), so if the main one dies, a copy takes over automatically in seconds.
>
> On the app side:
> - the DB driver reconnects automatically,
> - the health check says 'not ready', so the load balancer stops sending traffic,
> - APIs return a fast **503** instead of hanging,
> - if possible, writes are queued and done after recovery,
> - we alert immediately.
>
> And backups only matter if we **test restoring** them, so we practise that."

### E17. Handling time zones

> 🎤 **Interview answer:** "I save **all dates in UTC** in the database (`timestamptz` in Postgres; MongoDB dates are UTC already). APIs send dates as ISO strings, like `2026-11-25T10:30:00Z`. I convert to the user's local time **only when showing it**, in the UI or on an invoice. Cron jobs run with a clear time zone, like IST for Indian business jobs. And for date-only fields, like a delivery date, I'm careful they don't shift by a day when converted."

### E18. Code review – what do you check?

> 🎤 **Interview answer:** "When I reviewed code for the 6 junior developers I mentored at Appsinvo, I checked, in this order:
> 1. **Does it work correctly?** Edge cases like empty input, null, two requests at once.
> 2. **Security** – auth and ownership checks, input validation, injection.
> 3. **Error handling.**
> 4. **Database** – right indexes, no N+1 queries, pagination on lists, transactions where needed.
> 5. **Readability** – naming, no duplicate code, follows our folder structure.
> 6. **Tests** for the new logic.
> 7. **No secrets or leftover console.logs.**
>
> **How** I give feedback matters just as much: politely, explaining why, with an example, and marking clearly what must be fixed vs what is a suggestion. And I praise good work, so reviews help people learn."

---

[Home](README.md) | [← 10 System Design](10-system-design.md) | Next: [12 Project Questions →](12-project-questions.md)
