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

**Q. What microservices have you built?**

> 🎤 **Interview answer:** "The Referral and Staking services in NestJS at KindSwap, the Kafka consumers for notifications and order receipts at Appsinvo, and the SAP–WMS integration services at Air India *[adjust]*."

**Q. What makes your work "distributed"?**

> 🎤 **Interview answer:** "Real problems that only happen when there are many servers or many systems. For example: running 4–5 copies of the price service, which needed leader election so only one sends updates; services talking to each other through Redis pub/sub and Kafka; shared data in Redis; and connecting to outside systems like SAP and payment gateways."

**Q. What does "designing" mean in your role?**

> 🎤 **Interview answer:** "It means I decide things, not just complete tickets: how to split the services, the database schema, the API format, whether services talk directly or through events, and how to handle failures with retries, dead-letter queues and idempotency."

---

## Notus Services – Air India (current)

**Bullet: "Built 8 end-to-end backend interfaces syncing data between Air India's SAP and Warehouse Management System (WMS), with validation and logging to prevent data loss in transit."**

**Q. What is SAP and WMS? What data flows between them?**

> 🎤 **Interview answer:** "**SAP** is Air India's main business system (ERP). It holds purchase orders, parts / material details, vendors and finance. **WMS** (Warehouse Management System) handles the actual warehouse: receiving goods, storing them and issuing parts.
>
> They must stay in sync. For example: SAP creates a purchase order → WMS needs it to receive the goods → WMS sends back a goods receipt (GRN) → SAP updates stock and accounts.
>
> I built 8 interfaces for this, like material master, vendor master, purchase orders, goods receipts, goods issues, stock transfers, returns and stock reconciliation *[adjust to real names]*."

**The 8 interfaces** *[adjust to real names]*:
1. Material master (SAP → WMS)
2. Vendor master (SAP → WMS)
3. Purchase orders (SAP → WMS)
4. Goods receipt / GRN (WMS → SAP)
5. Goods issue (WMS → SAP)
6. Stock transfer
7. Returns to vendor
8. Stock reconciliation

**Q. How does the integration work technically?**

> 🎤 **Interview answer:** "They are Node.js + TypeScript services. They read data from SAP *[adjust: REST / OData / SOAP / middleware / file]*, check it, convert it to the format WMS expects, and send it. GRNs go the other way, from WMS to SAP. Every record is saved in PostgreSQL through Prisma, with a status: received → validated → sent → acknowledged, or failed. Scheduled jobs pull and push the data *[adjust]*."

**Q. "Prevent data loss in transit" – how exactly?**

> 🎤 **Interview answer:** "Six things:
> 1. **Check before sending** – required fields, quantities, units, does the PO exist. Bad records are marked 'failed' with a clear reason, instead of breaking silently.
> 2. **Save first, then send** – every record is saved in our DB **before** calling the other system. So if a call fails in the middle, nothing is lost.
> 3. **Log everything** – every request and response, with the PO or GRN number, so any record can be traced.
> 4. **Automatic retries** for temporary failures, waiting longer each time. After the max tries, mark it failed and notify.
> 5. **Safe to resend** – the receiving side uses 'update if exists, else create' by PO / GRN number, so sending a record again never creates duplicates.
> 6. **Regular cross-check** between the two systems *[adjust if done]*."

**Q. Why PostgreSQL here instead of MongoDB?**

> 🎤 **Interview answer:** "Because this data is **strongly connected**: a PO has lines, GRNs point to PO lines, and everything links to materials and vendors. It needs foreign keys, strict correctness, and transactions across tables, like creating a GRN and updating the PO's received quantity together. That's exactly what a SQL database is built for. I used MongoDB where data was document-shaped, but here SQL was the natural choice."

**Q. Why Prisma?**

> 🎤 **Interview answer:** "Prisma creates fully typed database code from the schema file, which fits our TypeScript code perfectly. A wrong field name fails while writing code, not in production. It also gives clean migrations, very readable queries, and `$transaction` for multi-step writes like GRN creation. For a rare complex report, we can still write raw SQL."

