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

> 🎤 **Interview answer:** "**REST** has many URLs, one for each resource, like `/users/1` or `/users/1/orders`. The HTTP method (GET, POST...) says what to do. The **server** decides what fields come back. Sometimes we get more data than we need, or we have to call many URLs.
>
> **GraphQL** has **one URL**. The **client** sends a query saying exactly which fields it wants, even from related data, and gets it in one request.
>
> REST is simpler, easy to cache, and fine for most APIs. GraphQL is useful when web and mobile apps need very different data, but it is more complex to build. All my production APIs have been REST, with Swagger docs."

| | REST | GraphQL |
|---|---|---|
| URLs | Many | One |
| Data returned | Decided by the server | Exactly what the client asks |
| Many resources | Many requests | One request |
| Caching | Easy | Harder |
| Errors | HTTP status codes | Usually 200 with an `errors` list |
| Best for | Most CRUD APIs, public APIs | Many client types needing different fields |

**REST principles (simple):**

> 🎤 **Interview answer:** "The main REST rules are:
> - **Stateless** – every request carries everything needed, like the token. The server doesn't remember anything between requests. This is what lets us add more servers easily.
> - **Uniform URLs** – nouns for resources (`/orders`), HTTP methods for actions.
> - **Cacheable** – responses can be cached.
> - **Client and server are separate.**"

**Good REST URL design:**

> 🎤 **Interview answer:** "I use plural nouns, and let the HTTP method say the action. GET `/orders` to list, GET `/orders/123` for one, POST to create, PATCH to update, DELETE to delete. For actions that aren't CRUD, like cancelling, I use POST `/orders/123/cancel`. No verbs like `/getOrders`. And always a version, like `/api/v1`."

```
GET    /api/v1/orders            list
GET    /api/v1/orders/123        get one
POST   /api/v1/orders            create
PATCH  /api/v1/orders/123        update some fields
DELETE /api/v1/orders/123        delete
GET    /api/v1/users/5/orders    orders of user 5
POST   /api/v1/orders/123/cancel action that isn't CRUD
```

---

### 2. HTTP methods and status codes

> 🎤 **Interview answer:**
> - "**GET** – read data. Must not change anything.
> - **POST** – create something, or run an action.
> - **PUT** – replace the whole thing.
> - **PATCH** – change only some fields.
> - **DELETE** – delete.
>
> Two important words:
> - **Safe** = doesn't change anything on the server (GET).
> - **Idempotent** = calling it 10 times has the same effect as calling it once. GET, PUT and DELETE are idempotent. **POST is not**: calling 'create order' twice creates two orders. That's why for payments I add an idempotency key to POST requests."

| Method | Use | Changes data? | Same result if repeated? |
|---|---|---|---|
| GET | Read | No | Yes |
| POST | Create / action | Yes | **No** |
| PUT | Replace whole resource | Yes | Yes |
| PATCH | Change some fields | Yes | Not always |
| DELETE | Delete | Yes | Yes |
| OPTIONS | Ask what's allowed (CORS preflight) | No | Yes |

**Q. PUT vs PATCH?**

> 🎤 **Interview answer:** "PUT sends the **whole object** and replaces it. Any field I don't send is lost. PATCH sends **only the fields that change**. Most of my update APIs are PATCH."

**Q. Why is POST not idempotent?**

> 🎤 **Interview answer:** "Because every POST usually creates something new. If a 'pay' request times out and the app retries, the customer could be charged twice. To make it safe, the client sends a unique `Idempotency-Key` header. The server saves this key with the result. If the same key comes again, the server returns the saved result instead of charging again. I built this for 5 payment gateways at Appsinvo."

