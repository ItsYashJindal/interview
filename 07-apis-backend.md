[Home](README.md) | [← 06 TypeScript](06-typescript.md) | Next: [08 NestJS →](08-nestjs.md)

# 07 – APIs & Advanced Backend

## Contents
1. [REST vs GraphQL](#1-rest-vs-graphql)
2. [HTTP methods and status codes](#2-http-methods-and-status-codes)
3. [API versioning](#3-api-versioning)
4. [Pagination](#4-pagination)
5. [Authentication methods](#5-authentication-methods)
6. [AWS Lambda](#6-aws-lambda)
7. [Serverless architecture](#7-serverless-architecture)
8. [Docker basics](#8-docker-basics)
9. [Redis caching (advanced)](#9-redis-caching-advanced)
10. [API security](#10-api-security)
- [Extra API & backend questions (added)](#extra-api--backend-questions)
- [AWS services on your resume (added)](#aws-services-on-your-resume)

---

### 1. REST vs GraphQL

> 🎤 **Interview answer:** "REST models the API as resources with their own URLs, like `/users/1/orders`, and uses HTTP methods for the actions. The server decides the response shape, which can mean over-fetching or needing multiple requests. GraphQL exposes a single endpoint, and the client sends a query asking for exactly the fields it needs, even across related entities, in one request.
>
> REST is simpler, works naturally with HTTP caching and status codes, and is great for most CRUD and public APIs. GraphQL shines when many clients, like web and mobile, need different shapes of deeply nested data, but it brings complexity: schema and resolvers, the N+1 problem that needs DataLoader, harder caching, and query cost limits. All my production APIs have been REST, with consistent response formats and Swagger documentation."

**REST** – resources with URLs, use HTTP methods. `GET /users/1`, `GET /users/1/orders`.
**GraphQL** – one endpoint (`POST /graphql`); client sends a query saying exactly which fields it wants.

| | REST | GraphQL |
|---|---|---|
| Endpoints | Many | One |
| Data returned | Fixed by server (can over-fetch or under-fetch) | Exactly what client asks |
| Multiple resources | Many requests | One request |
| HTTP caching | Easy (GET + URL) | Harder (mostly POST) |
| Errors | HTTP status codes | Usually 200 with `errors` array |
| Versioning | `/v1`, `/v2` | Add fields, deprecate old ones |
| Learning curve | Low | Higher (schema, resolvers, N+1 → DataLoader) |
| Best for | Most CRUD APIs, public APIs, file upload | Many client types (web + mobile) needing different fields, complex nested data |

**REST principles (simple):** client-server, stateless (each request carries all info, e.g. token), cacheable, uniform interface (nouns in URL, methods for actions), layered system.

> 🎤 **Interview answer:** "The key REST constraints are: client-server separation; statelessness, meaning every request carries everything needed, like the token, and the server keeps no session; cacheable responses; a uniform interface, with nouns for resources and HTTP methods for actions; and a layered system, so proxies, load balancers or gateways can sit in between transparently. Statelessness is what lets us scale horizontally behind a load balancer."

**Good REST URL design:**

> 🎤 **Interview answer:** "I use plural nouns for resources and let the HTTP method express the action: GET `/orders` to list, GET `/orders/123` for one, POST to create, PATCH for a partial update, DELETE to remove. Nested resources show relationships, like `/users/5/orders`, but I keep nesting shallow. For actions that don't fit CRUD, like cancelling an order, I use a sub-resource such as POST `/orders/123/cancel`. I avoid verbs like `/getOrders`, and I always version, like `/api/v1`."

```
GET    /api/v1/orders            list
GET    /api/v1/orders/123        one
POST   /api/v1/orders            create
PATCH  /api/v1/orders/123        partial update
DELETE /api/v1/orders/123        delete
GET    /api/v1/users/5/orders    nested resource
POST   /api/v1/orders/123/cancel action that doesn't fit CRUD
```
Use nouns (plural), not verbs (`/getOrders` ❌).

---

### 2. HTTP methods and status codes

> 🎤 **Interview answer:** "GET reads data and must have no side effects. POST creates a resource or triggers an action. PUT replaces a whole resource. PATCH partially updates it. DELETE removes it. HEAD and OPTIONS are for headers and CORS preflight. Two properties matter. Safe means no change on the server: GET, HEAD and OPTIONS. Idempotent means repeating the request has the same effect as doing it once: GET, PUT and DELETE are idempotent, and POST is not. That's why for payments I add idempotency keys to POST requests."

| Method | Use | Safe (no change) | Idempotent (same result if repeated) |
|---|---|---|---|
| GET | Read | Yes | Yes |
| POST | Create / action | No | **No** |
| PUT | Replace whole resource | No | Yes |
| PATCH | Partial update | No | Not guaranteed |
| DELETE | Delete | No | Yes |
| HEAD | Headers only | Yes | Yes |
| OPTIONS | Allowed methods (CORS preflight) | Yes | Yes |

**Q. PUT vs PATCH?** PUT sends the full object and replaces it. PATCH sends only changed fields.

> 🎤 **Interview answer:** "PUT replaces the entire resource with the payload, so any field you don't send is effectively removed or reset, and it's idempotent. PATCH sends only the fields that change. In practice most of my update endpoints are PATCH, validated with an update DTO where all fields are optional, built with `Partial`."

**Q. Why is POST not idempotent?** Calling "create order" twice creates two orders. That's why payment APIs use idempotency keys – see [Extra → Idempotency](#q-what-is-idempotency-and-how-do-you-implement-it).

> 🎤 **Interview answer:** "Because each POST usually creates something new. If a 'create order' or 'charge payment' request is retried after a network timeout, you get two orders or a double charge. To make it safe, the client sends an `Idempotency-Key` header with a unique id per operation. The server stores that key with the result, and if the same key comes again, it returns the stored response instead of processing again. I implemented that across 5 payment gateways at Appsinvo."

**Status codes to know:**

> 🎤 **Interview answer:** "I group them by class. 2xx is success: 200 OK, 201 Created for POST, 202 Accepted when work is queued, like a Kafka job, and 204 No Content for DELETE. 3xx is redirection, including 304 Not Modified for cache validation. 4xx is a client error: 400 for invalid input, 401 when not authenticated, 403 when authenticated but not allowed, 404 not found, 409 conflict like a duplicate email or wrong state, 422 when the format is valid but a business rule fails, and 429 when rate limited. 5xx is a server error: 500 for a bug, 502 and 504 when Nginx can't get a good or timely response from the app, and 503 when overloaded or in maintenance. Using the correct code matters because clients, retries and monitoring all depend on it."

| Code | Meaning | When |
|---|---|---|
| 200 OK | Success | GET, PATCH, PUT |
| 201 Created | Resource created | POST |
| 202 Accepted | Accepted, will process later | Queued jobs (Kafka/SQS) |
| 204 No Content | Success, no body | DELETE |
| 301 / 302 | Permanent / temporary redirect | |
| 304 Not Modified | Use your cached copy | ETag match |
| 400 Bad Request | Invalid input | Validation failed |
| 401 Unauthorized | Not logged in / bad token | Missing/expired JWT |
| 403 Forbidden | Logged in but not allowed | Wrong role |
| 404 Not Found | Resource doesn't exist | |
| 405 Method Not Allowed | | |
| 409 Conflict | Duplicate / state conflict | Email already exists |
| 413 Payload Too Large | | Big upload |
| 422 Unprocessable Entity | Valid format but business rule failed | |
| 429 Too Many Requests | Rate limited | |
| 500 Internal Server Error | Bug on server | |
| 502 Bad Gateway | Proxy got bad response from upstream | Nginx → app down |
| 503 Service Unavailable | Overloaded / maintenance | |
| 504 Gateway Timeout | Upstream too slow | Nginx timeout |

---

### 3. API versioning

> 🎤 **Interview answer:** "Versioning lets us make breaking changes, like renaming fields or changing the response structure, without breaking existing clients, especially old mobile app versions that users haven't updated. The most common approach, and what I use, is the URL path, `/api/v1` and `/api/v2`, because it's explicit and easy to route, test and document. Other options are a query parameter or a header like `Accept-Version`, which keeps URLs clean but is harder to test.
>
> I only create a new version for breaking changes. Adding an optional field isn't one. I keep the old version running for a while, announce deprecation with Deprecation and Sunset headers, and monitor its usage before removing it. In Express it's separate routers per version, and NestJS has built-in versioning."

**Why:** to change the API (rename fields, change response) without breaking existing clients (old mobile app versions).

| Way | Example | Note |
|---|---|---|
| URL path | `/api/v1/users` | Most common, clear, easy to route |
| Query param | `/users?version=2` | Easy but messy |
| Header | `Accept-Version: 2` or `Accept: application/vnd.app.v2+json` | Clean URLs, harder to test in browser |

**Good practice:**
- Only make a new version for **breaking** changes. Adding a new optional field is not breaking.
- Keep old version running for some time, announce deprecation (`Deprecation`/`Sunset` headers).
- In Express: `app.use("/api/v1", v1Router); app.use("/api/v2", v2Router);`. NestJS has built-in `enableVersioning()`.

---

### 4. Pagination

> 🎤 **Interview answer:** "There are two main approaches. Offset pagination, with page and limit, translates to skip and limit. It's simple and supports jumping to any page with a total count, but it gets slow on deep pages because the database still scans all the skipped rows, and results can shift if data changes between requests. Cursor or keyset pagination uses the last item's sort key, like `createdAt` plus `_id`, and asks for items after that. It uses the index directly, so it's fast at any depth and stable when new data is added, but you can only go next or previous.
>
> I use offset for small admin tables and cursor for feeds, chat history, infinite scroll and large tables. I always cap the limit at around 50 or 100, sort on a unique combination, and avoid expensive COUNT queries on huge tables, by caching or estimating them."

**1. Offset pagination** – `?page=3&limit=20` → `SKIP 40 LIMIT 20`
- ✅ Simple, can jump to any page, shows total pages.
- ❌ Slow for deep pages (DB still reads and skips all earlier rows). Duplicates/missing items if data changes between page loads.

**2. Cursor (keyset) pagination** – `?limit=20&cursor=<last id or createdAt>`
```js
// MongoDB
const query = cursor ? { _id: { $lt: cursor } } : {};
const items = await Order.find(query).sort({ _id: -1 }).limit(limit + 1);
const hasMore = items.length > limit;
if (hasMore) items.pop();
res.json({ items, nextCursor: hasMore ? items[items.length - 1]._id : null });
```
```sql
-- SQL (sort by created_at, id to make it unique)
SELECT * FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
- ✅ Fast at any depth (uses index), stable when new data is added.
- ❌ Can't jump to page 50; only next/previous.

**When to use:** offset for admin tables with small data; cursor for feeds, infinite scroll, large tables, mobile apps.

**Total count:** `COUNT(*)` on millions of rows is slow → cache it, show an estimate, or don't show it.

Response format:
```json
{ "data": [...], "pagination": { "limit": 20, "nextCursor": "6740...", "hasMore": true } }
```
Full answer for millions of records: [11 Scenarios → Q8](11-scenario-questions.md#8-how-would-you-implement-pagination-for-millions-of-records).

---

### 5. Authentication methods

> 🎤 **Interview answer:** "The main methods are: session cookies, where the server stores the session, for classic web apps; JWT bearer tokens, which are stateless, for SPAs, mobile and microservices; API keys for server-to-server or third-party integrations; OAuth 2.0 for delegated access and 'login with Google'; OpenID Connect, which adds an ID token for identity on top of OAuth; OTP or magic links for passwordless login; and HMAC signatures or mTLS for webhooks and internal services.
>
> In my work, I've used JWT with refresh tokens, OTP login in Rider Perfume, Auth0, which is OIDC with RS256 JWTs and RBAC, at KindSwap, and HMAC signature verification for payment webhooks."

| Method | How | Use |
|---|---|---|
| Session + cookie | Server stores session, browser sends cookie | Classic web apps |
| JWT (Bearer token) | Signed token in `Authorization` header | SPAs, mobile, microservices |
| API key | Static key in header | Server-to-server, third-party integrations |
| OAuth 2.0 | Login with Google/GitHub; app gets an access token from the provider | Social login, giving third-party apps limited access |
| OpenID Connect (OIDC) | OAuth 2.0 + ID token with user identity | "Login with Google", Auth0 |
| OTP / Magic link | Code sent to phone/email | Passwordless login (Rider Perfume) |
| mTLS / HMAC signatures | Signed requests | Webhooks, banking, internal services |

**OAuth 2.0 Authorization Code flow (simple):**

> 🎤 **Interview answer:** "The app redirects the user to the provider, like Google, with its client id, the requested scopes and a redirect URL. The user logs in and approves, and Google redirects back with a short-lived authorization code. The backend then exchanges that code, together with the client secret, for an access token and, with OIDC, an ID token containing the user's identity. The app uses that to fetch the profile and create its own session or JWT. For SPAs and mobile apps that can't keep a secret, we add PKCE, which uses a code verifier and challenge, so a stolen code is useless."

1. App redirects user to Google login.
2. User logs in and approves.
3. Google redirects back with a **code**.
4. App backend exchanges code (+ client secret) for **access token** (and ID token).
5. App uses the token to get user profile. (For SPAs/mobile use **PKCE**.)

**Auth0 (used at KindSwap):** a hosted identity provider. Users log in via Auth0, our NestJS services verify the Auth0-issued JWT (RS256, public key from JWKS URL) and check **roles/permissions** in the token for RBAC.

> 🎤 **Interview answer:** "At KindSwap, Auth0 was our hosted identity provider. Users authenticated with Auth0, which issued RS256-signed access tokens containing their roles and permissions. In our NestJS services, a JWT guard verified each token using Auth0's public keys from the JWKS endpoint, checking signature, expiry, issuer and audience. Then a permissions guard with a custom decorator checked the required permission on each route, like `payout:approve`. That way no service stored passwords, and roles were managed centrally."

JWT details: [03 → JWT and refresh tokens](03-nodejs-express.md#7-jwt-and-refresh-tokens).

---

### 6. AWS Lambda

**Q. What is Lambda?**

> 🎤 **Interview answer:** "AWS Lambda is a serverless compute service: we upload a function, and AWS runs it in response to events, scaling automatically, and we pay only per request and per millisecond of execution, with no servers to manage. It can be triggered by API Gateway for HTTP, S3 uploads, SQS messages, EventBridge schedules for cron jobs, SNS or DynamoDB streams.
>
> The things to keep in mind are the 15-minute maximum run time, cold starts after idle periods, that functions are stateless, though connections created outside the handler are reused on warm invocations, and that many concurrent functions can exhaust database connections, which is why RDS Proxy exists. At Appsinvo we used Lambda for small background, event-driven tasks outside the main API."

Run code without managing servers. You upload a function; AWS runs it when an **event** happens and you pay only for run time (per ms) and requests.

**Triggers:** API Gateway (HTTP), S3 upload, SQS message, EventBridge cron schedule, DynamoDB streams, SNS.

```js
export const handler = async (event) => {
  const body = JSON.parse(event.body || "{}");
  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
};
```

**Limits/points to remember:**
- Max run time **15 minutes**.
- Memory 128 MB – 10 GB (CPU increases with memory).
- **Cold start** – first request after idle takes extra time (container start + code load). Reduce by small bundles, fewer dependencies, provisioned concurrency.
- Stateless – don't keep data in memory between calls (it may or may not survive). Connections created **outside** the handler can be reused on warm starts.
- DB connections: many Lambdas can exhaust DB connections → use **RDS Proxy**.
- `/tmp` storage for temp files.

**My use (Appsinvo):** used Lambda for background tasks like [image processing / scheduled jobs – say what you actually did]. Good for small event-driven jobs outside the main API.

---

### 7. Serverless architecture

**Q. What is serverless?**

> 🎤 **Interview answer:** "Serverless means building with fully managed services, like Lambda, API Gateway, S3, SQS and DynamoDB, where the cloud provider handles provisioning, scaling and patching. There are still servers, we just don't manage them. The benefits are no ops work, automatic scaling, including down to zero, paying only for usage, which is cheap for low or spiky traffic, and fast delivery of small features. The downsides are cold starts, execution time limits, harder local testing and debugging, vendor lock-in, and higher cost at constant high load. So I'd use it for webhooks, cron jobs, file processing and spiky workloads, and use EC2 or containers for steady, high-traffic APIs and long-lived WebSocket servers."

You build with managed services (Lambda, API Gateway, DynamoDB, S3, SQS) and the cloud handles servers, scaling and patching. "Serverless" means you don't manage servers, not that there are none.

| Pros | Cons |
|---|---|
| No server management | Cold starts |
| Auto-scales to zero and up | 15-min limit, not for long-running jobs/websockets (needs extra setup) |
| Pay per use – cheap at low/spiky traffic | Expensive at constant high traffic |
| Fast to build small features | Harder local testing and debugging |
| | Vendor lock-in |

**Good for:** webhooks, cron jobs, file processing on S3 upload, low/spiky traffic APIs, notifications.
**Not good for:** long-running processes, constant heavy traffic, real-time socket servers (use EC2/ECS).

---

### 8. Docker basics

**Q. What is Docker?**

> 🎤 **Interview answer:** "Docker packages an application together with everything it needs, the Node version, dependencies and OS libraries, into an image, and runs it as an isolated container that behaves the same on any machine. That removes the 'works on my machine' problem. An image is a read-only template, like a class, and a container is a running instance of it, like an object. A Dockerfile defines how to build the image, a registry like ECR stores images, volumes persist data, and docker-compose runs multiple containers together, like the API, Postgres and Redis for local development. Compared to a VM, a container shares the host OS kernel, so it's much lighter and starts in seconds."

A tool to package your app with everything it needs (Node version, dependencies, OS libraries) into an **image**, and run it as a **container** the same way on any machine. Solves "works on my machine".

| Term | Meaning |
|---|---|
| Image | Read-only template (like a class) |
| Container | Running instance of an image (like an object) |
| Dockerfile | Steps to build an image |
| Registry | Store for images (Docker Hub, AWS ECR) |
| Volume | Persistent storage outside the container |
| docker-compose | Run multiple containers together (app + Mongo + Redis) |

**Container vs VM:** VM has a full OS of its own (heavy, GBs, minutes to start). Container shares the host OS kernel (light, MBs, seconds).

**Good Dockerfile for Node (multi-stage):**

> 🎤 **Interview answer:** "For Node I use a multi-stage build. The first stage installs all dependencies and compiles TypeScript. The final stage starts from a small alpine image, installs only production dependencies with `npm ci --omit=dev`, copies just the built `dist` folder, runs as the non-root `node` user and starts with `node dist/main.js`. I copy package.json and the lock file before the source code so Docker can cache the npm install layer, and only code changes trigger a rebuild of later layers. I also add a `.dockerignore` for node_modules, .env and .git. The result is a smaller, faster-building and more secure image."

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Run stage – smaller image, only production deps
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```
**Why copy package.json first?** Docker caches layers. If only code changes, `npm ci` layer is reused → faster builds.
Also add a `.dockerignore` (node_modules, .env, .git).

**docker-compose example:**
```yaml
services:
  api:
    build: .
    ports: ["3000:3000"]
    env_file: .env
    depends_on: [postgres, redis]
  postgres:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: secret }
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7
volumes:
  pgdata:
```

**Common commands:** `docker build -t app .`, `docker run -p 3000:3000 app`, `docker ps`, `docker logs -f <id>`, `docker exec -it <id> sh`, `docker compose up -d`, `docker compose down`.

**CMD vs ENTRYPOINT:** ENTRYPOINT is the fixed command; CMD gives default arguments that can be overridden. **COPY vs ADD:** use COPY; ADD can also extract archives / download URLs.

> 🎤 **Interview answer:** "ENTRYPOINT defines the fixed executable the container always runs, while CMD provides default arguments, or a default command, that can be overridden at `docker run`. Usually for a Node app I just use `CMD ['node', 'dist/main.js']`. For COPY versus ADD: I always use COPY, because ADD has extra magic like extracting archives and downloading URLs, which makes builds less predictable."

---

### 9. Redis caching (advanced)

> 🎤 **Interview answer:** "Beyond basic cache-aside, a few things matter in production. Key naming: a consistent namespace like `shop:product:123`. Invalidating list caches: when one product changes, many list keys go stale, so I either use short TTLs for lists or put a version number in the key per category and bump it on change, so old keys just expire. I never use `KEYS *` in production, because it blocks Redis; I use SCAN. Eviction policy: `allkeys-lru` for a pure cache, `noeviction` if Redis holds data that must not disappear. And caching has layers: HTTP caching with Cache-Control and ETags, a CDN like CloudFront, a small in-process LRU, then Redis, then the DB."

Basics (cache-aside, TTL, invalidation, stampede) are in [03 → Caching with Redis](03-nodejs-express.md#11-caching-with-redis).

**Extra points:**
- **Key naming:** `app:entity:id` e.g. `shop:product:123`, `shop:products:list:cat=5:page=1`.
- **Invalidate list caches** – when a product changes, list keys also become stale. Options: short TTL for lists, or keep a version number per category in the key (`v3`) and increase it on change.
- **Don't use `KEYS *` in production** (blocks Redis). Use `SCAN`.
- **Eviction policies:** `allkeys-lru` (remove least recently used – good for cache), `volatile-ttl`, `noeviction` (errors when full – good when Redis is a store, not a cache).
- **Persistence:** RDB snapshots (periodic) and AOF (log every write). Cache-only Redis can skip persistence.
- **HTTP caching** is another layer: `Cache-Control`, `ETag` → `304 Not Modified`, CDN (CloudFront) for static files and public GET APIs.
- **Multi-level:** CDN → in-memory (small LRU in the process) → Redis → DB.

---

### 10. API security

> 🎤 **Interview answer:** "I think about it in layers, based on the OWASP API Top 10. The most common real issue is broken object-level authorization, so every query is scoped to the user, like finding an order by both id and userId. Authentication: bcrypt, short-lived JWTs with refresh rotation, and rate limiting on login and OTP. Injection: parameterised queries or the ORM, strict input validation and type checks, which also blocks NoSQL operator injection. Mass assignment: whitelisting fields with DTOs so nobody can send `role: admin`. Data exposure: returning only the needed fields. On top of that: helmet headers, strict CORS, HTTPS with HSTS, secrets in Secrets Manager, `npm audit`, webhook signature verification, no stack traces in production responses, and logging security events without ever logging passwords or tokens."

Checklist (based on OWASP API Top 10, explained simply):

| Risk | Protection |
|---|---|
| **Broken object-level authorization** (user A reads user B's order by changing the id) | Always check ownership: `findOne({ _id: id, userId: req.user.id })` |
| Broken authentication | bcrypt passwords, short-lived JWT, refresh rotation, rate-limit login/OTP, account lockout |
| Injection (SQL / NoSQL / command) | Parameterised queries / ORM, validate input types, sanitize, never `eval` or build shell commands from input |
| Mass assignment (user sends `role: "admin"` in body) | Whitelist fields with DTOs (pick only allowed fields) |
| Excessive data exposure | Return only needed fields; remove password, internal ids |
| No rate limiting | Rate limit per IP/user, body size limits, pagination limits (max `limit=100`) |
| XSS | Escape output, Content-Security-Policy, httpOnly cookies |
| CSRF (when using cookies) | `SameSite` cookies, CSRF tokens |
| Security misconfiguration | `helmet`, hide stack traces in production, disable `X-Powered-By`, strict CORS |
| Secrets leakage | Env variables / AWS Secrets Manager, never commit `.env`, rotate keys |
| Data in transit | HTTPS everywhere (TLS), HSTS |
| Data at rest | Encrypt sensitive fields, encrypted DB/S3 |
| Old dependencies | `npm audit`, Dependabot, keep packages updated |
| Webhooks spoofing | Verify signatures (Razorpay/Stripe HMAC) |
| Logging | Log auth failures and important actions, but never log passwords/tokens/card data |
| SSRF (server fetches user-given URL) | Allow-list domains, block internal IPs |

Injection details: [11 Scenarios → Q9](11-scenario-questions.md#9-how-do-you-secure-an-api-against-sql-injection-and-nosql-injection).

---

## Extra API & backend questions

### Q. What is idempotency and how do you implement it?

> 🎤 **Interview answer:** "An operation is idempotent if doing it multiple times has the same effect as doing it once. It's critical for payments, because networks fail and clients retry, and a retry must never charge the customer twice.
>
> For the 5 payment gateways I integrated at Appsinvo, the client generates a unique Idempotency-Key per payment attempt and sends it in a header. On the server, I atomically try to store that key, using Redis `SET NX` with a TTL or a unique index in the DB, with status 'processing'. If the insert succeeds, I process the payment and save the final response against the key. If the key already exists and is completed, I return the saved response without calling the gateway again. If it's still processing, I return 409. Separately, gateway webhooks are de-duplicated with a unique index on the gateway's payment or event id, so a webhook delivered twice is processed only once."

An operation is idempotent if doing it many times has the **same effect as doing it once**. Important for payments – network retries must not charge twice.

**Idempotency key flow (what I did for 5 payment gateways):**
1. Client creates a unique key (UUID) per payment attempt and sends it in a header `Idempotency-Key`.
2. Server checks Redis/DB for this key:
   - Not found → save it with status `processing` (atomic: `SET key value NX EX 86400` or a unique DB index), then process.
   - Found and `completed` → return the **saved response**, don't process again.
   - Found and `processing` → return 409 / "in progress".
3. After processing save the response with the key.
4. Webhooks from gateways are also de-duplicated using the gateway's payment/event id (unique index).

Details in [13 Resume → payments](13-resume-questions.md#integrated-5-payment-gateways-with-idempotency-keys).

### Q. Webhooks

> 🎤 **Interview answer:** "A webhook is an HTTP callback that another system sends to us when an event happens, like a payment succeeding. My rules for handling them: first, verify the signature, an HMAC with the shared secret computed over the raw body, and reject if it doesn't match. Second, respond 200 quickly and do the heavy processing asynchronously, because gateways retry on timeouts. Third, make processing idempotent, because webhooks can arrive more than once. Fourth, handle out-of-order events by checking the current state before updating. And fifth, don't rely only on webhooks: a reconciliation job calls the gateway's status API for payments stuck in pending."

Server-to-server HTTP callbacks when an event happens (payment success). Best practices: verify signature, respond 200 fast and process async (queue), handle duplicates (idempotent), handle out-of-order events, retry/reconcile with a status-check API if a webhook is missed.

### Q. Polling vs Long polling vs WebSockets vs SSE

> 🎤 **Interview answer:** "Polling means the client asks every few seconds. It's simple but wasteful and delayed. Long polling means the server holds the request open until there's data or a timeout, which is better but is mostly an old fallback. WebSockets open one persistent, full-duplex connection, so both sides can send at any time, which is ideal for chat and live prices. I used Socket.io for chat in the matrimony app and for real-time price updates. SSE, server-sent events, is a one-way stream from server to client over plain HTTP, with automatic reconnection, and it's great for notifications, live feeds or streaming AI responses when the client doesn't need to send messages back."

| | How | Use |
|---|---|---|
| Polling | Client asks every N seconds | Simple, wasteful |
| Long polling | Server holds request until data or timeout | Old fallback |
| WebSocket | One persistent two-way connection | Chat, live prices (Socket.io) |
| SSE (Server-Sent Events) | One-way server → client stream over HTTP | Notifications, live feed, AI responses |

### Q. How does Socket.io scale across many servers?

> 🎤 **Interview answer:** "With multiple instances, a user connected to server A won't receive an event emitted on server B, because each server only knows its own sockets. The fix has two parts. First, the Socket.io Redis adapter, which uses Redis pub/sub to broadcast emits to all instances, so room messages reach users wherever they're connected. Second, sticky sessions on the load balancer, because the Socket.io handshake can use multiple HTTP requests that must reach the same server, unless we force WebSocket-only transport. I also authenticate sockets by verifying the JWT in an `io.use` middleware during the handshake, and each user joins a room named after their user id."

Use **sticky sessions** on the load balancer + the **Redis adapter** so an event emitted on server A reaches clients connected to server B. Authenticate sockets with JWT in the handshake (`io.use(...)` middleware).

### Q. What happens when you type a URL and press Enter?

> 🎤 **Interview answer:** "First the browser resolves the domain to an IP address through DNS, checking the browser, OS and resolver caches. Then it opens a TCP connection and, for HTTPS, does a TLS handshake to verify the certificate and agree on encryption keys. It sends the HTTP request, which may go through a CDN, a load balancer and Nginx, and reaches our Node app. There, middleware like auth and validation runs, then the controller and service, hitting the cache or database, and a response is sent back. The browser parses the HTML, downloads the CSS, JavaScript and images, builds the DOM and CSSOM, and renders the page. As a backend engineer, I usually go deeper into the server part: load balancing, caching and the request lifecycle."

1. DNS lookup → IP address.
2. TCP connection (+ TLS handshake for HTTPS).
3. Browser sends HTTP request.
4. Request goes through CDN / load balancer / Nginx → app server.
5. App runs middleware → controller → DB/cache → sends response.
6. Browser receives HTML, loads CSS/JS, renders the page.

### Q. HTTP vs HTTPS

> 🎤 **Interview answer:** "HTTPS is HTTP over TLS. It encrypts the data in transit so nobody can read it, it protects against tampering, and the certificate proves the server's identity, preventing man-in-the-middle attacks. Today HTTPS is mandatory for any production API, especially with tokens and payments. In our setups, TLS terminated at Nginx with Let's Encrypt or at the AWS load balancer with ACM certificates, plus HSTS so browsers always use HTTPS."

HTTPS = HTTP over TLS. Data is encrypted, server identity verified by certificate, protects from tampering. Free certificates via Let's Encrypt / AWS ACM.

### Q. HTTP/1.1 vs HTTP/2

> 🎤 **Interview answer:** "HTTP/1.1 handles one request at a time per connection, so browsers open several connections and requests can block each other. HTTP/2 multiplexes many requests and responses over a single connection at the same time, compresses headers with HPACK and uses a binary framing layer, so page loads and API-heavy apps are faster. HTTP/3 goes further and runs over QUIC on UDP to avoid TCP head-of-line blocking."

HTTP/2 sends many requests over one connection at the same time (multiplexing), compresses headers. HTTP/1.1 needs multiple connections.

### Q. Reverse proxy / Nginx

> 🎤 **Interview answer:** "A reverse proxy sits in front of the application servers and forwards client requests to them. On EC2 I put Nginx in front of the Node apps running under PM2. Nginx handles SSL termination, gzip compression, serving static files, load balancing across multiple Node processes, request size limits, timeouts and basic rate limiting. For Socket.io, I configure the WebSocket upgrade headers, which means HTTP 1.1 plus the Upgrade and Connection headers. It also hides the app servers from direct internet access."

Nginx sits in front of Node: SSL termination, serves static files, gzip, load balancing across PM2/containers, rate limiting, request size limits, timeouts.
```nginx
upstream api { server 127.0.0.1:3000; server 127.0.0.1:3001; }
server {
  listen 443 ssl;
  server_name api.example.com;
  location / {
    proxy_pass http://api;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_http_version 1.1;                       # needed for websockets
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

### Q. Load balancer algorithms

> 🎤 **Interview answer:** "Round robin sends requests to servers in turn, which is simple and works when servers are equal. Least connections sends to the server with the fewest active connections, which is better when request times vary. IP hash always maps a client to the same server, which gives sticky sessions, useful for Socket.io. Weighted versions send more traffic to stronger servers. Load balancers also run health checks and stop routing to unhealthy instances, which is key for zero-downtime deploys."

Round robin, least connections, IP hash (sticky), weighted. Load balancer also does health checks and removes unhealthy servers.

### Q. API Gateway

> 🎤 **Interview answer:** "An API gateway is a single entry point in front of multiple backend services. It handles cross-cutting concerns in one place: routing requests to the right microservice, authentication, rate limiting, logging and metrics, request and response transformation, and sometimes caching. So the individual services can focus on business logic. Examples are AWS API Gateway, Kong, or Nginx configured as a gateway."

A single entry point in front of many services: routing, auth, rate limiting, logging, request transformation (AWS API Gateway, Kong, Nginx).

### Q. API documentation

> 🎤 **Interview answer:** "I document APIs with Swagger, which is OpenAPI. In NestJS I use `@nestjs/swagger`, so decorators on DTOs and controllers generate the docs automatically and stay in sync with the code. They include request and response schemas, auth requirements and examples. In Express projects I used swagger-jsdoc or maintained Postman collections. Good docs act as a contract between backend, frontend and QA, and they've reduced back-and-forth a lot in my teams."

Swagger / OpenAPI (on my resume). In NestJS: `@nestjs/swagger` decorators generate docs automatically. Docs help frontend and QA teams and act as a contract.

### Q. gRPC (basic)

> 🎤 **Interview answer:** "gRPC is a high-performance RPC framework from Google. It uses Protocol Buffers, a compact binary format with a strongly typed schema, over HTTP/2, so it's faster and smaller than JSON over REST, and it supports streaming in both directions. It's mainly used for internal service-to-service communication where performance and strict contracts matter. It's not directly usable from browsers without gRPC-Web, so public APIs usually stay REST."

Fast binary RPC using Protocol Buffers over HTTP/2. Used for internal service-to-service calls. Not easy from browsers.

### Q. How do you design a good error response?

> 🎤 **Interview answer:** "Every error from my APIs has the same structure: `success: false`, a machine-readable error code like `ORDER_NOT_FOUND` that clients can branch on, a human-readable message, optional details such as field-level validation errors, and a request id that matches our logs for tracing. Plus the correct HTTP status code. In production I never expose stack traces or internal messages. Unexpected errors return a generic message, while the full detail goes to the logs."

```json
{ "success": false, "error": { "code": "ORDER_NOT_FOUND", "message": "Order not found", "details": [] }, "requestId": "abc-123" }
```
Consistent shape, machine-readable code, human message, request id for log tracing, no stack trace in production.

### Q. Health check endpoint

> 🎤 **Interview answer:** "I expose a liveness endpoint like `GET /health` that just returns 200 if the process is up, and a readiness endpoint that also checks dependencies like the DB and Redis with quick pings. Load balancers, Docker and Kubernetes use them to decide whether to send traffic to an instance or restart it. During graceful shutdown, readiness starts failing first so traffic drains away before the process exits."

`GET /health` returns 200 if app is up (liveness), and a readiness check that also pings DB/Redis. Used by load balancers, Docker, Kubernetes.

### Q. Timeouts

> 🎤 **Interview answer:** "I set timeouts at every level: on outgoing HTTP calls, like axios `timeout` or an AbortController, on DB queries and connection acquisition, and on the server and Nginx. Without them, one slow third-party API, like a payment gateway or SMS provider, can hold connections and memory until the whole service gets stuck. Combined with retries with backoff for safe operations, and a circuit breaker for repeatedly failing dependencies, timeouts make a service resilient."

Always set timeouts on outgoing HTTP calls (axios `timeout`), DB queries, and server (`server.setTimeout`). A slow third party should not hold your connections forever.

---

## AWS services on your resume

> 🎤 **Interview answer:** "On AWS, I've run Node applications on EC2 with PM2 behind Nginx, stored product images, invoices and uploads in S3 using pre-signed URLs, used Lambda for event-driven background tasks, and used CloudWatch for logs, metrics and alarms. For example, at KindSwap we monitored leadership changes and publish rates for the price service and alarmed on errors. I'm also familiar with RDS for managed Postgres and MySQL, SQS for queues with DLQs, IAM roles with least privilege instead of hard-coded keys, and VPC security groups so databases are only reachable from the app servers."

| Service | What it is | How you can say you used it |
|---|---|---|
| **EC2** | Virtual server | Ran Node apps with PM2 behind Nginx |
| **S3** | File/object storage | Stored images, invoices, uploads; pre-signed URLs for direct upload/download |
| **Lambda** | Serverless functions | Background / event-triggered tasks |
| **RDS** | Managed SQL DB (Postgres/MySQL) | Managed backups, Multi-AZ, read replicas |
| **CloudWatch** | Logs, metrics, alarms | Monitored leader-election service and API errors; alarms on CPU/errors |
| **SQS** | Managed message queue | Simple async jobs, retries with DLQ (see [09](09-kafka-redis-messaging.md#kafka-vs-rabbitmq-vs-sqs)) |
| **IAM** | Users, roles, permissions | Least-privilege roles for EC2/Lambda instead of hard-coded keys |
| **VPC / Security groups** | Network isolation, firewall | DB only reachable from app servers |

**Pre-signed URL (asked often):** Backend creates a time-limited signed URL with AWS SDK; client uploads/downloads directly to/from S3. Saves server bandwidth and keeps the bucket private.

> 🎤 **Interview answer:** "A pre-signed URL is a time-limited URL that the backend generates with the AWS SDK and signs with its credentials, granting permission for one specific operation on one S3 object, like uploading `invoices/123.pdf` for the next 5 minutes. The client then uploads or downloads directly to or from S3 using that URL. The file never passes through our server, which saves bandwidth and memory, and the bucket stays private. I validate the file type and size when generating the URL, and save the S3 key in the database after upload."

---

[Home](README.md) | [← 06 TypeScript](06-typescript.md) | Next: [08 NestJS →](08-nestjs.md)