**Bullet: "Built a Next.js GRN dashboard for goods-receipt entry against SAP purchase orders, with validation, log monitoring, notifications and analytics."**

**Q. What is a GRN, and what does the dashboard do?**

> 🎤 **Interview answer:** "A **GRN (Goods Receipt Note)** is the record that goods ordered on a PO have actually arrived: what, how many, and in what condition.
>
> I built a Next.js dashboard where warehouse users enter GRNs against SAP purchase orders.
> - **Checks:** received quantity can't be more than the pending PO quantity, the PO must be open, required fields must be filled, batch / serial numbers for parts *[adjust]*. These run on the frontend for a good experience, and on the backend as the real check.
> - **Two users at the same time:** to stop two users receiving more than the PO quantity at the same moment, the backend uses a single conditional update inside a transaction: 'add to received only if received + qty ≤ ordered'.
> - **Log monitoring:** a page that shows which records succeeded or failed, with the reason and a 'Retry' button.
> - **Notifications** for failed syncs or pending GRNs, and **analytics** like GRNs per day, pending POs and failure rates *[adjust]*.
>
> I chose Next.js because it gives routing, fast server-rendered pages and API routes in one React codebase."

```sql
UPDATE po_lines
SET received = received + $qty
WHERE id = $id AND received + $qty <= ordered;
-- 0 rows updated → would go over the PO quantity → reject
```