See [Extra → Idempotency](#q-what-is-idempotency-and-how-do-you-implement-it).

**Status codes to know:**

> 🎤 **Interview answer:** "Status codes are grouped by the first number:
> - **2xx = success.** 200 OK, 201 Created (after POST), 204 No Content (after DELETE).
> - **3xx = redirect.** 304 means 'use your cached copy'.
> - **4xx = client's mistake.** 400 bad input, 401 not logged in, 403 logged in but not allowed, 404 not found, 409 conflict (like email already exists), 429 too many requests.
> - **5xx = server's problem.** 500 bug in our code, 502/504 Nginx couldn't get a good or fast answer from the app, 503 server busy or under maintenance.
>
> Using the right code matters, because apps, retries and monitoring depend on it."

| Code | Meaning | When |
|---|---|---|
| 200 OK | Success | GET, PATCH, PUT |
| 201 Created | Created | POST |
| 202 Accepted | Accepted, will do it later | Job added to a queue |
| 204 No Content | Success, nothing to return | DELETE |
| 301 / 302 | Moved permanently / temporarily | |
| 304 Not Modified | Use your cached copy | |
| 400 Bad Request | Wrong input | Validation failed |
| 401 Unauthorized | Not logged in / bad token | Missing or expired JWT |
| 403 Forbidden | Logged in but not allowed | Wrong role |
| 404 Not Found | Doesn't exist | |
| 409 Conflict | Duplicate / wrong state | Email already exists |
| 413 Payload Too Large | Request too big | Big upload |
| 422 Unprocessable Entity | Format OK, but a business rule failed | |
| 429 Too Many Requests | Rate limited | |
| 500 Internal Server Error | Bug on the server | |
| 502 Bad Gateway | Nginx got a bad reply from the app | App is down |
| 503 Service Unavailable | Server busy / maintenance | |
| 504 Gateway Timeout | App too slow to reply | Nginx timeout |

---

### 3. API versioning

> 🎤 **Interview answer:** "Versioning lets us make **breaking changes**, like renaming a field, without breaking old clients, for example users who haven't updated the mobile app. I put the version in the URL: `/api/v1`, `/api/v2`. It's clear and easy to test.
>
> I only make a new version for breaking changes. Adding a new optional field is **not** breaking. I keep the old version running for a while, tell clients it will be removed, and watch its usage before turning it off. In Express I use one router per version. NestJS has built-in versioning."

| Way | Example | Note |
|---|---|---|
| URL path | `/api/v1/users` | Most common, clear |
| Query param | `/users?version=2` | Messy |
| Header | `Accept-Version: 2` | Clean URLs, harder to test |

```js
app.use("/api/v1", v1Router);
app.use("/api/v2", v2Router);
```

---

### 4. Pagination

> 🎤 **Interview answer:** "There are two main ways:
>
> **1. Offset pagination** – `?page=3&limit=20`, which becomes skip 40, take 20. Simple, and the user can jump to any page. But for deep pages it's slow, because the database still reads all the skipped rows. And if new data is added while the user is paging, items can repeat or be missed.
>
> **2. Cursor pagination** – the client sends the id (or date) of the **last item it saw**, and we return the items after it. It uses the index directly, so it's fast at any depth. The downside: you can only go 'next', not jump to page 50.
>
> I use offset for small admin tables, and cursor for feeds, chat history, infinite scroll and big tables. I always set a max limit, like 100."

**1. Offset pagination** – `?page=3&limit=20`
- ✅ Simple, can jump to any page, can show total pages.
- ❌ Slow for deep pages. Items can repeat or go missing if data changes.

```js
const page = Number(req.query.page) || 1;
const limit = Math.min(Number(req.query.limit) || 20, 100); // max 100
const items = await Order.find().sort({ createdAt: -1 }).skip((page - 1) * limit).limit(limit);
```

**2. Cursor pagination** – `?limit=20&cursor=<id of last item>`
```js
// MongoDB – newest first
const filter = cursor ? { _id: { $lt: cursor } } : {};   // items older than the cursor
const items = await Order.find(filter).sort({ _id: -1 }).limit(limit + 1); // take 1 extra

const hasMore = items.length > limit;   // if we got the extra one, there are more
if (hasMore) items.pop();               // remove the extra one

res.json({
  items,
  nextCursor: hasMore ? items[items.length - 1]._id : null,
});
```
```sql
-- SQL (sort by created_at, then id so the order is always unique)
SELECT * FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
- ✅ Fast at any depth, stable when new data is added.
- ❌ Can't jump to page 50; only next / previous.

**Total count:** `COUNT(*)` on millions of rows is slow → cache it, show an estimate, or don't show it.

Response format:
```json
{ "data": [...], "pagination": { "limit": 20, "nextCursor": "6740...", "hasMore": true } }
```
Full answer for millions of records: [11 Scenarios → Q8](11-scenario-questions.md#8-how-would-you-implement-pagination-for-millions-of-records).

---

### 5. Authentication methods

> 🎤 **Interview answer:** "The main ways are:
> - **Session + cookie** – the server saves the session. For classic websites.
> - **JWT** – a signed token sent in a header. For mobile apps, SPAs and microservices.
> - **API key** – a fixed secret key. For server-to-server calls.
> - **OAuth 2.0 / OpenID Connect** – 'Login with Google'.
> - **OTP / magic link** – a code sent to phone or email. No password.
> - **HMAC signature** – for webhooks, to prove the request really came from the payment gateway.
>
> I've used JWT with refresh tokens, OTP login in Rider Perfume, Auth0 at KindSwap, and HMAC checks for payment webhooks."

| Method | How | Use |
|---|---|---|
| Session + cookie | Server saves session, browser sends cookie | Classic websites |
| JWT (Bearer token) | Signed token in `Authorization` header | SPAs, mobile, microservices |
| API key | Fixed key in header | Server-to-server |
| OAuth 2.0 / OIDC | Login through Google/GitHub/Auth0 | Social login |
| OTP / Magic link | Code sent to phone/email | Passwordless login |
| HMAC signature | Request signed with a shared secret | Webhooks |

**OAuth 2.0 flow ("Login with Google"), simple:**

> 🎤 **Interview answer:** "1. Our app sends the user to Google's login page.
> 2. The user logs in and clicks 'Allow'.
> 3. Google sends the user back to our app with a short-lived **code**.
> 4. Our backend sends this code plus our secret to Google, and gets back an **access token** and an **ID token** (who the user is).
> 5. We use that to get the user's profile, and then create our own JWT for the user.
>
> For mobile apps and SPAs, which can't keep a secret safely, we add an extra step called PKCE, so a stolen code is useless."

**Auth0 (used at KindSwap):**

> 🎤 **Interview answer:** "At KindSwap, Auth0 handled login for us. Users logged in through Auth0, and Auth0 gave them a signed token with their roles and permissions inside. In our NestJS services, a guard checked each token using Auth0's public key, and another guard checked if the user had the right permission for that route, like `payout:approve`. So our services never stored passwords, and roles were managed in one place."

JWT details: [03 → JWT and refresh tokens](03-nodejs-express.md#7-jwt-and-refresh-tokens).

---

### 6. AWS Lambda

**Q. What is Lambda?**

> 🎤 **Interview answer:** "AWS Lambda lets us run a function **without managing any server**. We upload the code, and AWS runs it when an **event** happens, like an HTTP call, a file uploaded to S3, a message in a queue, or a schedule (cron). It scales automatically, and we pay only for the time it runs.
>
> Things to remember:
> - Max run time is **15 minutes**.
> - **Cold start**: the first call after a quiet period is slower, because AWS has to start it up.
> - It doesn't keep data between calls, so we store data in a DB or S3.
> - Many Lambdas at once can open too many DB connections, so we use RDS Proxy.
>
> At Appsinvo we used Lambda for small background tasks outside the main API."

```js
export const handler = async (event) => {
  const body = JSON.parse(event.body || "{}");
  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
};
```

**Triggers:** API Gateway (HTTP), S3 upload, SQS message, EventBridge schedule (cron), SNS.

**My use (Appsinvo):** background tasks like [image processing / scheduled jobs – say what you actually did].

---

### 7. Serverless architecture

**Q. What is serverless?**

> 🎤 **Interview answer:** "Serverless means we use services where **AWS manages the servers** for us: Lambda, API Gateway, S3, SQS, DynamoDB. There are still servers, we just don't manage them.
>
> **Good:** no server maintenance, scales automatically (even down to zero), pay only for what you use, which is cheap for low or uneven traffic.
> **Bad:** cold starts, 15-minute limit, harder to test locally, tied to one cloud provider, and costly at constant high traffic.
>
> So I'd use it for webhooks, cron jobs and file processing. For a busy main API or a WebSocket server, I'd use EC2 or containers."

| Pros | Cons |
|---|---|
| No server management | Cold starts |
| Scales automatically | 15-minute limit |
| Pay only for use – cheap at low traffic | Expensive at constant high traffic |
| Fast to build small features | Harder to test and debug |
| | Locked to one cloud provider |

---

### 8. Docker basics

**Q. What is Docker?**

> 🎤 **Interview answer:** "Docker packs the app **with everything it needs**, like the Node version and libraries, into an **image**. Then it runs that image as a **container**, which works the same on any machine. This solves the 'it works on my machine' problem.
>
> - **Image** = the template (like a class).
> - **Container** = a running copy of the image (like an object).
> - **Dockerfile** = the steps to build the image.
> - **docker-compose** = run many containers together, like the API + Postgres + Redis.
>
> A container is much lighter than a virtual machine, because it shares the host's operating system. It starts in seconds."

| Term | Meaning |
|---|---|
| Image | Read-only template (like a class) |
| Container | Running copy of an image (like an object) |
| Dockerfile | Steps to build an image |
| Registry | Place to store images (Docker Hub, AWS ECR) |
| Volume | Storage that survives when the container is deleted |
| docker-compose | Run many containers together |

**Good Dockerfile for Node (two stages):**

> 🎤 **Interview answer:** "I use two stages. The first stage installs everything and builds the TypeScript. The second stage starts fresh from a small image, installs **only production** packages, and copies only the built `dist` folder. So the final image is small.
>
> I copy `package.json` **before** the code. Docker caches each step, so if only the code changes, it reuses the `npm ci` step and the build is much faster. I also run the app as a non-root user for safety, and add a `.dockerignore` for `node_modules`, `.env` and `.git`."

```dockerfile
# Stage 1: build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: run – small image, only production packages
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

**CMD vs ENTRYPOINT / COPY vs ADD:**

> 🎤 **Interview answer:** "`ENTRYPOINT` is the fixed command the container always runs. `CMD` gives default arguments that can be changed when running the container. For a Node app I just use `CMD ['node', 'dist/main.js']`. For copying files I always use `COPY`. `ADD` can also unzip files and download URLs, which makes builds less predictable."

---

### 9. Redis caching (advanced)

> 🎤 **Interview answer:** "A few things matter in production:
> - **Clear key names**, like `shop:product:123`.
> - **List caches**: when one product changes, many cached lists become old. I either keep a short expiry on lists, or put a version number in the key and increase it when something changes, so old keys are simply ignored.
> - **Never run `KEYS *` in production**: it freezes Redis. I use `SCAN` instead.
> - **When memory is full**: for a pure cache I use `allkeys-lru`, which removes the least-used keys.
> - **Caching has layers**: browser cache → CDN → small in-memory cache → Redis → database."

Basics (cache-aside, expiry, stampede) are in [03 → Caching with Redis](03-nodejs-express.md#11-caching-with-redis).

**Extra points:**
- **Key naming:** `app:entity:id` e.g. `shop:product:123`, `shop:products:list:cat=5:page=1`.
- **Don't use `KEYS *` in production** (it blocks Redis). Use `SCAN`.
- **When memory is full:** `allkeys-lru` (remove least-used keys – good for cache), `noeviction` (give an error – good when Redis stores important data).
- **Saving to disk:** RDB (snapshots from time to time) and AOF (logs every write). A pure cache can skip this.
- **HTTP caching:** `Cache-Control`, `ETag` → `304 Not Modified`, CDN (CloudFront) for static files.

---

### 10. API security

> 🎤 **Interview answer:** "I think of security in layers:
> - **Check ownership** – the most common real bug is user A reading user B's data by changing the id in the URL. So every query includes the user: `findOne({ _id: id, userId: req.user.id })`.
> - **Login** – bcrypt passwords, short-lived JWTs, and rate limits on login and OTP.
> - **Injection** – use the ORM or parameterised queries, and validate every input.
> - **Mass assignment** – only accept allowed fields, so nobody can send `role: 'admin'` in the body.
> - **Return only needed fields** – never send passwords or internal data.
> - **Other basics** – helmet headers, strict CORS, HTTPS, secrets in AWS Secrets Manager, `npm audit`, verify webhook signatures, no error details in production, and never log passwords or tokens."

Checklist (based on the OWASP API Top 10, explained simply):

| Risk | Protection |
|---|---|
| **User reads another user's data by changing the id** | Always check ownership: `findOne({ _id: id, userId: req.user.id })` |
| Weak login | bcrypt, short-lived JWT, refresh rotation, rate-limit login/OTP |
| Injection (SQL / NoSQL / shell) | ORM / parameterised queries, validate input types, never run user input as code |
| Mass assignment (user sends `role: "admin"`) | Only accept allowed fields (DTOs) |
| Sending too much data | Return only needed fields; remove password |
| No rate limiting | Rate limit per IP/user, max body size, max `limit=100` |
| XSS (injected scripts) | Escape output, Content-Security-Policy, httpOnly cookies |
| CSRF (when using cookies) | `SameSite` cookies, CSRF tokens |
| Bad configuration | `helmet`, hide error details in production, strict CORS |
| Leaked secrets | AWS Secrets Manager, never commit `.env`, change keys regularly |
| Data on the network | HTTPS everywhere |
| Old packages | `npm audit`, Dependabot |
| Fake webhooks | Verify the signature (Razorpay / Stripe) |
| Logging | Log failed logins and important actions, never passwords / tokens / card data |

Injection details: [11 Scenarios → Q9](11-scenario-questions.md#9-how-do-you-secure-an-api-against-sql-injection-and-nosql-injection).

---

## Extra API & backend questions

### Q. What is idempotency and how do you implement it?

> 🎤 **Interview answer:** "Idempotent means **doing it many times has the same effect as doing it once**. This is very important for payments. Networks fail and apps retry, and a retry must never charge the customer twice.
>
> For the 5 payment gateways I integrated at Appsinvo:
> 1. The app creates a unique key for each payment attempt and sends it in the `Idempotency-Key` header.
> 2. The server tries to save this key in Redis (`SET NX` – 'set only if it doesn't exist') with status 'processing'.
> 3. If the save works, it's a new request, so I process the payment and save the result with the key.
> 4. If the key already exists and is done, I return the saved result. I don't call the gateway again.
> 5. If it's still processing, I return 409 'in progress'.
>
> Webhooks from gateways can also come twice, so I save the gateway's payment id with a unique index, and process each one only once."

Details in [13 Resume → payments](13-resume-questions.md#integrated-5-payment-gateways-with-idempotency-keys).

### Q. Webhooks

> 🎤 **Interview answer:** "A webhook is when **another system calls our API** to tell us something happened, like 'payment successful'. My rules:
> 1. **Verify the signature**, to make sure it really came from the gateway.
> 2. **Reply 200 quickly**, and do the heavy work in the background. Otherwise the gateway thinks it failed and sends it again.
> 3. **Handle duplicates** – the same webhook can come twice.
> 4. **Handle wrong order** – check the current status before updating.
> 5. **Don't depend only on webhooks** – a background job checks the gateway for payments stuck in 'pending'."

### Q. Polling vs Long polling vs WebSockets vs SSE

> 🎤 **Interview answer:**
> - "**Polling** – the client asks 'anything new?' every few seconds. Simple, but wasteful.
> - **Long polling** – the server keeps the request open until there is new data. An old method.
> - **WebSocket** – one connection that stays open, and **both sides** can send messages any time. Best for chat and live prices. I used Socket.io for chat in the matrimony app.
> - **SSE (Server-Sent Events)** – the server sends updates to the client, **one way only**, over normal HTTP. Good for notifications or live feeds."

| | How | Use |
|---|---|---|
| Polling | Client asks every N seconds | Simple, wasteful |
| Long polling | Server holds request until data | Old method |
| WebSocket | One open two-way connection | Chat, live prices (Socket.io) |
| SSE | One-way server → client stream | Notifications, live feed |

### Q. How does Socket.io scale across many servers?

> 🎤 **Interview answer:** "With many servers, a user connected to server A won't get a message sent from server B, because each server only knows its own users. The fix has two parts:
> 1. **Redis adapter** – it uses Redis to pass messages between all servers, so a message reaches the user on whichever server they're connected to.
> 2. **Sticky sessions** on the load balancer – so a user always connects to the same server during the connection setup.
>
> I also check the user's JWT when the socket connects, and each user joins a room named after their user id."

### Q. What happens when you type a URL and press Enter?

> 🎤 **Interview answer:** "1. **DNS** – the browser finds the IP address of the domain.
> 2. **Connection** – it connects to the server, and for HTTPS it does a secure handshake (checks the certificate, sets up encryption).
> 3. **Request** – it sends the HTTP request.
> 4. The request goes through CDN / load balancer / Nginx to our Node app.
> 5. In the app: middleware (auth, validation) → controller → service → cache or DB → response.
> 6. The browser gets the HTML, downloads CSS, JS and images, and draws the page."

### Q. HTTP vs HTTPS

> 🎤 **Interview answer:** "HTTPS is HTTP with encryption (TLS). Nobody in the middle can read or change the data, and the certificate proves the server is real. It's a must for any production API, especially with tokens and payments. We used free certificates from Let's Encrypt or AWS."

### Q. HTTP/1.1 vs HTTP/2

> 🎤 **Interview answer:** "HTTP/1.1 sends one request at a time per connection, so browsers open many connections. HTTP/2 sends **many requests at the same time over one connection**, and compresses headers. So pages and APIs load faster."

### Q. Reverse proxy / Nginx

> 🎤 **Interview answer:** "A reverse proxy sits **in front of** our app servers and forwards requests to them. On EC2 I put Nginx in front of the Node apps running with PM2. Nginx handles HTTPS, compresses responses, serves static files, spreads requests across many Node processes, limits request size and sets timeouts. It also hides our app servers from the internet. For Socket.io, I add the special 'upgrade' headers needed for WebSockets."

```nginx
upstream api { server 127.0.0.1:3000; server 127.0.0.1:3001; }
server {
  listen 443 ssl;
  server_name api.example.com;
  location / {
    proxy_pass http://api;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_http_version 1.1;                 # needed for websockets
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

### Q. Load balancer algorithms

> 🎤 **Interview answer:**
> - "**Round robin** – send requests to servers one by one in turn.
> - **Least connections** – send to the server that is least busy.
> - **IP hash** – the same user always goes to the same server (sticky). Useful for Socket.io.
> - **Weighted** – stronger servers get more traffic.
>
> The load balancer also checks server health and stops sending traffic to a server that is down."

### Q. API Gateway

> 🎤 **Interview answer:** "An API gateway is **one front door** for many backend services. It handles common jobs in one place: sending each request to the right service, checking auth, rate limiting and logging. So each service can focus on its own business logic. Examples: AWS API Gateway, Kong, Nginx."

### Q. API documentation

> 🎤 **Interview answer:** "I document APIs with Swagger (OpenAPI). In NestJS, `@nestjs/swagger` creates the docs automatically from the code, so they're always up to date. They show the request and response shape, auth, and examples. Good docs act as a contract between backend, frontend and QA, and save a lot of back-and-forth."

### Q. gRPC (basic)

> 🎤 **Interview answer:** "gRPC is a fast way for services to call each other. Instead of JSON, it uses a compact binary format (Protocol Buffers) with a strict schema. It's faster and smaller than REST with JSON. It's mostly used for internal service-to-service calls. Browsers can't use it easily, so public APIs usually stay REST."

### Q. How do you design a good error response?

> 🎤 **Interview answer:** "Every error from my APIs has the **same shape**: `success: false`, an error code like `ORDER_NOT_FOUND` that the app can check, a readable message, optional details like which field failed validation, and a request id that matches our logs. Plus the correct HTTP status code. In production I never show stack traces."

```json
{
  "success": false,
  "error": { "code": "ORDER_NOT_FOUND", "message": "Order not found", "details": [] },
  "requestId": "abc-123"
}
```

### Q. Health check endpoint

> 🎤 **Interview answer:** "I add `GET /health`, which returns 200 if the app is running. I also add a 'ready' check that pings the DB and Redis. Load balancers and Docker use these to decide whether to send traffic to this server, or restart it."

### Q. Timeouts

> 🎤 **Interview answer:** "I set timeouts everywhere: on outgoing API calls (axios `timeout`), on DB queries, and on the server and Nginx. Without them, one slow third-party API, like a payment gateway or SMS provider, can hold connections forever until our whole service gets stuck. Together with retries and a circuit breaker, timeouts keep the service healthy."

---

## AWS services on your resume

> 🎤 **Interview answer:** "On AWS, I've run Node apps on **EC2** with PM2 behind Nginx, stored images, invoices and uploads in **S3** using pre-signed URLs, used **Lambda** for background tasks, and used **CloudWatch** for logs, metrics and alerts. For example, at KindSwap we watched leader changes and errors for the price service. I also know **RDS** for managed Postgres/MySQL, **SQS** for queues, **IAM roles** to give permissions without hard-coded keys, and **security groups** so the database can only be reached from our app servers."

| Service | What it is | How you can say you used it |
|---|---|---|
| **EC2** | Virtual server | Ran Node apps with PM2 behind Nginx |
| **S3** | File storage | Images, invoices, uploads; pre-signed URLs |
| **Lambda** | Run functions without servers | Background / event tasks |
| **RDS** | Managed SQL database | Backups, read replicas |
| **CloudWatch** | Logs, metrics, alerts | Monitored the leader-election service and API errors |
| **SQS** | Message queue | Simple background jobs with retries (see [09](09-kafka-redis-messaging.md#kafka-vs-rabbitmq-vs-sqs)) |
| **IAM** | Users, roles, permissions | Roles for EC2/Lambda instead of hard-coded keys |
| **VPC / Security groups** | Network firewall | DB only reachable from app servers |

**Pre-signed URL (asked often):**

> 🎤 **Interview answer:** "A pre-signed URL is a **temporary link** that the backend creates with the AWS SDK. It allows one action on one file, like 'upload invoices/123.pdf in the next 5 minutes'. The app then uploads the file **directly to S3** with this link. The file never passes through our server, which saves bandwidth and memory, and the S3 bucket stays private. I check file type and size before creating the link, and save the S3 key in the DB after upload."

---

[Home](README.md) | [← 06 TypeScript](06-typescript.md) | Next: [08 NestJS →](08-nestjs.md)
