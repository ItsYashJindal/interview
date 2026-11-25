# Interview Prep – Yash Jindal (Backend / Node.js, 4.5 Years)

One place to prepare for every interview. Every file links back here and to the next/previous file, so you can move through them like tabs.

**How to open:** Open this folder in **Obsidian**, **VS Code** (Markdown preview: `Ctrl+Shift+V`) or push it to GitHub. All links are normal relative Markdown links, so they work everywhere.

---

## Files (in priority order)

| # | File | What is inside | Priority |
|---|------|----------------|----------|
| 1 | [JavaScript Concepts](01-javascript.md) | var/let/const, hoisting, closures, promises, event loop, `this`, prototypes + extra topics | Very high |
| 2 | [JavaScript Coding & Output Questions](02-javascript-coding.md) | Most asked practical/coding questions for 4 years experience, polyfills, output-guessing | Very high |
| 3 | [Node.js & Express](03-nodejs-express.md) | Architecture, event loop, middleware, errors, JWT, streams, worker threads, rate limiting, Redis | Very high |
| 4 | [MongoDB & SQL](04-mongodb-sql.md) | CRUD, indexing, aggregation, populate vs lookup, schema design, joins, transactions, optimisation | Very high |
| 5 | [React.js (+ Next.js basics)](05-react.md) | Hooks, state, Context, Redux Toolkit, lifecycle, performance, Virtual DOM | High |
| 6 | [TypeScript](06-typescript.md) | type vs interface, generics, enums, utility types, TS with Express | High |
| 7 | [APIs & Advanced Backend](07-apis-backend.md) | REST vs GraphQL, status codes, versioning, pagination, Lambda, Docker, security | High |
| 8 | [NestJS](08-nestjs.md) | Added – it is on your resume. Modules, DI, guards, pipes, interceptors, lifecycle | High (resume) |
| 9 | [Kafka, Redis & Messaging](09-kafka-redis-messaging.md) | Added – on your resume. Kafka, DLQ, Redis pub/sub, leader election, locks, RabbitMQ, SQS | High (resume) |
| 10 | [System Design Basics](10-system-design.md) | Added – always asked at 4+ years. Scaling, caching, microservices, small designs | High |
| 11 | [Real-World Scenario Questions](11-scenario-questions.md) | Your 12 scenario questions + extra common ones | Very high |
| 12 | [Project Questions](12-project-questions.md) | Your 9 project questions answered for your real projects | Very high |
| 13 | [Resume Deep-Dive](13-resume-questions.md) | Every line on your resume → questions interviewers will ask + answers | Very high |
| 14 | [HR & Behavioural](14-hr-behavioural.md) | Tell me about yourself, job changes, salary, mentoring, conflict | High |
| 15 | [Testing, Git, DevOps & Linux](15-testing-git-devops.md) | Jest, Git commands, PM2, Nginx, CI/CD, Linux commands | Medium |
| 16 | [Last-Day Revision Sheet](16-revision-cheatsheet.md) | One-line answers for everything – read before the interview | Read last |

---

## Suggested study plan (7 days)

| Day | Read | Practice |
|-----|------|----------|
| 1 | [01 JavaScript](01-javascript.md) | First 15 questions of [02 Coding](02-javascript-coding.md) |
| 2 | [03 Node & Express](03-nodejs-express.md) | Rest of [02 Coding](02-javascript-coding.md) (polyfills) |
| 3 | [04 MongoDB & SQL](04-mongodb-sql.md) | Write 10 SQL queries + 3 aggregation pipelines by hand |
| 4 | [07 APIs](07-apis-backend.md), [09 Kafka/Redis](09-kafka-redis-messaging.md), [08 NestJS](08-nestjs.md) | Explain leader election and DLQ out loud |
| 5 | [06 TypeScript](06-typescript.md), [05 React](05-react.md) | Write a generic function + a custom hook |
| 6 | [10 System Design](10-system-design.md), [11 Scenarios](11-scenario-questions.md) | Answer 5 scenarios out loud in 2 minutes each |
| 7 | [12 Projects](12-project-questions.md), [13 Resume](13-resume-questions.md), [14 HR](14-hr-behavioural.md) | Record yourself saying "Tell me about yourself" |
| Interview day | [16 Revision Sheet](16-revision-cheatsheet.md) | — |

---

## 🎤 Interview answers

Under almost every question you'll find a **🎤 Interview answer** block, placed above the notes. It's a ready-to-speak, first-person answer (about 30–90 seconds) that you can say to the interviewer. The notes and code below it are for deeper understanding. Wherever you see *[adjust]*, replace it with your real details before the interview.

---

## How to answer any question (simple method)

1. **What it is** – one line definition.
2. **Why / when we use it** – one line.
3. **Small example** – code or a real example from your work.
4. **One trade-off or gotcha** – this shows experience.

Example: *"What is a closure?"* → "A function that remembers variables from where it was created, even after the outer function has finished. We use it for private data and things like debounce. For example, my debounce function keeps the timer id in a closure. One gotcha: closures can keep large objects in memory if we hold references for too long."

Always try to connect the answer to **your own work** (Kafka, Redis leader election, payment idempotency, 4.5 s → 550 ms API). That is what separates 4 years experience from theory.

---

**Start here → [01 JavaScript Concepts](01-javascript.md)**
