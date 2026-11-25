[Home](README.md) | [← 15 Testing, Git & DevOps](15-testing-git-devops.md) | Back to start: [01 JavaScript](01-javascript.md)

# 16 – Last-Day Revision Sheet

Read this 1 hour before the interview. Each line is a one-line answer. Click a section title to open the full notes.

---

## Your numbers (know them by heart)
- 4.5 years experience (May 2022 – now)
- Air India: **8** SAP–WMS interfaces + Next.js GRN dashboard (Node, TS, Prisma, PostgreSQL)
- KindSwap: **100K+** users, **2** NestJS microservices (Referral, Staking), leader election across **4–5** instances
- Appsinvo: mentored **6** devs, API **4.5 s → 550 ms (88%)**, Kafka + DLQ, **5** payment gateways (Razorpay, Stripe, Cashfree, MyFatoorah, Tamara)
- Matrimony: **25K+** registered, **2K** daily active, MongoDB 2dsphere

---

## [JavaScript](01-javascript.md)
- **var/let/const** – var: function scope, hoisted as undefined; let/const: block scope, TDZ; const can't be re-assigned (object can be changed).
- **Hoisting** – declarations moved up in memory before running; functions fully, var as undefined, let/const uninitialised.
- **TDZ** – time from block start until let/const line; access → ReferenceError.
- **Closure** – function remembers outer variables after outer function ends (private data, debounce, memoize).
- **Promise** – future value; pending/fulfilled/rejected. `all` (fail fast), `allSettled` (all results), `race` (first settled), `any` (first success).
- **async/await** – syntax over promises; async returns a promise; throw → rejected promise.
- **Event loop** – stack empty → all microtasks (nextTick, promises) → one macrotask (timers, I/O, setImmediate) → repeat.
- **map/filter/reduce** – transform / keep matching / combine to one; don't mutate.
- **Debounce** – run after user stops for X ms (search). **Throttle** – at most once per X ms (scroll).
- **Shallow vs deep copy** – spread copies top level only; `structuredClone` copies everything.
- **this** – decided by how a function is called; arrow uses outer `this`. **call** (comma args), **apply** (array), **bind** (returns new function).
- **Prototype** – hidden link to parent object; lookup goes up the chain; `class` is syntax over prototypes.
- `==` converts types, `===` doesn't. `null` = intentionally empty, `undefined` = not assigned.

## [Coding](02-javascript-coding.md)
Write from memory: debounce, throttle, deep clone, flatten, memoize, curry `sum(1)(2)()`, polyfills (map, filter, reduce, bind, Promise.all), retry with backoff, concurrency limit, LRU cache, EventEmitter, group by, two sum.

## [Node.js & Express](03-nodejs-express.md)
- **Architecture** – V8 + libuv; single main thread + event loop; thread pool (4) for fs/crypto/dns; OS for network.
- **Event loop phases** – timers → pending → poll → check (setImmediate) → close; nextTick + promises between phases.
- **Middleware** – `(req, res, next)`; order matters; error middleware has 4 args and goes last.
- **Error handling** – AppError + asyncHandler + central handler + unhandledRejection/uncaughtException.
- **AuthN vs AuthZ** – who you are (401) vs what you can do (403).
- **JWT** – header.payload.signature; payload is readable, not encrypted. Access 15 min + refresh 7 days in httpOnly cookie, rotate refresh tokens.
- **Streams** – process data in chunks; `pipeline` handles backpressure and errors. **Buffer** – raw bytes.
- **Worker threads** (CPU work, shared memory) vs **child process** (run other programs) vs **cluster/PM2** (use all cores for HTTP).
- **Rate limiting** – Redis counter shared by all servers; token bucket / sliding window; 429.
- **Redis cache** – cache-aside + TTL + delete on update; watch for stampede and stale data.

## [MongoDB & SQL](04-mongodb-sql.md)
- **Index** – B-tree for fast lookup; compound order = Equality, Sort, Range; check with `explain()` (IXSCAN vs COLLSCAN); slows writes.
- **Aggregation** – `$match` first → `$group` / `$lookup` / `$project` / `$sort` / `$limit`; `$facet` for data + count.
- **populate vs $lookup** – populate = extra queries in Node (simple); $lookup = join inside DB (reports, filter on joined data).
- **Schema** – embed when read together & small; reference when large/unbounded; 16 MB doc limit; snapshot price/address in orders.
- **Joins** – inner (matching), left (all left + matches), right, full, self.
- **PK** – unique + not null; **FK** – points to another table's PK; index FKs in Postgres.
- **Transactions / ACID** – all or nothing; keep short; `FOR UPDATE` / optimistic version; Mongo needs replica set.
- **Isolation** – read committed (PG default) / repeatable read (MySQL default) / serializable.
- **Optimise** – find slow query → EXPLAIN → index → select fewer fields → fix N+1 → paginate (keyset) → cache.
- **WHERE vs HAVING** – before vs after grouping. 2nd highest salary: `MAX(salary) WHERE salary < MAX`.

