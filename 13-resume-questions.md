[Home](README.md) | [← 12 Project Questions](12-project-questions.md) | Next: [14 HR & Behavioural →](14-hr-behavioural.md)

# 13 – Resume Deep-Dive (every line → likely questions)

Interviewers read your resume line by line and ask "**How exactly did you do this?**" Every number on your resume (4.5 s → 550 ms, 5 gateways, 4–5 instances, 6 engineers, 25K users) **will** be questioned. Prepare each one below.

> ⚠️ Answers are built from your resume. Where you see *[adjust]*, replace with your real details. If you can't explain a line deeply, practise it until you can – or be ready to describe your exact part honestly.

## Contents
- [Summary line](#summary-line)
- [Notus Services – Air India (current)](#notus-services--air-india-current)
- [Revinfotech – KindSwap](#revinfotech--kindswap)
- [Appsinvo](#appsinvo)
  - [Led and mentored 6 junior developers](#led-and-mentored-6-junior-developers)
  - [Cut a user-listing API from 4.5 s to 550 ms](#cut-a-user-listing-api-from-45-s-to-550-ms)
  - [Introduced Apache Kafka with a dead-letter queue](#introduced-apache-kafka-with-a-dead-letter-queue)
  - [Integrated 5 payment gateways with idempotency keys](#integrated-5-payment-gateways-with-idempotency-keys)
  - [REST APIs and real-time chat (Socket.io, JWT)](#rest-apis-and-real-time-chat-socketio-jwt)
  - [Jest unit tests](#jest-unit-tests)
  - [Deployed on AWS (EC2, S3, Lambda) with Nginx and PM2](#deployed-on-aws-ec2-s3-lambda-with-nginx-and-pm2)
- [Byte Digital Solutions](#byte-digital-solutions)
- [Key projects](#key-projects)
- [Skills section – be ready for each](#skills-section--be-ready-for-each)
- [Questions about your career timeline](#questions-about-your-career-timeline)

---

## Summary line

**"Backend Engineer with 4.5 years designing and building Node.js/TypeScript microservices and distributed backend systems on AWS."**

> 🎤 **Interview answer:** "If they ask about my summary: I've been a backend engineer for about 4.5 years, mainly with Node.js and TypeScript. The microservices I've built include the Referral and Staking services in NestJS at KindSwap, the Kafka-based notification and order-receipt consumers at Appsinvo, and the SAP–WMS integration interfaces at Air India *[adjust]*. By distributed systems I mean real problems like running 4 to 5 instances of a service that needed leader election, services communicating through Redis pub/sub and Kafka, shared state in Redis, and external systems like SAP and payment gateways. And by designing I mean that I decide service boundaries, database schemas, API contracts, sync versus event-based communication, and failure handling with retries, DLQs and idempotency, not just implementing tickets."

- *Q. What microservices have you built?* → Referral and Staking services in NestJS at KindSwap; Kafka-based notification/order-receipt consumers at Appsinvo; the SAP–WMS integration interfaces at Air India *[adjust]*.
- *Q. What makes a system "distributed" in your work?* → Multiple instances of the same service (4–5 price service instances needing leader election), services communicating via Redis pub/sub and Kafka, shared state in Redis, external systems like SAP and payment gateways.
- *Q. What does "designing" mean in your role?* → Deciding the service boundaries, DB schemas, API contracts, how services talk (sync vs events), failure handling (retries, DLQ, idempotency).

---

## Notus Services – Air India (current)

**Bullet: "Built 8 end-to-end backend interfaces syncing data between Air India's SAP and Warehouse Management System (WMS), with validation and logging to prevent data loss in transit."**

> 🎤 **Interview answer:** "At Notus I work on Air India's integration between SAP, their ERP, which holds purchase orders, material and vendor master data and finance, and the Warehouse Management System, which handles the physical receiving, storing and issuing of parts. They must stay in sync: SAP creates a PO, WMS needs it to receive goods, and WMS sends the goods receipt back so SAP updates stock and accounts. I built 8 interfaces covering things like material master, vendor master, purchase orders, goods receipts, goods issues, stock transfers, returns and stock reconciliation *[adjust to real names]*.
>
> Technically, they're Node.js and TypeScript services that read from SAP's interface *[adjust: REST/OData/SOAP/middleware]*, validate and transform the data into the WMS format and send it, plus the reverse direction for GRNs. Every record is stored in PostgreSQL through Prisma with a status lifecycle: received, validated, sent, acknowledged or failed."

**Q. What is SAP and WMS? What data flows between them?**
SAP is Air India's ERP – it holds purchase orders (POs), materials/parts master data, vendors, and financial records. WMS manages the physical warehouse – receiving goods, storing, issuing parts. They must stay in sync: e.g. SAP creates a PO → WMS needs it to receive goods → WMS confirms receipt (GRN) → SAP updates stock/accounts.

**Q. What were the 8 interfaces?** *[adjust to real names]* e.g.:
1. Material master (SAP → WMS)
2. Vendor master (SAP → WMS)
3. Purchase orders (SAP → WMS)
4. Goods receipt / GRN (WMS → SAP)
5. Goods issue (WMS → SAP)
6. Stock transfer
7. Returns to vendor
8. Stock/inventory reconciliation

**Q. How does the integration work technically?** *[adjust: REST/OData/SOAP/files/middleware]*
- Node.js + TypeScript services; data read from SAP's interface (e.g. REST/OData/SOAP/IDoc via middleware or file drop *[adjust]*), transformed to WMS format and sent; reverse direction for GRN.
- Each record stored in PostgreSQL (via Prisma) with a status: `received → validated → sent → acknowledged / failed`.
- Scheduled jobs and/or triggers to pull/push data *[adjust]*.

**Q. "Prevent data loss in transit" – how exactly?**

> 🎤 **Interview answer:** "There are six things. First, validation before sending: mandatory fields, quantities, units, PO exists. Invalid records go to a failed state with a clear reason instead of breaking silently. Second, persist first, then send: every record is saved in our database before calling the other system, so nothing is lost if a call fails midway. Third, logging of every request and response with business references like the PO or GRN number, so any record can be traced. Fourth, automatic retries with backoff for temporary failures, and after the maximum attempts the record is marked failed and we notify. Fifth, idempotent processing with upserts by business reference, so a re-sent record never creates duplicates. And sixth, reconciliation between the systems *[adjust if done]*."

- **Validation** before sending (mandatory fields, quantities, units, PO exists) → invalid records go to a failed state with a reason instead of silently breaking.
- **Persist first, then send** – every record is saved in our DB before calling the other system, so nothing is lost if a call fails.
- **Logging** of every request/response with reference numbers (PO number, GRN number).
- **Retries** with backoff for temporary failures; after max retries mark failed + notify.
- **Idempotent processing** – upsert by unique business reference so a re-sent record doesn't create duplicates.
- **Reconciliation** – compare records between systems *[adjust if done]*.

**Q. Why PostgreSQL here instead of MongoDB?** Data is highly relational (PO → PO lines → GRN → GRN lines → materials → vendors), needs strict consistency and transactions. SQL is the natural fit.

> 🎤 **Interview answer:** "Because this data is strongly relational: a PO has lines, GRNs reference PO lines, and everything links to materials and vendors. It needs foreign keys, strict consistency and multi-table transactions, for example creating a GRN and updating the PO's received quantity together. That's exactly what a relational database is built for. I used MongoDB in projects where the data was document-shaped, but here SQL was the natural fit."

**Q. Why Prisma?** Type-safe queries generated from schema (fits TypeScript), easy migrations, readable code, `$transaction` support.

> 🎤 **Interview answer:** "Prisma generates a fully typed client from the schema file, which fits our TypeScript codebase perfectly: wrong field names or types fail at compile time. It also gives clean, versioned migrations, very readable query code, and `$transaction` for multi-step writes like GRN creation. For the rare complex report query, we can still drop down to raw SQL."

**Bullet: "Built a Next.js GRN dashboard for goods-receipt entry against SAP purchase orders, with validation, log monitoring, notifications and analytics."**

> 🎤 **Interview answer:** "A GRN, goods receipt note, is the record that goods ordered on a PO were physically received: what, how much and in what condition. I built a Next.js dashboard where warehouse users enter GRNs against SAP purchase orders. The validations are that received quantity can't exceed the pending PO quantity, the PO must be open, mandatory fields must be present, and batch or serial numbers are required for parts *[adjust]*. These checks run on the frontend for UX, and on the backend as the real check.
>
> To stop two users over-receiving the same PO line at the same time, the backend uses a conditional update, only incrementing received if received plus quantity is less than or equal to ordered, inside a transaction. The dashboard also has interface log monitoring, which shows which records succeeded or failed, with the reasons and a retry option, notifications for failed syncs or pending GRNs, and analytics like GRNs per day, pending POs and failure rates *[adjust]*. I chose Next.js for routing, fast server-rendered pages and API routes in one React codebase."

- *Q. What is a GRN?* Goods Receipt Note – a record that goods ordered in a PO were physically received (what, how much, condition).
- *Q. What validations?* Received quantity can't exceed pending PO quantity, PO must be open, mandatory fields, batch/serial numbers for parts *[adjust]*; both on frontend (UX) and backend (real check).
- *Q. How do you prevent two users receiving more than the PO quantity at the same time?* DB transaction + row lock or conditional update: `UPDATE po_lines SET received = received + $q WHERE id = $id AND received + $q <= ordered`.
- *Q. Log monitoring?* Dashboard page showing interface logs – which records succeeded/failed, error reasons, retry option *[adjust]*.
- *Q. Notifications?* Alerts on failed syncs or pending GRNs (email/in-app) *[adjust]*.
- *Q. Analytics?* Counts of GRNs per day, pending POs, failure rates *[adjust]*.
- *Q. Why Next.js?* Routing, server-side rendering for fast loads, API routes, one React codebase. See [05 → Next.js](05-react.md#nextjs-basics).

---

## Revinfotech – KindSwap

**Bullet: "Built 2 NestJS microservices (Referral, Staking) with Auth0 RBAC, batch payouts via a multisig wallet and Redis pub/sub between services."**

**Q. Explain the Referral service.**

> 🎤 **Interview answer:** "Every user gets a unique referral code. When a referred user signs up or does a qualifying action like trading or staking *[adjust]*, the referrer earns a reward. The service tracks referral relationships, qualifying events, reward calculation and pending versus paid rewards. The important integrity rules are enforced at the database level: a unique index on the referral code, preventing self-referral, and a unique constraint on referrer, referee and event type, so the same event can never be rewarded twice, even if it's delivered twice."

Users get a referral code; when a referred user signs up/trades/stakes *[adjust]*, the referrer earns a reward. Service tracks referral relationships, qualifying events, reward calculation, and pending/paid rewards.
Key points: unique referral code index, prevent self-referral and duplicate rewards (unique constraint on `(referrerId, refereeId, eventType)`), rewards calculated from events.

**Q. Explain the Staking service.**

> 🎤 **Interview answer:** "In the Staking service, users lock tokens in a pool for a period to earn rewards. The service manages pools, user stakes with amount, start time and lock period, reward calculation, and unstake and claim requests *[adjust]*. One critical detail: amounts are stored as high-precision numeric values or strings, never JavaScript floats, because crypto amounts have up to 18 decimals and floating-point errors would mean real money errors."

Users lock tokens for a period to earn rewards. Service manages pools, user stakes (amount, start, lock period), reward calculation, and unstake/claim requests *[adjust]*. Amounts stored as `numeric` / strings (big decimals), never JS floats.

**Q. What is Auth0 RBAC and how did you use it?**

> 🎤 **Interview answer:** "Auth0 is a hosted identity provider, and RBAC means role-based access control. Roles and permissions, like `payout:approve` or `admin`, are defined in Auth0 and included in the access token. In our NestJS services, a JWT guard verifies the token's RS256 signature using Auth0's public JWKS keys, plus the audience and issuer. Then a custom permissions guard, driven by a `@Roles` decorator, checks the required permission on each route. So no service handles passwords, and access is managed centrally."

Auth0 is a hosted login/identity provider. RBAC = role-based access control. Roles and permissions (e.g. `payout:approve`, `admin`) are defined in Auth0 and added to the access token. Our NestJS services verify the token (RS256 via Auth0's public JWKS key) in a JWT guard, then a **Roles/Permissions guard** checks the required permission on each route. Code in [08 → Guards](08-nestjs.md#7-guards-and-rbac-with-auth0).

**Q. What is a multisig wallet and "batch payouts"?**

> 🎤 **Interview answer:** "A multisig wallet needs approval from M of N owners, for example 2 of 3, before a transaction executes, so no single person or single compromised key can move company funds. For batch payouts *[adjust]*, a scheduled job calculates the referral and staking rewards due and creates a payout batch, a list of addresses and amounts, in the database with status pending. That batch is proposed as a transaction to the multisig wallet, the required signers approve it, and once it executes on-chain, we store the transaction hash and mark each payout as paid. Failed ones are retried in the next batch. Batching means fewer on-chain transactions, so lower fees and fewer approvals. To avoid double payouts, each reward can belong to only one batch, enforced by status and a unique constraint, and processing is idempotent by transaction hash."

- A **multisig (multi-signature) wallet** needs approval from **M of N** owners (e.g. 2 of 3) before a transaction executes. Protects company funds – no single person or single compromised key can move money.
- **Batch payout flow** *[adjust]*:
  1. A scheduled job calculates rewards due (referral/staking) and creates a **payout batch** (list of user addresses + amounts) in DB with status `pending`.
  2. The batch transaction is proposed to the multisig wallet.
  3. Required signers approve.
  4. Once executed on-chain, we track the transaction hash and mark each payout `paid`; failed ones are retried in the next batch.
- Why batch? Fewer on-chain transactions → lower fees and fewer approvals.
- Avoid double payout: each reward row can be in only one batch (status + unique constraint); idempotent processing by tx hash.

**Q. How did the two services communicate? Why Redis pub/sub?**

> 🎤 **Interview answer:** "They communicated with events over Redis pub/sub. For example, when a stake was created, the Staking service published an event, and the Referral service subscribed and processed referral rewards *[adjust events]*. We chose Redis because it was already in our stack, it's very low latency and simple to use.
>
> If they ask about the limitation, I'm upfront: Redis pub/sub doesn't persist messages, so if a subscriber is down, it misses events. We handled that by also storing the events in the database and running a periodic reconciliation job *[adjust]*. If I were designing it today for critical reward flows, I'd use Redis Streams or Kafka with consumer groups and acknowledgements."

Events like `stake.created` were published on Redis channels and the other service subscribed *[adjust events]*. Redis was already in the stack, low latency, simple.
**Follow-up – "Redis pub/sub loses messages if a service is down. How did you handle that?"** → Honest answer options: the events were also stored in DB and a periodic job reconciled missed ones *[adjust]*; or "for critical flows I would use Redis Streams or Kafka with consumer groups and acks". See [09 → Redis pub/sub](09-kafka-redis-messaging.md#redis-pubsub).

**Bullet: "Designed Redis-based leader election so only one of 4–5 instances publishes real-time crypto price updates, eliminating duplicates; monitored with AWS CloudWatch."**

> 🎤 **Interview answer:** "The price service ran on 4 to 5 instances, and each one was publishing price updates, which caused duplicates for users and extra calls to the price provider. Each instance tries `SET leader myId NX PX 10000`, so only one succeeds and becomes leader. The leader renews the key every 3 seconds with a Lua script that first checks it's still the owner. Others keep trying, so if the leader crashes, the key expires and another instance takes over in about 10 seconds. If the leader fails to renew, it stops publishing immediately. Prices come from a price provider *[adjust]*, and the leader pushes updates to clients over Socket.io *[adjust]*. In CloudWatch we logged leadership changes, errors and publish rate, with alarms when there was no leader for too long or errors spiked *[adjust]*."

Full answer + follow-ups in [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis). Short version:
"Each instance tries `SET leader <myId> NX PX 10000`. Only one succeeds and becomes leader; it renews the key every 3 seconds using a Lua script that checks it is still the owner. Others keep trying. If the leader crashes the key expires and another instance takes over in about 10 seconds. If the leader fails to renew, it stops publishing immediately. We logged leadership changes and publish counts to CloudWatch and set alarms."

- *Q. Where did prices come from and how did they reach users?* From a price provider API/websocket *[adjust]*; the leader publishes updates to clients via Socket.io/websocket *[adjust]* (and/or Redis pub/sub to other instances that hold client connections).
- *Q. What did you monitor in CloudWatch?* Logs for leader changes, errors, publish rate; alarms when no leader for > X seconds or error spikes *[adjust]*.

---

## Appsinvo

### Led and mentored 6 junior developers

> 🎤 **Interview answer:** "At Appsinvo I led and mentored 6 junior developers. My approach was practical. I did code reviews where I explained why, not just what to change. I pair-programmed on tough tasks, broke big features into smaller, clear tasks, set shared standards for folder structure, error handling and naming, helped with debugging, and ran short knowledge-sharing sessions *[adjust]*. If a junior kept making the same mistake, I explained the reason behind the fix, gave a small example or checklist, paired with them once, reviewed again, and appreciated the improvement. I was also involved in client calls: gathering requirements, clarifying edge cases, giving estimates, demoing features and handling change requests."

- *Q. How did you mentor?* Code reviews with explanations, pair programming on tough tasks, broke features into smaller tasks, shared coding standards (folder structure, error handling, naming), helped with debugging, conducted short knowledge sessions *[adjust]*.
- *Q. What did you look for in code reviews?* See [11 → E18](11-scenario-questions.md#e18-code-review--what-do-you-check).
- *Q. A junior keeps making the same mistake – what do you do?* Explain the reason (not just the fix), give a small example/checklist, pair once, then review again; appreciate improvement.
- *Q. How did client calls go?* Gathered requirements, clarified edge cases, gave estimates, demoed features, handled change requests.

### Cut a user-listing API from 4.5 s to 550 ms

> 🎤 **Interview answer:** "This is the story I'm proudest of on the performance side. The admin user-listing API, with filters like status, date range, search and role *[adjust]*, was taking around 4.5 seconds, and it was getting worse as the user base grew. I logged the timing of each step and ran the query with `explain('executionStats')`. It was doing a full collection scan because no index matched the filters, it returned full user documents with nested data, and part of the filtering was happening in Node after loading the data *[adjust]*.
>
> I fixed it in four steps. I added compound indexes matching the filters and sort, following the ESR rule, so the query moved to an index scan. I used projection to return only the fields the list needed, with `.lean()`, so the payload was much smaller. I moved all filtering and pagination into the database query. And I ran the count and data queries in parallel *[adjust]*. The result was about 550 milliseconds, around 88% faster, with lower database CPU. I verified it by comparing response times before and after on the same data volume. If they ask why not just cache: the filters vary a lot and the data changes often, so fixing the query and indexes was the right first step, and caching could be added later for common filter combinations."

**Story (practise this – it's your best number):**
- *Situation:* admin user-listing API with filters (status, date range, search, role *[adjust]*) took ~4.5 s; it was slower as users grew.
- *How I found the cause:* logged timings for each step and used `explain("executionStats")` – the query was doing a **collection scan** (no matching index), the API returned **full user documents** with nested data, and some **filtering happened in Node after loading data** *[adjust]*.
- *What I changed:*
  1. Added **compound indexes** matching the filters and sort (ESR rule) – query moved to IXSCAN.
  2. **Projection** – returned only fields the list needed (name, email, status, createdAt) and used `.lean()` → much smaller payload.
  3. Moved **filtering and pagination into the DB query** instead of JS.
  4. Ran count and data queries in parallel / with `$facet` *[adjust]*.
- *Result:* ~**550 ms** (88% faster), less DB CPU.
- *Follow-ups:* "Why not cache?" → filters vary a lot and data changes often, so indexes and query fixes were the right first step; caching could be added for common filter combinations. "How did you verify?" → compared response times before/after in logs/Postman under the same data volume.

### Introduced Apache Kafka with a dead-letter queue

> 🎤 **Interview answer:** "Before Kafka, notifications and order receipts were sent inside the order API request, so responses were slow, and if the email or SMS provider failed, either the order request failed or the message was lost. I introduced Kafka: the order API publishes an event like `order.placed` and returns, and consumers handle notifications and receipts asynchronously *[adjust topic names]*. Events are keyed by orderId so all events for one order stay in order.
>
> On failure, a consumer retries a few times with backoff via a retry topic, and after the maximum attempts it publishes the message to a DLQ topic with the error details, commits the offset so the main flow continues, and we monitor and replay from the DLQ. To avoid duplicate emails, consumers are idempotent: the event id is checked and stored before sending. We chose Kafka over RabbitMQ or SQS because multiple consumers needed the same events and replay was useful *[adjust honestly]*."

- *Q. Why Kafka?* Notifications and order receipts were sent **inside the API request** → slow responses and if email/SMS provider failed, the order API failed or the message was lost. Moving them to Kafka made the order API faster and made sending reliable with retries.
- *Q. Topics?* e.g. `order.placed`, `notification.send`, plus `*.retry` and `*.dlq` *[adjust]*.
- *Q. How does your DLQ work?* Retry N times with backoff → then publish to DLQ topic with error details → commit offset → monitor and replay. See [09 → DLQ](09-kafka-redis-messaging.md#retries-and-dead-letter-queue-dlq).
- *Q. How did you avoid duplicate emails?* Consumers are idempotent – event id stored/checked before sending.
- *Q. Partitions and keys?* Key by `orderId`/`userId` so events for one order stay in order *[adjust partition count]*.
- *Q. Why Kafka over RabbitMQ/SQS?* *[adjust honestly]* e.g. multiple consumers needed the same events (notification + receipts + analytics), replay ability, team/infra already had it. See [comparison](09-kafka-redis-messaging.md#kafka-vs-rabbitmq-vs-sqs).

### Integrated 5 payment gateways with idempotency keys

> 🎤 **Interview answer:** "I integrated Razorpay, Stripe, Cashfree, MyFatoorah and Tamara, because different clients and regions needed different ones: Razorpay and Cashfree for India, Stripe for international cards, MyFatoorah for the Middle East, and Tamara for buy-now-pay-later in the Gulf *[adjust]*. To keep the code clean, I defined a common PaymentGateway interface, with createOrder, verifyPayment, handleWebhook, refund and getStatus, and one adapter class per gateway, with a factory choosing the gateway by config, country or currency. The business code never knows which gateway is used.
>
> For idempotency, each payment attempt has a unique key sent in a header and stored with a unique constraint, together with the result, so a retry with the same key returns the stored result instead of charging again. Stripe supports this natively, and for the others we enforced it on our side *[adjust]*. Webhooks are verified with HMAC signatures, de-duplicated with a unique index on the gateway's payment id, acknowledged quickly and processed async, with a reconciliation job for pending payments. Refunds are also idempotent by refund reference, and amounts are stored in the smallest currency unit. Some Gulf currencies, like KWD, have 3 decimals, which is an easy thing to get wrong."

Razorpay, Stripe, Cashfree, MyFatoorah, Tamara.
- *Q. Why 5?* Different clients/regions: Razorpay & Cashfree for India, Stripe for international cards, MyFatoorah for Middle East (Kuwait/Saudi), Tamara for buy-now-pay-later in the Gulf *[adjust]*.
- *Q. How did you keep code clean with 5 gateways?* A common `PaymentGateway` interface (`createOrder`, `verifyPayment`, `handleWebhook`, `refund`, `getStatus`) with one adapter class per gateway; a factory picks the gateway by config/country/currency. Business code doesn't know which gateway is used.
- *Q. What exactly is the idempotency key and where is it stored?* Unique key per payment attempt (UUID from client or generated from `orderId + attempt`), sent in header; stored in DB/Redis with unique constraint and the result. A retry with the same key returns the stored result instead of creating a new charge. Stripe also supports an `Idempotency-Key` header natively; for others we enforced it on our side *[adjust]*. Details in [07 → Idempotency](07-apis-backend.md#q-what-is-idempotency-and-how-do-you-implement-it).
- *Q. Webhook handling?* Verify signature (HMAC), unique index on gateway payment/event id, respond 200 fast, process async, reconcile pending payments with a status API job.
- *Q. Refunds?* Refund API per gateway; store refund records; idempotent by refund reference.
- *Q. Currency/amount?* Stored in smallest unit (paise/fils/cents); note some currencies (KWD) have 3 decimals.

### REST APIs and real-time chat (Socket.io, JWT)

> 🎤 **Interview answer:** "For the chat, the JWT is verified in Socket.io's handshake middleware, `io.use`, so unauthenticated sockets never connect. Each user joins a room named after their user id. Messages are saved to the database before being emitted to the receiver's room, so nothing is lost, with delivered and read acknowledgements, and history loaded with cursor pagination. To run on multiple instances, we used the Socket.io Redis adapter *[adjust]*."

- JWT verified in Socket.io handshake middleware (`io.use`), user joins room = their userId, messages saved to DB before emit, delivered/read acks, history via cursor pagination, Redis adapter for multiple instances *[adjust]*. See [10 → Chat](10-system-design.md#design-a-chat-system).

### Jest unit tests

> 🎤 **Interview answer:** "I wrote Jest unit tests mainly for services and business logic, like price calculations, coupon rules and payment state transitions, plus utilities and middleware. Database models and payment gateways were mocked with `jest.mock`, so tests were fast and didn't depend on external systems *[adjust]*. Unit tests check one piece in isolation, and for flows across layers we used integration tests with supertest and Postman collections."

- *Q. What did you test?* Services/business logic (price calc, coupon rules, payment state changes), utils, middleware; mocked DB models and gateways with `jest.mock` *[adjust]*.
- *Q. Unit vs integration?* See [15 → Jest](15-testing-git-devops.md#jest-and-testing).

### Deployed on AWS (EC2, S3, Lambda) with Nginx and PM2

> 🎤 **Interview answer:** "Node apps ran on EC2 under PM2 in cluster mode, which gave us auto-restart, logs, a startup script after reboot, and `pm2 reload` for zero-downtime deploys. Nginx sat in front for SSL *[adjust: Let's Encrypt or ACM]*, reverse proxying, gzip and the WebSocket upgrade headers needed for Socket.io. S3 stored images and invoices, accessed through pre-signed URLs, and Lambda handled *[adjust: e.g. image resizing on upload or scheduled jobs]*."

- EC2 runs Node with **PM2** (cluster mode, auto-restart, logs, `pm2 reload` for zero downtime, startup script after reboot).
- **Nginx** in front: SSL (Let's Encrypt/ACM *[adjust]*), reverse proxy, gzip, websocket upgrade headers for Socket.io.
- **S3** for images/invoices with pre-signed URLs.
- **Lambda** for *[adjust: e.g. image resize on upload, scheduled jobs]*.
- See [15 → PM2 & Nginx](15-testing-git-devops.md#pm2-and-nginx).

---

## Byte Digital Solutions

**Bullet: "Built a business website and internal dashboard end to end (Node.js, Express.js, React.js, MongoDB), with automated receipt emails, notifications and stock recalculation on each booking."**

> 🎤 **Interview answer:** "At Byte Digital, my first company, I built a business website and an internal dashboard end to end with the MERN stack for *[adjust – describe the business]*. On each booking, stock was recalculated with an atomic `$inc` that has a condition of stock greater than or equal to the quantity, so concurrent bookings couldn't oversell, using a transaction when multiple items were involved, and stock was restored on cancellation. Receipt emails were sent with HTML templates *[adjust: Nodemailer/SES/SendGrid]* after the booking was saved. Ideally that should be async, so the booking API isn't slowed down, which is a lesson I applied later with Kafka. The biggest learning from that job was owning features end to end and talking directly to the business owner."

- *Q. What business?* *[adjust – know the domain]*
- *Q. Stock recalculation on each booking – how? Race conditions?* Atomic `$inc` with condition `stock >= qty`, or transaction when multiple items; restore stock on cancellation.
- *Q. How were receipt emails sent?* Nodemailer/SES/SendGrid with HTML templates *[adjust]*, sent after booking is saved (ideally async so booking API isn't slowed).
- *Q. What did you learn from your first job?* Full-stack basics, owning features end to end, talking to the business owner.

---

## Key projects

**Veer Maratha Matrimony** – 25K+ registered / 2K DAU. Questions:

> 🎤 **Interview answer:** "Veer Maratha Matrimony has over 25,000 registered users and about 2,000 daily active users. 'Near me' search uses a 2dsphere index with `$geoNear`, coordinates in longitude, latitude order, combined with preference filters. Matchmaking filters stay fast with compound indexes on the common filters like gender, date of birth for age, religion and caste, plus pagination and preference-based defaults. Premium plans are paid through a payment gateway with webhook verification, and SMS is used for OTP and alerts *[adjust providers]*."

- *How does geo-search work?* `2dsphere` index + `$near`/`$geoNear` with `[lng, lat]`, combined with filters. [04 → Geo query](04-mongodb-sql.md#extra-mongodb-questions)
- *Matchmaking filters – how did you make them fast?* Compound indexes on common filters (gender, age via dob, religion/caste), pagination, preference-based defaults.
- *Payment/SMS integration?* Payment gateway for premium plans with webhook verification; SMS for OTP/alerts *[adjust providers]*.
- *2K DAU – any scaling issues?* *[adjust: e.g. chat load, search speed]*.

**Rider Perfume** – full backend: orders, inventory, OTP auth, caching, pagination, rate limiting, invoices. Full answers in [12 Project Questions](12-project-questions.md).

> 🎤 **Interview answer:** "For Rider Perfume, I built the complete backend: orders, inventory, OTP authentication, caching, pagination, rate limiting and automated invoices. Invoices are generated automatically after payment succeeds: a PDF is created *[adjust: pdfkit/puppeteer]* with the order details and GST, stored in S3, and emailed to the customer."

- *How were invoices automated?* After payment success, generate PDF (pdfkit/puppeteer *[adjust]*) with order details and GST, store in S3, email link/attachment.

---

## Skills section – be ready for each

Anything on your resume is fair game. Quick check:

| Skill | Be ready to answer | Where |
|---|---|---|
| TypeScript | Generics, utility types, TS with Express | [06](06-typescript.md) |
| NestJS | DI, guards, pipes, interceptors, lifecycle | [08](08-nestjs.md) |
| Microservices | Communication, saga, failure handling | [10](10-system-design.md#monolith-vs-microservices) |
| Socket.io | Rooms, auth, scaling with Redis adapter | [07](07-apis-backend.md#q-how-does-socketio-scale-across-many-servers) |
| Swagger | How you documented APIs | [08](08-nestjs.md#q-how-do-you-add-swagger) |
| PostgreSQL / MySQL | Joins, transactions, indexes, isolation | [04](04-mongodb-sql.md#sql) |
| Prisma / TypeORM | Pros/cons, transactions, migrations | [04](04-mongodb-sql.md#orms--prisma-typeorm-mongoose) |
| Kafka / RabbitMQ / SQS | Differences and when to use | [09](09-kafka-redis-messaging.md#kafka-vs-rabbitmq-vs-sqs) |
| AWS | EC2, S3, Lambda, RDS, CloudWatch | [07](07-apis-backend.md#aws-services-on-your-resume) |
| Docker / Nginx / PM2 / Linux | Dockerfile, reverse proxy, commands | [07](07-apis-backend.md#8-docker-basics), [15](15-testing-git-devops.md) |
| Jest | Mocks, unit vs integration | [15](15-testing-git-devops.md#jest-and-testing) |
| System design, caching, idempotency, rate limiting, DLQ | | [10](10-system-design.md), [09](09-kafka-redis-messaging.md) |
| React / Next.js | Hooks, SSR vs CSR | [05](05-react.md) |

**RabbitMQ and SQS** are in skills but not in any bullet → expect *"Where did you use RabbitMQ/SQS?"* Have a clear answer *[adjust: which project, what for]*. If it was small usage, say so honestly and explain the concepts well.

---

## Questions about your career timeline

> 🎤 **Interview answer:** "For timeline questions I keep it honest, positive and short. For the short stays, I give a genuine reason focused on growth or the project, without blaming anyone *[prepare your real reason]*. For doing the MCA while working, I explain how I managed my time *[adjust]*, and that it strengthened my fundamentals in DBMS, operating systems, networking and DSA. And I'm always clear and truthful about my current notice period and availability."

- *"You've been at Notus since Sep 2026 and Revinfotech was May–Aug 2026. Why short stays?"* → Prepare an honest, positive answer – see [14 → Job changes](14-hr-behavioural.md#why-did-you-change-jobs--short-tenure).
- *"You did MCA (2024–2025) while working full-time. How?"* → Weekend/distance classes *[adjust]*, time management; it strengthened fundamentals (DBMS, OS, networking, DSA).
- *"Bachelor's 2021–2023 and job from May 2022?"* → Worked while finishing the degree *[adjust]*.
- *"Immediate joiner – are you currently serving notice?"* → Be clear and truthful about your status.

---

[Home](README.md) | [← 12 Project Questions](12-project-questions.md) | Next: [14 HR & Behavioural →](14-hr-behavioural.md)
