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

> 🎤 **Interview answer:** "Rider Perfume is a live e-commerce website for a perfume and beauty brand: riderbeauty.com. Before this, the brand sold mostly offline or through marketplaces *[adjust]*, and wanted its own online store to sell directly, manage stock and send invoices automatically.
>
> I built the **complete backend**: product catalogue, cart and orders, stock management, OTP login, payments, automatic invoices, caching, pagination and rate limiting. The frontend *[adjust: team / I]* used my REST APIs.
>
> The main goals were: correct stock (never sell more than we have), fast product pages, and a secure checkout."

**Veer Maratha Matrimony:**

> 🎤 **Interview answer:** "It's a matrimony platform for the Maratha community: pairsindia.in. Users create profiles, search for matches by preferences and location, send interests, chat in real time, and buy premium plans. I owned the backend. It has **25K+ registered users and about 2K daily active users**.
>
> Key features I built: 'matches near me' location search using MongoDB, match filters, real-time chat with Socket.io, payment and SMS integration, and deployment on AWS."

**Tip:** Say *problem → users → your role → 3 main features → result/scale*.

---

### 14. Architecture – how does the frontend communicate with the backend?

> 🎤 **Interview answer:** "The frontend (React / Next.js, or a mobile app) talks to the backend using **REST APIs with JSON over HTTPS**. For real-time chat, it keeps a **Socket.io** connection open.
>
> Every request first reaches **Nginx**, which handles HTTPS and forwards it to our **Node.js + Express** app, running with PM2 on AWS EC2.
>
> Inside the app, the code is in layers:
> - **Routes** – map URLs to controllers.
> - **Middleware** – rate limiting, JWT check, input validation.
> - **Controllers** – read the request and call a service. Kept thin.
> - **Services** – the business logic. They talk to MongoDB, Redis (cache, OTP), S3 (images, invoices), and third parties (payment gateway, SMS, email).
>
> **Example – placing an order:** the app sends `POST /api/v1/orders` with the token. Middleware runs. The order service safely reduces stock, creates the order and the payment order, and returns 201. Later, the payment gateway's webhook confirms the payment, and the invoice and email are created in the background.
>
> The layers keep controllers clean, and let me test services with Jest."

```
 React / Next.js frontend  (or mobile app)
        │  HTTPS (REST, JSON)  + WebSocket (Socket.io for chat)
        ▼
     Nginx  (HTTPS, gzip, forwards requests)
        ▼
 Node.js + Express API  (PM2 cluster on AWS EC2)
   ├── Routes → Middleware (auth, validation, rate limit) → Controllers → Services
   ├── MongoDB  (main data)
   ├── Redis    (cache, OTP, rate limit) [adjust]
   ├── S3       (product images, invoices)
   └── 3rd party: payment gateway, SMS provider, email
```

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

**Real-time (matrimony chat):** the app opens a Socket.io connection with the JWT. The server puts the user in a room named after their userId. Messages are saved in MongoDB first, then sent to the receiver's room.

---

### 15. Tech stack – why MongoDB, Express, React and Node?

> 🎤 **Interview answer:**
> - "**Node.js** – it doesn't wait on I/O, so it's great for APIs and real-time chat with many open connections. And the same language (JavaScript / TypeScript) on frontend and backend makes development faster.
> - **Express** – small and flexible, with lots of ready middleware for auth, rate limiting and uploads.
> - **MongoDB** – the data was naturally 'document-shaped'. Different perfumes have different attributes, and matrimony profiles have many optional fields and preferences. MongoDB's built-in location search (2dsphere) was a big reason for choosing it for the matrimony 'near me' feature.
> - **React** – component-based UI, good for dashboards.
>
> I'm also honest about the downsides. MongoDB has weaker joins, so for orders and payments we kept documents well structured, used single-step updates for stock, and transactions where many documents change together. When data is very relational, like the Air India SAP–WMS work, I use PostgreSQL with Prisma. I choose the database based on the data, not habit."

---

### 16. Authentication – how did you implement login and registration?

**Rider Perfume – OTP login** *[adjust details]*:

> 🎤 **Interview answer:** "In Rider Perfume we used OTP login.
> 1. The user enters a phone number. I check it, and limit OTP requests (about 3 per 10 minutes per phone and IP).
> 2. I create a 6-digit OTP, save its **hash** in Redis for 5 minutes, and send it by SMS.
> 3. When the user enters the OTP, I compare it. Max about 5 wrong tries. After success I delete it, so it can't be used again.
> 4. A new user is created on first login.
> 5. I return a short-lived access token, and a refresh token in an httpOnly cookie. There are refresh and logout APIs too. Admin routes also check the role."

**Matrimony – email/phone + password and JWT** *[adjust]*:

> 🎤 **Interview answer:** "Registration is by email or phone with a password. I validate the input, check for duplicates with a unique index, hash the password with bcrypt, and verify the phone with OTP. Login returns a JWT. For chat, the same token is checked when the Socket.io connection opens. Users can only edit their own profile, and premium-only features are checked in middleware."