## [React](05-react.md)
- Props (from parent, read-only) vs state (owned, changes cause re-render).
- `useEffect` deps: none = every render, `[]` = mount, `[x]` = when x changes; return cleanup.
- `useMemo` = memo value, `useCallback` = memo function, `React.memo` = skip re-render if props same.
- `useRef` – mutable box, no re-render; DOM access.
- Context for rarely changing global data; Redux Toolkit for big/frequent state (`createSlice`, `createAsyncThunk`, Immer).
- Virtual DOM – diff new vs old tree, update only changes; keys identify list items.
- Next.js – CSR / SSR / SSG / ISR; server vs client components.

## [TypeScript](06-typescript.md)
- interface (objects, merging, extends) vs type (unions, tuples, anything).
- Union `|` = one of; intersection `&` = all of; discriminated unions with a `type` field.
- Generics = type parameters: `ApiResponse<T>`, `<T extends {id: string}>`.
- Prefer string enums or string unions. Assertions (`as`) don't check at runtime.
- Utility: Partial, Required, Pick, Omit, Record, ReturnType, Awaited.
- `unknown` safer than `any`; `never` for impossible cases.
- Express: `Request<Params, ResBody, Body, Query>`; extend `Request` with `declare global namespace Express`.

## [APIs & Backend](07-apis-backend.md)
- REST (many endpoints, HTTP caching) vs GraphQL (one endpoint, client picks fields).
- Idempotent methods: GET, PUT, DELETE. POST is not → idempotency keys.
- 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504.
- Versioning – `/api/v1`; new version only for breaking changes.
- Pagination – offset (simple, slow deep) vs cursor (fast, stable).
- Lambda – event-driven, pay per use, 15-min limit, cold starts, RDS Proxy for DB connections.
- Docker – image (template) vs container (running); multi-stage builds; compose for local stack.
- Security – ownership checks, validation, parameterised queries, rate limits, helmet, CORS, HTTPS, secrets in env, webhook signatures.

## [NestJS](08-nestjs.md)
- Module / Controller / Provider; DI through constructor; singleton scope by default.
- Order: middleware → guards → interceptors (before) → pipes → handler → interceptors (after) → exception filters.
- ValidationPipe `{ whitelist, transform }` with class-validator DTOs.
- RBAC: JWT guard (Auth0 RS256 via JWKS) + Roles guard reading `@Roles()` metadata.

## [Kafka & Redis](09-kafka-redis-messaging.md)
- Kafka: topic → partitions → offsets; consumer group shares partitions; ordering only per partition (use key).
- At-least-once → idempotent consumers (eventId).
- DLQ – retry N times with backoff → send to DLQ topic with error info → commit → alert → replay.
- Kafka (log, replay, high throughput) vs RabbitMQ (routing, task queues) vs SQS (managed simple queue).
- Redis pub/sub = fire and forget (no storage); Streams/Kafka for guaranteed delivery.
- Leader election: `SET key id NX PX 10000` → renew every 3 s with owner-check Lua → others retry → stop if renewal fails.
- Distributed lock: same `SET NX PX`, release only if owner.
- Outbox: write data + event row in one DB transaction; worker publishes.

## [System Design](10-system-design.md)
- Framework: clarify → features → high-level diagram → data → APIs → scale/failure → trade-offs.
- Horizontal scaling needs stateless servers. Cache → replicas → sharding.
- Saga (compensating steps), circuit breaker, retry with backoff + jitter, idempotency.
- Overselling fix: `UPDATE ... SET stock = stock - q WHERE stock >= q`.

## [Scenarios](11-scenario-questions.md)
- **10K requests** – cluster + horizontal scale + cache + queue + rate limit + DB indexes/pool + monitor.
- **Slow API** – measure → DB (index, projection, N+1) → parallel calls → cache → queue heavy work.
- **Duplicates** – idempotency key (Redis NX) + unique DB constraints + webhook dedupe.
- **Prod issue** – impact → mitigate/rollback → metrics → logs (request id) → what changed → fix + test → post-mortem.
- **Async throw** – rejected promise; unhandled → process crash (Node 15+); Express 4 needs asyncHandler.
- **Third-party API** – adapter, timeouts, retries, circuit breaker, webhooks verified, logs, sandbox tests.

## [Projects](12-project-questions.md) & [Resume](13-resume-questions.md)
- Story format: problem → users → my role → 3 features → result/numbers.
- Best stories: leader election, 4.5 s → 550 ms, Kafka DLQ, 5 gateways with idempotency, SAP–WMS no-data-loss design.

## [HR](14-hr-behavioural.md)
- Tell me about yourself: present (Air India) → past (KindSwap, Appsinvo) → what you want next. 90 seconds.
- Short tenures: honest, brief, positive, future-focused.
- If you don't know: say so, then reason it out.

---

**All the best, Yash!**

[Home](README.md) | [← 15 Testing, Git & DevOps](15-testing-git-devops.md) | Back to start: [01 JavaScript](01-javascript.md)
