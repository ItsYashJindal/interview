[Home](README.md) | [← 15 Testing, Git & DevOps](15-testing-git-devops.md) | Back to start: [01 JavaScript](01-javascript.md)

# 16 – Last-Day Revision Sheet

Read this 1 hour before the interview. Each line is a one-line answer in simple words. Click a section title to open the full notes.

---

## Your numbers (know them by heart)
- 4.5 years experience (May 2022 – now)
- Air India: **8** SAP–WMS interfaces + Next.js GRN dashboard (Node, TS, Prisma, PostgreSQL)
- KindSwap: **100K+** users, **2** NestJS microservices (Referral, Staking), leader election across **4–5** servers
- Appsinvo: mentored **6** devs, API **4.5 s → 550 ms (88% faster)**, Kafka + DLQ, **5** payment gateways (Razorpay, Stripe, Cashfree, MyFatoorah, Tamara)
- Matrimony: **25K+** registered, **2K** daily active, MongoDB location search (2dsphere)

---

## [JavaScript](01-javascript.md)
- **var / let / const** – var: whole function, `undefined` if used early. let/const: only inside `{}`, error if used early. const can't get a new value (but an object's inside can change).
- **Hoisting** – JS notes all variables and functions before running. Functions work early, var gives `undefined`, let/const give an error.
- **TDZ** – the part before a let/const line where you can't use it.
- **Closure** – a function remembers the variables around it, even after the outer function is done (private data, debounce, memoize).
- **Promise** – "result coming later". States: pending / fulfilled / rejected. `all` (fails if one fails), `allSettled` (status of each), `race` (first to finish), `any` (first to pass).
- **async/await** – an easier way to write promise code. It works on top of promises. `async` function always returns a promise. `await` waits only inside that function.
- **Event loop** – normal code first → then promise callbacks → then setTimeout. Repeat.
- **map / filter / reduce** – change each item / keep matching items / turn into one value. Don't change the original.
- **Debounce** – run after the user stops for X ms (search box). **Throttle** – run at most once every X ms (scroll).
- **Shallow vs deep copy** – spread copies only the top level; `structuredClone` copies everything.
- **this** – depends on how the function is called; arrow functions use the outside `this`. **call** (args with commas), **apply** (args as array), **bind** (gives a new function for later).
- **Prototype** – hidden link to a parent object; JS looks up the chain for missing properties. `class` is an easier way to write the same thing.
- `==` changes types before comparing, `===` doesn't. `null` = empty on purpose, `undefined` = no value given.

## [Coding](02-javascript-coding.md)
Write from memory: debounce, throttle, deep clone, flatten, memoize, curry `sum(1)(2)()`, polyfills (map, filter, reduce, bind, Promise.all), retry with backoff, concurrency limit, LRU cache, EventEmitter, group by, two sum, palindrome (with a simple loop, no regex).

## [Node.js & Express](03-nodejs-express.md)
- **Architecture** – V8 runs JS, libuv runs the event loop. One main thread for our code; slow work (files, crypto) goes to a background thread pool (4 threads).
- **Event loop phases** – timers → poll (I/O) → check (setImmediate) → close. nextTick and promises run between each phase.
- **Middleware** – `(req, res, next)`; runs in the order added; error middleware has 4 args and goes last.
- **Error handling** – AppError + asyncHandler + one central error handler + process-level handlers.
- **AuthN vs AuthZ** – who you are (401) vs what you're allowed to do (403).
- **JWT** – header.payload.signature; anyone can read the payload (not secret). Access token 15 min + refresh token 7 days in httpOnly cookie; give a new refresh token each time.
- **Streams** – handle data piece by piece; `pipeline` handles errors and speed differences. **Buffer** – raw bytes.
- **Worker threads** (heavy JS work) vs **child process** (run another program) vs **cluster / PM2** (use all CPU cores).
- **Rate limiting** – one Redis counter shared by all servers; return 429.
- **Redis cache** – check cache → if missing read DB and save with expiry → delete key on update.

## [MongoDB & SQL](04-mongodb-sql.md)
- **Index** – like a book index, for fast search. Compound order = Equality, Sort, Range. Check with `explain()` (IXSCAN good, COLLSCAN bad). Makes writes a bit slower.
- **Aggregation** – a list of steps: `$match` first → `$group` / `$lookup` / `$project` / `$sort` / `$limit`.
- **populate vs $lookup** – populate = extra query from Node (simple); $lookup = join inside the DB (reports).
- **Schema** – embed when small and read together; reference when big or growing. 16 MB limit. Copy price/address into orders.
- **Joins** – inner (only matches), left (all left + matches), right, full, self.
- **Primary key** – unique + not empty. **Foreign key** – points to another table's primary key. Index foreign keys in Postgres yourself.
- **Transactions / ACID** – all or nothing; keep them short; `FOR UPDATE` or a version column for two people editing the same row.
- **Isolation** – read committed (Postgres default) / repeatable read (MySQL default) / serializable (safest, slowest).
- **Optimise** – find slow query → EXPLAIN → add index → fewer fields → fix N+1 → cursor pagination → cache.
- **WHERE vs HAVING** – filter rows before grouping vs filter groups after. 2nd highest salary: `MAX(salary) WHERE salary < (SELECT MAX(salary) ...)`.