Theory in [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens), full flow in [11 → Q4](11-scenario-questions.md#4-how-would-you-implement-jwt-authentication).

---

### 17. APIs – explain two or three important APIs

> 🎤 **Interview answer:** "I'll explain three from Rider Perfume.
>
> **1. Place order** – `POST /api/v1/orders`
> - Takes cart items, address id, and an `Idempotency-Key` header.
> - I load prices **from the database**. I never trust prices sent by the app.
> - For each item, I reduce stock in **one safe step**: 'reduce only if stock ≥ quantity'. If any item fails, everything is undone and I return 409 'out of stock'.
> - I create the order with status `pending_payment`, create the payment order on the gateway, and return 201.
>
> **2. Payment webhook** – `POST /api/v1/payments/webhook`
> - I check the gateway's signature, to be sure the request is real.
> - If this payment was already processed, I just return 200 (so duplicates are safe).
> - Otherwise I mark the payment and order as paid, and create the invoice and email in the background.
> - If the payment failed or expired, I put the stock back.
>
> **3. Product listing** – `GET /api/v1/products?category=&minPrice=&sort=&page=`
> - Only allowed sort fields, max 50 per page.
> - Indexes for common filters, and only the fields the list needs.
> - Cached in Redis for a few minutes, cleared when products change.
>
> In the matrimony app, the key API is **match search**: a location search with preference filters, cursor pagination, and only a rounded distance in the result."

---

### 18. Database – how did you design your schemas?

> 🎤 **Interview answer:** "For Rider Perfume *[adjust fields]*, the main collections were users, products, orders, payments and coupons. Key decisions:
> - **Order items and address are copied into the order.** So if a product's price or the user's address changes later, old orders and invoices stay correct.
> - **Addresses are inside the user document**, because there are few, and they're always read with the user.
> - **Orders and payments are separate collections**, because they keep growing.
> - **Amounts are saved as whole numbers in paise**, to avoid decimal rounding errors.
> - **Indexes match the queries**: category + price for listings, unique slug, userId + date on orders, and a unique gateway payment id on payments to stop duplicate webhooks.
>
> For the matrimony app, the profile contains its preferences and a few photos, with a location index. Interests, messages and subscriptions are separate collections. Interests have a unique `{from, to}` index, and messages are indexed by conversation and date for chat history."

**Rider Perfume** *[adjust fields]*:
```js
User     { _id, phone (unique), name, email, role: "user"|"admin", addresses: [ {line1, city, pincode, isDefault} ], createdAt }
Product  { _id, name, slug (unique), description, category, brand, price, mrp, stock, images: [String], attributes: { volumeMl, fragrance }, isActive, ratingAvg, ratingCount }
Order    { _id, userId (ref), items: [ { productId, name, price, qty } ], address: {...copy}, totalAmount,
           status: "pending_payment"|"paid"|"shipped"|"delivered"|"cancelled", paymentId, invoiceUrl, createdAt }
Payment  { _id, orderId, gateway, gatewayOrderId, gatewayPaymentId (unique), amount, status, idempotencyKey (unique), raw }
Coupon   { code (unique), type, value, minAmount, expiresAt, usageLimit, usedCount }
```

**Matrimony** – see the profile schema in [04 → Schema design](04-mongodb-sql.md#5-mongodb-schema-design). Plus: `Interest { from, to, status }` with unique `{from, to}`, `Message { conversationId, senderId, text, createdAt, readAt }` with index `{ conversationId: 1, createdAt: -1 }`, `Subscription { userId, plan, validTill, paymentId }`.

---

### 19. Challenges – what technical issue did you face?

> 🎤 **Interview answer (STAR):**
> - "**Situation:** At KindSwap, our price service ran on 4–5 servers, and **every** server fetched crypto prices and sent updates. So users saw **duplicate** price updates, and we called the price provider many more times than needed.
> - **Task:** Make sure only **one** server sends updates, and if it dies, another takes over automatically.
> - **Action:** I built leader election with Redis. Each server tries `SET key myId NX PX 10000`. Only one can succeed, and it becomes the leader. The leader renews the key every 3 seconds, but only if it still owns it. The others keep trying, so if the leader crashes, the key expires and another server takes over within about 10 seconds. If renewal fails, the old leader stops sending immediately. We logged leader changes in CloudWatch, with alerts.
> - **Result:** Duplicate updates were gone, and failover worked automatically when a server restarted *[adjust]*.
>
> I also have other stories ready: the 4.5 s → 550 ms API optimisation, stopping overselling with safe stock updates, scaling chat across servers with the Redis adapter, and stopping double payments with idempotency keys."

Details: [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis).

**Other options:**
- **Slow user-listing API (4.5 s → 550 ms)** – see [13](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).
- **Overselling stock** – two users bought the last item at the same time and both succeeded (read stock, then save). Fixed with a single conditional update (`stock >= qty` in the filter) + a transaction for many items.
- **Chat on multiple servers** – after adding a second server, messages didn't reach users connected to the other one. Fixed with the Socket.io Redis adapter + sticky sessions *[adjust if true]*.
- **Double payments on retry** – fixed with idempotency keys + unique gateway payment id ([13](13-resume-questions.md#integrated-5-payment-gateways-with-idempotency-keys)).

---

### 20. Optimisation – how did you improve performance?

> 🎤 **Interview answer:** "I always **measure first** with response-time logs, then fix.
> - **Database** – indexes on filter and sort fields, checked with `explain()`. Only return needed fields with `.lean()`. Pagination everywhere, cursor-based for chat history.
> - **Redis cache** for product lists, categories and home page data, cleared when data changes.
> - **Parallel queries** with `Promise.all`, like dashboard counts.
> - **Background jobs** for emails, invoices and SMS (Kafka at Appsinvo), so APIs reply faster.
> - **Images** served from S3 + CDN, resized, never through Node.
> - **gzip** at Nginx and **PM2 cluster** to use all CPU cores.
>
> The clearest result: the user-listing API went from 4.5 seconds to 550 ms."

---

### 21. Security – how did you protect user data?

> 🎤 **Interview answer:** "I think in layers:
> - **Passwords** hashed with bcrypt. **OTPs** hashed, with a short expiry.
> - **Short-lived JWTs**, refresh tokens in httpOnly cookies, role checks on admin APIs.
> - **Ownership checks** on every query, so users only see their own orders and chats.
> - **Input validation** with Joi / express-validator *[adjust]*, and protection from NoSQL injection.
> - **Rate limiting** on login, OTP and search.
> - **HTTPS**, helmet headers, strict CORS.
> - **Secrets** in environment variables, never in code.
> - **Payments** – card details never touch our servers, webhook signatures are checked, prices always calculated on the server.
>
> For the matrimony app, privacy was extra important: phone numbers stay hidden until an interest is accepted *[adjust]*, exact locations are never shown (only rounded distance), and users can block and report. Responses never include password hashes, and logs never contain passwords, OTPs or tokens."

---

## Extra project questions

**Q. What was your exact role / team size?**

> 🎤 **Interview answer:** "I owned the backend end to end: requirement discussions with the client, API and database design, coding, integrations and deployment. The team was small: me on backend, one frontend developer and one QA *[adjust]*. I also talked directly to the client for requirements and demos."

**Q. If you rebuilt it today, what would you change?**

> 🎤 **Interview answer:** "I'd use TypeScript from day one, add automated tests earlier instead of mostly manual and Postman testing, use a queue for invoices and emails from the start, use cursor pagination for chat from the beginning, and set up monitoring and alerts on day one. None of these were big problems, but they would have saved time as the project grew."

**Q. How did you deploy it?**

> 🎤 **Interview answer:** "It runs on AWS EC2. Nginx sits in front with HTTPS, and the Node app runs with PM2 in cluster mode to use all CPU cores. Settings and secrets are in environment variables. The database is MongoDB Atlas *[adjust]*, and images and invoices are on S3. To deploy, we pull the latest code, install, build, and run `pm2 reload`, which restarts the app copies one by one, so there's no downtime *[adjust – or describe your CI/CD pipeline]*."

**Q. How did you test?**

> 🎤 **Interview answer:** "I wrote Jest unit tests for the business logic, like price calculation, coupon rules and payment status changes, using fake versions of the database and payment gateway. I kept Postman collections to test full API flows, and used the payment gateways' sandbox mode to test checkout and webhooks before going live."

**Q. How many users / requests?**

> 🎤 **Interview answer:** "The matrimony platform has over 25,000 registered users and about 2,000 daily active users. The KindSwap platform, where I built the Referral and Staking services, has over 100,000 users."

**Q. What was the hardest bug?**

> 🎤 **Interview answer:** "The duplicate crypto price updates at KindSwap. It only happened in production, because locally we ran just one server. Every server was sending updates on its own. I found it in CloudWatch logs, which showed the same update coming from different server ids, and fixed it with Redis leader election, so only one server sends, with automatic failover."

**Q. How did you handle payments failing midway?**

> 🎤 **Interview answer:** "The order stays `pending_payment` until we get a confirmed result. The webhook is the source of truth. If a webhook is missed, a background job checks pending payments with the gateway's status API and updates them. If the payment fails or doesn't finish in time, the order is cancelled and the stock is put back. Since both the payment creation and the webhook handler are safe to repeat, retries never cause double charges or double stock reduction."

**Q. How do you handle image uploads?**

> 🎤 **Interview answer:** "For small uploads, I use `multer`, check type and size, and upload to S3. For bigger files, the backend creates a pre-signed S3 URL and the app uploads directly to S3, so the file never passes through our server. In both cases, only the S3 key is saved in the DB, and images are served through a CDN."

---

[Home](README.md) | [← 11 Scenario Questions](11-scenario-questions.md) | Next: [13 Resume Deep-Dive →](13-resume-questions.md)