See [05 → Next.js](05-react.md#nextjs-basics).

---

## Revinfotech – KindSwap

**Bullet: "Built 2 NestJS microservices (Referral, Staking) with Auth0 RBAC, batch payouts via a multisig wallet and Redis pub/sub between services."**

**Q. Explain the Referral service.**

> 🎤 **Interview answer:** "Every user gets a unique referral code. When someone signs up with that code and does a qualifying action, like trading or staking *[adjust]*, the person who referred them earns a reward. The service tracks who referred whom, the qualifying events, the reward amount, and whether it's pending or paid.
>
> The important rules are enforced **in the database**: a unique referral code, no self-referral, and a unique rule on (referrer, referee, event type), so the same event can never be rewarded twice, even if the event arrives twice."

**Q. Explain the Staking service.**

> 🎤 **Interview answer:** "Users lock their tokens in a pool for some time to earn rewards. The service manages pools, each user's stake (amount, start time, lock period), reward calculation, and unstake / claim requests *[adjust]*. One critical detail: amounts are saved as high-precision numbers or strings, **never** JavaScript decimal numbers. Crypto amounts can have 18 decimal places, and small rounding errors would mean real money errors."

**Q. What is Auth0 RBAC and how did you use it?**

> 🎤 **Interview answer:** "Auth0 is an outside service that handles login for us. RBAC means **role-based access control**. Roles and permissions, like `payout:approve` or `admin`, are set up in Auth0, and Auth0 puts them inside the user's token. In our NestJS services:
> 1. A JWT guard checks the token is real, using Auth0's public key.
> 2. A permissions guard, with a `@Roles()` decorator, checks if the user has the permission needed for that route.
>
> So our services never handle passwords, and access is managed in one place."

Code in [08 → Guards](08-nestjs.md#7-guards-and-rbac-with-auth0).

**Q. What is a multisig wallet and "batch payouts"?**

> 🎤 **Interview answer:** "A **multisig wallet** needs approval from several owners, for example 2 out of 3, before any money can move. So no single person, or single stolen key, can move company funds.
>
> **Batch payouts** *[adjust]*:
> 1. A scheduled job calculates the referral and staking rewards that are due, and creates a **payout batch** (list of wallet addresses + amounts) in the DB with status 'pending'.
> 2. The batch is sent to the multisig wallet as one transaction.
> 3. The required people approve it.
> 4. Once it runs on the blockchain, we save the transaction hash and mark each payout 'paid'. Failed ones go into the next batch.
>
> **Why batch?** Fewer blockchain transactions → lower fees and fewer approvals.
> **No double payouts:** each reward can be in only one batch (status + unique rule), and processing is checked by the transaction hash."

**Q. How did the two services communicate? Why Redis pub/sub?**

> 🎤 **Interview answer:** "They sent events through Redis pub/sub. For example, when a stake was created, the Staking service sent an event, and the Referral service picked it up and processed the referral reward *[adjust events]*. We chose Redis because it was already in our setup, it's very fast, and simple to use.
>
> If they ask about the weakness, I'm honest: Redis pub/sub **doesn't save messages**. If the Referral service is down at that moment, it misses the event. We handled that by also saving events in the DB and running a regular job to catch missed ones *[adjust]*. If I designed it today for reward events, I'd use Redis Streams or Kafka, which save messages and support acknowledgements."

See [09 → Redis pub/sub](09-kafka-redis-messaging.md#redis-pubsub).

**Bullet: "Designed Redis-based leader election so only one of 4–5 instances publishes real-time crypto price updates, eliminating duplicates; monitored with AWS CloudWatch."**

> 🎤 **Interview answer:** "The price service ran on 4–5 servers, and **each** one was sending price updates, so users got duplicates and we called the price provider too many times.
>
> I fixed it with leader election in Redis:
> - Each server tries `SET leader myId NX PX 10000`. `NX` means only one can succeed, and that one becomes the **leader**. The key expires in 10 seconds.
> - The leader renews the key every 3 seconds, but only if it still owns it.
> - The others keep trying. If the leader crashes, its key expires, and another server takes over within about 10 seconds.
> - If the leader fails to renew, it stops sending immediately.
>
> Prices come from a price provider *[adjust]*, and the leader sends updates to users through Socket.io *[adjust]*. In CloudWatch we logged leader changes, errors and how many updates were sent, with alerts if there was no leader for too long *[adjust]*."

Full answer + follow-ups in [09 → Leader election](09-kafka-redis-messaging.md#leader-election-with-redis).

---

## Appsinvo

### Led and mentored 6 junior developers

> 🎤 **Interview answer:** "At Appsinvo I led and mentored 6 junior developers. I kept it practical:
> - **Code reviews** where I explained **why**, not just what to change.
> - **Pair programming** on hard tasks.
> - Breaking big features into **small, clear tasks**.
> - **Shared standards** for folder structure, error handling and naming.
> - Helping with **debugging**, and short **knowledge-sharing sessions** *[adjust]*.
>
> If a junior kept making the same mistake, I explained the reason behind the fix, gave a small example or checklist, sat with them once, reviewed again, and appreciated the improvement.
>
> I was also in client calls: collecting requirements, clarifying edge cases, giving estimates, showing demos and handling change requests."

What I check in code reviews: see [11 → E18](11-scenario-questions.md#e18-code-review--what-do-you-check).

### Cut a user-listing API from 4.5 s to 550 ms

> 🎤 **Interview answer (practise this – it's your best number):**
> - "**Situation:** The admin user-listing API, with filters like status, date range, search and role *[adjust]*, took about **4.5 seconds**, and got slower as users grew.
> - **Finding the cause:** I logged the time of each step and ran the query with `explain('executionStats')`. I found three problems:
>   1. It was reading **every document**, because no index matched the filters.
>   2. It returned **full user documents** with lots of nested data.
>   3. Some **filtering was done in Node** after loading the data *[adjust]*.
> - **What I changed:**
>   1. Added **compound indexes** that match the filters and sort (ESR rule), so the query used the index.
>   2. Returned **only the fields the list needs** (name, email, status, date), with `.lean()` – much smaller response.
>   3. Moved **all filtering and pagination into the DB query**.
>   4. Ran the count query and data query **in parallel** *[adjust]*.
> - **Result:** about **550 ms** – around 88% faster, with lower DB CPU. I checked it by comparing response times before and after, on the same amount of data.
>
> If they ask 'why not just cache?': the filters change a lot and the data changes often, so fixing the query and indexes was the right first step. Caching could be added later for common filters."

### Introduced Apache Kafka with a dead-letter queue

> 🎤 **Interview answer:** "**Before:** notifications and order receipts were sent **inside** the order API request. So the API was slow, and if the email or SMS provider failed, either the order failed or the message was lost.
>
> **What I did:** I added Kafka. Now the order API just sends an event like `order.placed` and returns. Separate consumers send the notifications and receipts in the background *[adjust topic names]*. Events use `orderId` as the key, so all events of one order stay in order.
>
> **When something fails:** the consumer retries a few times through a retry topic, waiting longer each time. After the max tries, it sends the message to a **DLQ topic** with the error details, and the main flow continues. We watch the DLQ, fix the problem, and resend the messages.
>
> **No duplicate emails:** consumers check and save the event id before sending.
>
> **Why Kafka and not RabbitMQ or SQS?** Several consumers needed the same events, and being able to replay them was useful *[adjust honestly]*."

More: [09 → DLQ](09-kafka-redis-messaging.md#retries-and-dead-letter-queue-dlq), [comparison](09-kafka-redis-messaging.md#kafka-vs-rabbitmq-vs-sqs).

### Integrated 5 payment gateways with idempotency keys

> 🎤 **Interview answer:** "I integrated **Razorpay, Stripe, Cashfree, MyFatoorah and Tamara**. Different clients and regions needed different ones: Razorpay and Cashfree for India, Stripe for international cards, MyFatoorah for the Middle East, and Tamara for buy-now-pay-later in the Gulf *[adjust]*.
>
> **Keeping the code clean:** I made one common `PaymentGateway` interface with `createOrder`, `verifyPayment`, `handleWebhook`, `refund` and `getStatus`, and one class per gateway. A factory picks the right gateway by country or currency. The business code never knows which gateway is used.
>
> **Idempotency:** each payment attempt has a unique key, sent in a header and saved with a unique rule, together with the result. If the same key comes again, we return the saved result instead of charging again. Stripe supports this itself; for the others we did it on our side *[adjust]*.
>
> **Webhooks:** check the signature, unique index on the gateway's payment id (so duplicates are ignored), reply fast, process in the background, and a job checks payments stuck in 'pending'.
>
> **Amounts** are saved in the smallest unit (paise / cents). Some Gulf currencies like KWD have **3** decimal places, which is easy to get wrong."

Details in [07 → Idempotency](07-apis-backend.md#q-what-is-idempotency-and-how-do-you-implement-it).

### REST APIs and real-time chat (Socket.io, JWT)

> 🎤 **Interview answer:** "For chat, I check the JWT when the Socket.io connection opens (`io.use` middleware), so users without a valid token can't connect. Each user joins a room named after their user id. Messages are **saved in the DB first**, then sent to the receiver's room, so nothing is lost. There are 'delivered' and 'read' updates, and old messages load page by page. To run on many servers, we used the Socket.io Redis adapter *[adjust]*."

See [10 → Chat](10-system-design.md#design-a-chat-system).

### Jest unit tests

> 🎤 **Interview answer:** "I wrote Jest unit tests mainly for business logic, like price calculation, coupon rules and payment status changes, plus helpers and middleware. The database and payment gateways were replaced with fakes using `jest.mock`, so tests were fast and didn't depend on outside systems *[adjust]*. Unit tests check one piece alone. For full flows, we used integration tests with supertest, and Postman collections."

See [15 → Jest](15-testing-git-devops.md#jest-and-testing).

### Deployed on AWS (EC2, S3, Lambda) with Nginx and PM2

> 🎤 **Interview answer:**
> - "**EC2** ran the Node apps with **PM2** in cluster mode: auto-restart, logs, starts again after a server reboot, and `pm2 reload` for deploys without downtime.
> - **Nginx** in front: HTTPS *[adjust: Let's Encrypt / ACM]*, forwarding requests, gzip, and the special headers needed for Socket.io.
> - **S3** for images and invoices, with pre-signed URLs.
> - **Lambda** for *[adjust: e.g. resizing images on upload, scheduled jobs]*."

See [15 → PM2 & Nginx](15-testing-git-devops.md#pm2-and-nginx).

---

## Byte Digital Solutions

**Bullet: "Built a business website and internal dashboard end to end (Node.js, Express.js, React.js, MongoDB), with automated receipt emails, notifications and stock recalculation on each booking."**

> 🎤 **Interview answer:** "At Byte Digital, my first company, I built a business website and an internal dashboard end to end with the MERN stack, for *[adjust – describe the business]*.
> - **Stock on each booking:** reduced with one safe update – `$inc` with the condition 'stock ≥ quantity' – so two bookings at the same time couldn't oversell. A transaction when many items were involved, and stock put back on cancellation.
> - **Receipt emails:** sent with HTML templates *[adjust: Nodemailer / SES / SendGrid]* after the booking was saved. Ideally this should be in the background so the booking API isn't slowed down – a lesson I applied later with Kafka.
>
> The biggest learning from that job: owning features end to end, and talking directly to the business owner."

---

## Key projects

**Veer Maratha Matrimony** – 25K+ registered / 2K DAU.

> 🎤 **Interview answer:** "It has over 25,000 registered users and about 2,000 daily active users.
> - **'Near me' search** uses a `2dsphere` index with `$geoNear`, with location saved as `[longitude, latitude]`, plus preference filters.
> - **Fast match filters** – compound indexes on common filters like gender, date of birth (for age), religion and caste, plus pagination.
> - **Premium plans** paid through a payment gateway with webhook checks, and **SMS** for OTP and alerts *[adjust providers]*.
> - **Scaling issues at 2K daily users:** *[adjust: e.g. chat load, search speed]*."

See [04 → Geo query](04-mongodb-sql.md#extra-mongodb-questions).

**Rider Perfume** – full backend: orders, inventory, OTP auth, caching, pagination, rate limiting, invoices.

> 🎤 **Interview answer:** "I built the complete backend: orders, stock, OTP login, caching, pagination, rate limiting and automatic invoices. After a payment succeeds, a PDF invoice is created *[adjust: pdfkit / puppeteer]* with the order details and GST, saved in S3, and emailed to the customer."

Full answers in [12 Project Questions](12-project-questions.md).

---

## Skills section – be ready for each

Anything on your resume is fair game. Quick check:

| Skill | Be ready to answer | Where |
|---|---|---|
| TypeScript | Generics, utility types, TS with Express | [06](06-typescript.md) |
| NestJS | DI, guards, pipes, interceptors, lifecycle | [08](08-nestjs.md) |
| Microservices | How they talk, saga, failure handling | [10](10-system-design.md#monolith-vs-microservices) |
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

**RabbitMQ and SQS** are in your skills but not in any bullet → expect *"Where did you use RabbitMQ/SQS?"* Have a clear answer *[adjust: which project, what for]*. If it was small usage, say so honestly and explain the concepts well.

---

## Questions about your career timeline

> 🎤 **Interview answer:** "For timeline questions I keep it honest, positive and short. For short stays, I give the real reason, focused on growth or the project, without blaming anyone *[prepare your real reason]*. For doing my MCA while working, I explain how I managed my time *[adjust]*, and that it made my basics stronger in DBMS, operating systems, networking and DSA. And I'm always clear and truthful about my notice period."

- *"You've been at Notus since Sep 2026 and Revinfotech was May–Aug 2026. Why short stays?"* → Prepare an honest, positive answer – see [14 → Job changes](14-hr-behavioural.md#why-did-you-change-jobs--short-tenure).
- *"You did MCA (2024–2025) while working full-time. How?"* → Weekend/distance classes *[adjust]*, time management; it made your basics stronger (DBMS, OS, networking, DSA).
- *"Bachelor's 2021–2023 and job from May 2022?"* → Worked while finishing the degree *[adjust]*.
- *"Immediate joiner – are you currently serving notice?"* → Be clear and truthful about your status.

---

[Home](README.md) | [← 12 Project Questions](12-project-questions.md) | Next: [14 HR & Behavioural →](14-hr-behavioural.md)
