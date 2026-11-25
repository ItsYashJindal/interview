[Home](README.md) | [← 11 Scenario Questions](11-scenario-questions.md) | Next: [13 Resume Deep-Dive →](13-resume-questions.md)

# 12 – Project Questions

Interviewers usually say: *"Pick one project and explain it."* Prepare **two** projects fully:
- **Project A – Rider Perfume** (riderbeauty.com, live e-commerce, you built the full backend)
- **Project B – Veer Maratha Matrimony** (pairsindia.in, 25K+ users, 2K daily active)

And keep the **Air India SAP–WMS** work ready as your "current work" story (see [13 Resume](13-resume-questions.md#notus-services--air-india-current)).

> ⚠️ The answers below are **sample answers built from your resume**. Wherever you see *[adjust]*, change it to what you actually did. Never say something in the interview you can't explain in depth.

## Contents
13. [Project overview – what problem does it solve?](#13-project-overview--what-problem-does-it-solve)
14. [Architecture – how does the frontend talk to the backend?](#14-architecture--how-does-the-frontend-communicate-with-the-backend)
15. [Tech stack – why MongoDB, Express, React, Node?](#15-tech-stack--why-mongodb-express-react-and-node)
16. [Authentication – how did you implement login and registration?](#16-authentication--how-did-you-implement-login-and-registration)
17. [APIs – explain 2 or 3 important APIs](#17-apis--explain-two-or-three-important-apis)
18. [Database – how did you design your schemas?](#18-database--how-did-you-design-your-schemas)
19. [Challenges – what technical issue did you face?](#19-challenges--what-technical-issue-did-you-face)
20. [Optimisation – how did you improve performance?](#20-optimisation--how-did-you-improve-performance)
21. [Security – how did you protect user data?](#21-security--how-did-you-protect-user-data)
- [Extra project questions (added)](#extra-project-questions)

---

### 13. Project overview – what problem does it solve?

**Rider Perfume (2-minute answer):**
"Rider Perfume is a live e-commerce website for a perfume and beauty brand – riderbeauty.com. Before this, the brand sold mostly offline/through marketplaces *[adjust]* and wanted its own store to sell directly, manage stock and send invoices automatically.
I built the **complete backend**: product catalogue, cart and orders, inventory, OTP login, payments, automated invoices, caching, pagination and rate limiting. The frontend team / I *[adjust]* consumed my REST APIs.
The main goals were: correct stock (no overselling), fast product pages, and secure checkout."

**Veer Maratha Matrimony:**
"A matrimony platform for the Maratha community – pairsindia.in. Users create profiles, search matches by preferences and location, send interests, chat in real time and buy premium plans. I owned the backend. It has **25K+ registered users and about 2K daily active users**. Key features I built: geo-search with MongoDB 2dsphere ('matches near me'), matchmaking filters, real-time chat with Socket.io, payment and SMS integrations, deployed on AWS."

**Tip:** Say *problem → users → your role → 3 main features → result/scale*.

---

### 14. Architecture – how does the frontend communicate with the backend?

> 🎤 **Interview answer:** "The frontend, React or Next.js on the web or a mobile app, talks to the backend over HTTPS using REST APIs with JSON, and for real-time features like chat it keeps a Socket.io WebSocket connection. Every request first hits Nginx, which handles SSL, gzip and reverse proxying to the Node.js and Express API running in PM2 cluster mode on EC2.
>
> Inside the API I follow a layered structure. Routes map URLs to controllers. Middleware handles rate limiting, JWT authentication and validation. Controllers stay thin: they read the request and call services. Services contain the business logic and talk to MongoDB, Redis for cache and OTP, S3 for images and invoices, and third parties like the payment gateway, SMS and email.
>
> For example, when placing an order, the frontend sends POST `/api/v1/orders` with the Bearer token. The middleware chain runs, and the order service checks and reduces stock atomically, creates the order and the gateway payment order, and returns 201. Later the gateway's webhook confirms payment, and the invoice and email are generated in the background. The layered structure keeps controllers clean and lets me unit-test services with Jest."

```
 React / Next.js frontend  (or mobile app)
        │  HTTPS (REST, JSON)  + WebSocket (Socket.io for chat)
        ▼
     Nginx  (SSL, gzip, reverse proxy, load balancing)
        ▼
 Node.js + Express API  (PM2 cluster on AWS EC2)
   ├── Routes → Middleware (auth, validation, rate limit) → Controllers → Services
   ├── MongoDB  (main data)
   ├── Redis    (cache, OTP, rate limit) [adjust]
   ├── S3       (product images, invoices)
   └── 3rd party: payment gateway, SMS provider, email
```

**Request flow example (place order):**
1. Frontend sends `POST /api/v1/orders` with `Authorization: Bearer <accessToken>` and JSON body.
2. Nginx forwards to a Node process.
3. Middleware: rate limit → JWT auth → validate body.
4. Controller calls `OrderService`: checks stock atomically, creates order, creates payment order with gateway.
5. Returns JSON `{ success, data }` with proper status code (201).
6. Payment gateway calls our **webhook** → order marked paid → invoice generated and emailed (background).

**Folder structure (layered):**
```
src/
  routes/       → URL to controller mapping
  controllers/  → read req, call service, send res
  services/     → business logic
  models/       → Mongoose schemas
  middlewares/  → auth, validate, errorHandler, rateLimit
  utils/        → helpers (logger, AppError, asyncHandler)
  config/       → env, db, redis
  jobs/         → background tasks (invoice, emails)
```
**Why layered?** Controllers stay thin, business logic is reusable and testable with Jest.

**Real-time (matrimony chat):** client opens a Socket.io connection with the JWT; server joins the user to a room = userId; messages are saved in MongoDB then emitted to the receiver's room.

---

### 15. Tech stack – why MongoDB, Express, React and Node?

> 🎤 **Interview answer:** "Node.js because its non-blocking I/O is a great fit for APIs and real-time features like chat with many open connections, and because using JavaScript or TypeScript on both frontend and backend speeds up development and lets us share validation. Express because it's minimal and flexible, with a huge middleware ecosystem for auth, rate limiting and uploads, so we could structure it the way we wanted. MongoDB because the data was document-shaped: perfumes have different attributes, and matrimony profiles have many optional fields and nested preferences. Its built-in 2dsphere geo queries were a major reason for the matrimony 'near me' search. React for a component-based UI and fast dashboards.
>
> I'm also honest about the trade-offs. MongoDB has weaker joins, so for orders and payments we kept documents well structured, used atomic updates for stock and transactions where several documents change together. And when data is heavily relational, like the Air India SAP–WMS work, I choose PostgreSQL with Prisma. I pick the database based on the data, not habit."

- **Node.js** – non-blocking I/O, good for APIs and real-time features (chat) with many open connections. Same language (JavaScript/TypeScript) on frontend and backend → faster development, shared validation/types.
- **Express** – minimal and flexible, huge middleware ecosystem (auth, rate limit, upload), easy to structure the way we want.
- **MongoDB** – flexible schema fits product catalogues (different perfumes have different attributes) and matrimony profiles (many optional fields, preferences). JSON documents map directly to JS objects. Built-in **geo queries (2dsphere)** were a big reason for the matrimony project. Easy to scale with replica sets.
- **React** – component-based UI, large ecosystem, fast with the Virtual DOM; good for dashboards and SPAs.

**Be honest about trade-offs (shows experience):**
"MongoDB doesn't have strong joins like SQL, so for things like orders and payments we kept the documents well structured, used transactions where multiple documents change together, and used atomic updates for stock. For heavily relational data, like the Air India SAP–WMS work, I used PostgreSQL with Prisma – I choose the DB based on the data."

---

### 16. Authentication – how did you implement login and registration?

> 🎤 **Interview answer:** "In Rider Perfume we used OTP login *[adjust details]*. The user enters a phone number. I validate it and rate-limit OTP requests per phone and IP. Then I generate a 6-digit OTP, store its hash in Redis with a 5-minute TTL, and send it by SMS. On verify, I compare the hash, allow a maximum of about 5 wrong attempts, and delete the OTP after use so it's one-time. A new user is created on first login. Then I issue a short-lived access token and a refresh token in an httpOnly cookie, with refresh and logout endpoints. Admin routes also check the role.
>
> In the matrimony app *[adjust]*, registration is email or phone with a password: validated, checked for duplicates with a unique index, hashed with bcrypt, and the phone verified with OTP. Login returns a JWT. For chat, the same token is verified in the Socket.io handshake middleware before the connection is accepted. For authorisation, users can only edit their own profile, and premium-only features are checked in middleware."

**Rider Perfume – OTP login** *[adjust details]*:
1. `POST /auth/otp/send { phone }` → validate phone → rate limit (e.g. 3 OTPs / 10 min per phone and IP) → generate 6-digit OTP → store **hashed** OTP in Redis with 5-min TTL → send via SMS provider.
2. `POST /auth/otp/verify { phone, otp }` → compare → delete OTP (one-time use) → max 5 wrong attempts → create user if new → return **access token (short-lived)** + **refresh token** (httpOnly cookie).
3. Protected routes use JWT middleware; admin routes check `role === "admin"`.
4. `POST /auth/refresh` and `POST /auth/logout` (delete refresh token).

**Matrimony – email/phone + password and JWT** *[adjust]*:
- Register: validate → check duplicate (unique index on phone/email) → **bcrypt** hash → save → verify phone via OTP.
- Login: compare password → issue JWT.
- **Socket.io auth:** token sent in the handshake, verified in `io.use()` middleware before connection is accepted.
- Authorization: user can only edit their own profile; premium-only features checked in middleware (`isPremium`).

Theory in [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens), full flow in [11 → Q4](11-scenario-questions.md#4-how-would-you-implement-jwt-authentication).

---

### 17. APIs – explain two or three important APIs

> 🎤 **Interview answer:** "I'll explain three from Rider Perfume. First, place order. It takes the cart items, address id and an Idempotency-Key header. I validate the items and always load prices from the database, never trusting client prices. I reduce stock atomically per item with a conditional update, stock greater than or equal to the quantity, and if any item fails, the transaction rolls back and we return 409 out of stock. Then I create the order in `pending_payment` with the total calculated on the server, create the gateway payment order, and return 201.
>
> Second, the payment webhook. I verify the gateway's HMAC signature, find the payment by gateway payment id, and if it's already processed, just return 200, which makes it idempotent. Otherwise I mark the payment and order as paid and trigger the invoice and email in the background. If payment failed or expired, the reserved stock is released.
>
> Third, product listing, with filters, sort and pagination. Sort fields are whitelisted, the limit is capped at 50, compound indexes support the common filters, projection returns only listing fields, and results are cached in Redis by query key and invalidated when products change. In the matrimony app, the key API is match search, a `$geoNear` aggregation with preference filters and cursor pagination, returning only rounded distance."

**API 1 – Place order (Rider Perfume)** `POST /api/v1/orders`
- Input: cart items, address id, `Idempotency-Key` header.
- Steps:
  1. Validate items; load **prices from DB** (never trust prices from the client).
  2. For each item reduce stock **atomically**: `updateOne({ _id, stock: { $gte: qty } }, { $inc: { stock: -qty } })`; if any fails → roll back (transaction) → 409 "out of stock".
  3. Create order with `status: pending_payment`, total calculated on server.
  4. Create payment order on gateway; return its id to the client.
- Output: 201 with order id + payment details.

**API 2 – Payment webhook** `POST /api/v1/payments/webhook`
- Verify gateway **signature** (HMAC with secret) – reject if invalid.
- Find payment by gateway payment id; if already processed → return 200 (idempotent).
- Mark payment and order as paid; trigger invoice PDF + email in the background.
- If payment failed/expired → release reserved stock.

**API 3 – Product listing** `GET /api/v1/products?category=&minPrice=&maxPrice=&sort=&page=&limit=`
- Whitelist sort fields, cap `limit` at 50.
- Compound indexes for common filters; projection to return only listing fields.
- Cached in Redis by query key for a few minutes; cache cleared when products change.

**Matrimony – Search matches** `GET /api/v1/matches?distanceKm=50&ageMin=24&ageMax=30&religion=...&cursor=`
- Builds a `$geoNear` aggregation from the user's location + filters from their preferences; excludes blocked users and self; paginated; returns distance (rounded), not exact location.

---

### 18. Database – how did you design your schemas?

> 🎤 **Interview answer:** "For Rider Perfume *[adjust fields]*, the main collections were users, products, orders, payments and coupons. A few design decisions are worth highlighting. Order items and the shipping address are copied into the order as a snapshot, so if a product's price or the user's address changes later, old orders and invoices stay correct. Addresses are embedded in the user because there are few and they're always read with the user. Orders and payments are separate collections because they grow without limit. Amounts are stored as integers in paise to avoid floating-point errors. Indexes match the queries: category plus price for listings, a unique slug, userId plus createdAt on orders, and a unique gateway payment id on payments for webhook idempotency.
>
> For the matrimony app, the profile embeds preferences and a few photos and has a 2dsphere index on location, while interests, messages and subscriptions are separate collections. Interests have a unique index on from and to, and messages are indexed on conversationId and createdAt for chat history."

**Rider Perfume** *[adjust fields]*:
```js
User     { _id, phone (unique), name, email, role: "user"|"admin", addresses: [ {line1, city, pincode, isDefault} ], createdAt }
Product  { _id, name, slug (unique), description, category, brand, price, mrp, stock, images: [String], attributes: { volumeMl, fragrance }, isActive, ratingAvg, ratingCount }
Order    { _id, userId (ref), items: [ { productId, name, price, qty } ], address: {...snapshot}, totalAmount,
           status: "pending_payment"|"paid"|"shipped"|"delivered"|"cancelled", paymentId, invoiceUrl, createdAt }
Payment  { _id, orderId, gateway, gatewayOrderId, gatewayPaymentId (unique), amount, status, idempotencyKey (unique), raw }
Coupon   { code (unique), type, value, minAmount, expiresAt, usageLimit, usedCount }
```
**Design decisions to mention:**
- Order **items and address are copied (snapshot)** into the order – if a product price or user address changes later, old orders stay correct.
- Addresses **embedded** in user (few per user, always read with user).
- Orders and payments in **separate collections** (grow without limit).
- Indexes: `{ category: 1, price: 1 }`, `{ slug: 1 }` unique, `{ userId: 1, createdAt: -1 }` on orders, unique `gatewayPaymentId`.
- Amounts stored as integers in paise *[adjust]* to avoid float errors.

**Matrimony** – see the profile schema in [04 → Schema design](04-mongodb-sql.md#5-mongodb-schema-design). Plus: `Interest { from, to, status }` with unique `{from, to}`, `Message { conversationId, senderId, text, createdAt, readAt }` with index `{ conversationId: 1, createdAt: -1 }`, `Subscription { userId, plan, validTill, paymentId }`.

---

### 19. Challenges – what technical issue did you face?

> 🎤 **Interview answer:** "The most interesting challenge was at KindSwap. Our price service ran on 4 to 5 instances, and every instance was fetching crypto prices and publishing updates, so users saw duplicate price updates and we were calling the price provider several times more than needed. The task was to make exactly one instance publish, with automatic failover if it died.
>
> I built leader election with Redis. Each instance tries `SET key myId NX PX 10000`, and only one succeeds and becomes leader. The leader renews the key every 3 seconds with a Lua script that checks it's still the owner. Followers keep retrying, so if the leader crashes, the key expires and another instance takes over within about 10 seconds. If renewal fails, the old leader immediately stops publishing. We logged leadership changes and publish counts to CloudWatch with alarms. The result: duplicate updates were eliminated, and failover happened automatically when an instance restarted *[adjust]*.
>
> I also have other stories ready if they want a different one: the 4.5-second-to-550-millisecond API optimisation, preventing overselling with atomic stock updates, scaling chat across servers with the Redis adapter, and preventing double payments with idempotency keys."

Pick **one** and tell it with **STAR** (Situation, Task, Action, Result).

**Option 1 – Duplicate price updates (KindSwap) – strongest story**
- *Situation:* price service ran on 4–5 instances; each published crypto prices → users saw duplicate updates, extra calls to price provider.
- *Task:* only one publisher, with automatic failover.
- *Action:* built Redis leader election with `SET NX PX`, heartbeat renewal through a Lua script that checks ownership, followers retry, leader stops publishing if renewal fails, CloudWatch logs/alarms.
- *Result:* duplicates eliminated; when the leader instance restarted, another took over within ~10 s *[adjust]*.
Details: [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis).

**Option 2 – Slow user-listing API (4.5 s → 550 ms)** – see [13](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).

**Option 3 – Overselling stock in Rider Perfume / Byte Digital**
- Two users buying the last item at the same time both succeeded (read stock, then save). Fixed with **atomic conditional update** (`stock >= qty` in the update filter) + transaction for multiple items.

**Option 4 – Chat on multiple servers (matrimony)**
- After adding a second instance/PM2 cluster, messages didn't reach users connected to another process. Fixed with **Socket.io Redis adapter** + sticky sessions *[adjust if true]*.

**Option 5 – Double payments on retry** – fixed with idempotency keys + unique gateway payment id ([13](13-resume-questions.md#integrated-5-payment-gateways-with-idempotency-keys)).

---

### 20. Optimisation – how did you improve performance?

> 🎤 **Interview answer:** "I always measure first with response-time logs, then optimise. On the database side: indexes on the filter and sort fields, verified with explain, using compound indexes and the ESR rule, plus projection and `.lean()` so we return only what's needed, and pagination everywhere, cursor-based for chat history. Then Redis caching for product lists, categories and home page data, invalidated on update. Independent queries run in parallel with `Promise.all`, like dashboard counts. Slow work like emails, invoices and SMS moved to background processing, with Kafka at Appsinvo, so APIs return faster. Images are served from S3 and a CDN, resized, never through Node. And at the infrastructure level, gzip at Nginx and PM2 cluster mode to use all CPU cores. The most measurable result was the user-listing API going from 4.5 seconds to 550 milliseconds."

- **Indexes** on filter/sort fields, checked with `explain()` (compound indexes using ESR).
- **Projection + `.lean()`** – return only needed fields; smaller payloads.
- **Pagination** everywhere; cursor-based for chat history.
- **Redis caching** for product lists, categories and home page data; invalidated on update.
- **Parallel queries** with `Promise.all` (e.g. dashboard counts).
- **Background jobs** for emails, invoices, SMS (Kafka at Appsinvo / async jobs) so APIs return faster.
- **Images** stored on S3 (+ CDN) and resized, not served by Node.
- **gzip** via Nginx, **PM2 cluster** to use all CPU cores.
- **Measured** before and after (response time logs).

---

### 21. Security – how did you protect user data?

> 🎤 **Interview answer:** "I think in layers. Credentials: passwords hashed with bcrypt, OTPs hashed with a short TTL. Sessions: short-lived JWTs with refresh tokens in httpOnly cookies, and role checks on admin APIs. Authorisation: ownership checks on every query, so users can only see their own orders and chats. Input: validation with Joi or express-validator *[adjust]* and NoSQL injection protection with type checks and mongo-sanitize. Abuse: rate limiting on login, OTP and search. Transport and headers: HTTPS via Nginx, helmet and strict CORS. Secrets live in environment variables, never in code. Payments: no card data touches our servers, webhook signatures are verified, and prices are always calculated on the server.
>
> For the matrimony app, privacy was especially important: phone numbers stay hidden until an interest is accepted *[adjust]*, exact locations are never exposed, only rounded distance, and users can block and report. Responses never include password hashes or internal fields, and logs never contain passwords, OTPs or tokens."

- Passwords hashed with **bcrypt**; OTPs hashed and short TTL.
- **JWT** with short expiry + refresh token in httpOnly cookie; role checks for admin APIs.
- **Ownership checks** – users can only see their own orders/chats (`{ _id, userId: req.user.id }`).
- **Input validation** (Joi / express-validator *[adjust]*) and NoSQL injection protection (type checks, `express-mongo-sanitize`).
- **Rate limiting** on login/OTP/search APIs.
- **HTTPS** (Nginx + SSL), **helmet** headers, strict **CORS** (only our domain).
- **Secrets** in environment variables, never in code.
- **Payment security** – no card data touches our server (gateway handles it), webhook signature verification, prices calculated on the server.
- **Privacy (matrimony)** – phone numbers hidden until interest is accepted / premium *[adjust]*, exact location never exposed (only rounded distance), users can block/report.
- Responses never include password hashes or internal fields.
- Logs don't contain passwords, OTPs or tokens.

---

## Extra project questions

**Q. What was your exact role / team size?** – Be specific: "I owned the backend; 1 frontend dev and 1 QA *[adjust]*; I talked to the client for requirements."

> 🎤 **Interview answer:** "I owned the backend end to end: requirements discussions with the client, API and schema design, implementation, integrations and deployment. The team was small: me on the backend, one frontend developer and one QA *[adjust]*. I also coordinated directly with the client on requirements and demos."

**Q. If you rebuilt it today, what would you change?** – "TypeScript from day one, add automated tests earlier, use a queue for invoices/emails from the start, cursor pagination for chat from the start, better monitoring/alerts."

> 🎤 **Interview answer:** "Looking back, I'd use TypeScript from day one for safer refactoring, add automated tests earlier instead of relying mostly on manual and Postman testing, use a proper queue for invoices and emails from the start instead of adding it later, use cursor pagination for chat from the beginning, and set up monitoring and alerts on day one. None of these were critical problems, but they would have saved time as the project grew."

**Q. How did you deploy it?** – "AWS EC2, Nginx reverse proxy with SSL, PM2 cluster mode, environment variables on server, MongoDB Atlas *[adjust]*, images on S3. Deploy by pulling code and `pm2 reload` for zero downtime *[adjust – or CI/CD]*."

> 🎤 **Interview answer:** "It runs on AWS EC2. Nginx works as a reverse proxy with SSL, and the Node app runs under PM2 in cluster mode to use all CPU cores. Environment variables hold the configuration and secrets. The database is MongoDB Atlas *[adjust]*, and images and invoices are on S3. To deploy, we pull the latest code, install and build, and run `pm2 reload`, which restarts workers one by one for zero downtime *[adjust – or describe your CI/CD pipeline]*."

**Q. How did you test?** – Jest unit tests for services (mocking DB/gateway), Postman collections for APIs, sandbox mode for payment gateways.

> 🎤 **Interview answer:** "I wrote Jest unit tests for the services and business logic, like price calculation, coupon rules and payment state changes, mocking the database models and the payment gateway. I maintained Postman collections for the APIs to test complete flows, and used the payment gateways' sandbox modes to test checkout and webhooks end to end before going live."

**Q. How many users / requests?** – Matrimony: 25K+ registered, ~2K daily active. KindSwap platform: 100K+ users. Know these numbers.

> 🎤 **Interview answer:** "The matrimony platform has over 25,000 registered users and about 2,000 daily active users. The KindSwap platform, where I built the Referral and Staking services, has over 100,000 users."

**Q. What was the hardest bug?** – Use one of the challenges above.

> 🎤 **Interview answer:** "The hardest one was the duplicate crypto price updates at KindSwap. It only showed up in production, because locally we ran a single instance. Every instance was publishing independently. I found it from CloudWatch logs showing the same update coming from different instance ids, and I fixed it with Redis-based leader election, so only one instance publishes, with automatic failover."

**Q. How did you handle payments failing midway?** – Order stays `pending_payment`; webhook or reconciliation job updates it; stock released after timeout.

> 🎤 **Interview answer:** "The order stays in `pending_payment` until we get a confirmed result. The webhook is the source of truth, and if a webhook is missed, a reconciliation job checks pending payments against the gateway's status API and updates them. If payment fails or doesn't complete within the timeout, the order is cancelled and the reserved stock is released. Because both the webhook handler and the payment creation are idempotent, retries never cause double charges or double stock deductions."

**Q. How do you handle image uploads?** – Multer / pre-signed S3 URL, validate type and size, store S3 key in DB.

> 🎤 **Interview answer:** "For small uploads I use multer with file type and size validation, then upload to S3. For larger files the backend issues a pre-signed S3 URL and the client uploads directly to S3, so the file never passes through our server. In both cases, only the S3 key is stored in the database, and images are served through a CDN."

---

[Home](README.md) | [← 11 Scenario Questions](11-scenario-questions.md) | Next: [13 Resume Deep-Dive →](13-resume-questions.md)