## [React](05-react.md)
- Props (given by parent, read-only) vs state (own data; changing it re-renders).
- `useEffect` deps: none = every render, `[]` = only first time, `[x]` = when x changes. Return a cleanup function.
- `useMemo` = remember a value, `useCallback` = remember a function, `React.memo` = skip re-render if props are the same.
- `useRef` – a box that remembers a value without re-rendering; also for DOM access.
- Context for data that rarely changes (user, theme); Redux Toolkit for big, often-changing shared data.
- Virtual DOM – compare new vs old copy, update only what changed; keys identify list items.
- Next.js – CSR (browser builds page) / SSR (server, every request) / SSG (at build time) / ISR (rebuilt every N seconds).

## [TypeScript](06-typescript.md)
- interface (objects, can be merged) vs type (unions and everything else).
- Union `|` = one of; intersection `&` = all of; discriminated union = a common `type` field to tell them apart.
- Generics = a type placeholder: `ApiResponse<T>`, `<T extends {id: string}>`.
- Prefer string enums or string unions. `as` does no real check.
- Utility types: Partial, Required, Pick, Omit, Record, ReturnType, Awaited.
- `unknown` is safer than `any`; `never` = can't happen.
- Express: `Request<Params, ResBody, Body, Query>`; add `user` to Request with a `.d.ts` file.

## [APIs & Backend](07-apis-backend.md)
- REST (many URLs, server picks the fields) vs GraphQL (one URL, client picks the fields).
- Same result if repeated: GET, PUT, DELETE. POST is not → use idempotency keys.
- Status codes: 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504.
- Versioning – `/api/v1`; new version only for breaking changes.
- Pagination – offset (simple, slow for deep pages) vs cursor (fast, stable).
- Lambda – runs on events, pay per use, 15-min limit, cold starts, RDS Proxy for DB connections.
- Docker – image (template) vs container (running copy); two-stage builds; compose for local setup.
- Security – ownership checks, validation, parameterised queries, rate limits, helmet, CORS, HTTPS, secrets in env, webhook signatures.

## [NestJS](08-nestjs.md)
- Module / Controller / Provider; dependencies come through the constructor (DI); one shared instance by default.
- Order: middleware → guards → interceptors (before) → pipes → controller → interceptors (after) → exception filters.
- ValidationPipe `{ whitelist, transform }` with DTO classes.
- RBAC: JWT guard (checks Auth0 token) + Roles guard (checks `@Roles()` permission).

## [Kafka & Redis](09-kafka-redis-messaging.md)
- Kafka: topic → partitions → offsets. A consumer group shares partitions. Order is kept only inside one partition (use a key like orderId).
- At-least-once → a message can come twice → consumers must handle duplicates (eventId).
- DLQ – retry a few times → then send to DLQ topic with error info → continue → alert → fix → resend.
- Kafka (saved log, replay, very fast) vs RabbitMQ (task queues, routing) vs SQS (managed, simple).
- Redis pub/sub = fire and forget (not saved); use Streams/Kafka when messages must not be lost.
- Leader election: `SET key id NX PX 10000` → leader renews every 3 s (only if still owner) → others keep trying → old leader stops if renewal fails.
- Distributed lock: same `SET NX PX`; delete only if it's your lock.
- Outbox: save data + event row in one DB transaction; a worker sends the event.

## [System Design](10-system-design.md)
- Steps: ask questions → features → draw flow → data → APIs → scale/failures → trade-offs.
- Adding servers needs stateless servers. Then cache → read copies → sharding.
- Saga (undo steps), circuit breaker (stop calling a failing service), retry with growing waits + random jitter, idempotency.
- Overselling fix: `UPDATE ... SET stock = stock - q WHERE stock >= q`.

## [Scenarios](11-scenario-questions.md)
- **10K requests** – all CPU cores + more servers + cache + queue + rate limit + DB indexes/pool + monitoring.
- **Slow API** – measure → DB (index, fewer fields, N+1) → parallel calls → cache → move heavy work to a queue.
- **Duplicates** – idempotency key (Redis NX) + unique DB rules + webhook dedupe.
- **Production issue** – impact → stop the damage / rollback → dashboards → logs (request id) → what changed → fix + test → post-mortem.
- **Async throw** – becomes a rejected promise; if nobody catches it, Node 15+ crashes; Express 4 needs asyncHandler.
- **Third-party API** – wrapper class, timeouts, retries, circuit breaker, verify webhooks, logs, sandbox tests.

## [Projects](12-project-questions.md) & [Resume](13-resume-questions.md)
- Story format: problem → users → my role → 3 features → result / numbers.
- Best stories: leader election, 4.5 s → 550 ms, Kafka DLQ, 5 gateways with idempotency, SAP–WMS no-data-loss design.

## [HR](14-hr-behavioural.md)
- Tell me about yourself: now (Air India) → past (KindSwap, Appsinvo) → what you want next. 90 seconds.
- Short stays: honest, short, positive, focused on the future.
- If you don't know: say so, then think it through out loud.

---

**All the best, Yash!**

[Home](README.md) | [← 15 Testing, Git & DevOps](15-testing-git-devops.md) | Back to start: [01 JavaScript](01-javascript.md)
