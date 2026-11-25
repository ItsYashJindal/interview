[Home](README.md) | [← 02 JavaScript Coding](02-javascript-coding.md) | Next: [04 MongoDB & SQL →](04-mongodb-sql.md)

# 03 – Node.js & Express

## Contents
1. [Node.js architecture](#1-nodejs-architecture)
2. [Event-driven architecture](#2-event-driven-architecture)
3. [Event loop in Node.js](#3-event-loop-in-nodejs)
4. [Middleware](#4-middleware)
5. [Error handling](#5-error-handling)
6. [Authentication and authorization](#6-authentication-and-authorization)
7. [JWT and refresh tokens](#7-jwt-and-refresh-tokens)
8. [Streams and buffers](#8-streams-and-buffers)
9. [Worker threads vs child processes vs cluster](#9-worker-threads-vs-child-processes-vs-cluster)
10. [API rate limiting](#10-api-rate-limiting)
11. [Caching with Redis](#11-caching-with-redis)
- [Extra Node/Express questions (added)](#extra-nodeexpress-questions)

---

### 1. Node.js architecture

**Q. What is Node.js?**

> 🎤 **Interview answer:** "Node.js is a JavaScript runtime that lets us run JavaScript on the server, outside the browser. It's built on Google's V8 engine, which compiles and runs the JavaScript, and libuv, which provides the event loop and non-blocking I/O. Because of that, Node is very efficient for I/O-heavy applications like REST APIs, real-time chat and microservices. That's what I've been building with it for the last 4.5 years."

Node.js is a runtime that lets us run JavaScript outside the browser (on the server). It is built on Google's **V8 engine** (runs JS) and **libuv** (handles async I/O and the event loop).

**Q. Explain Node.js architecture.**

> 🎤 **Interview answer:** "Node runs our JavaScript on a single main thread using V8. When the code does I/O, like a DB query, an HTTP call or reading a file, Node doesn't wait. It hands the work off: network I/O goes to the operating system's async mechanisms like epoll, and things like file system, DNS, crypto and zlib go to libuv's thread pool, which has 4 threads by default. When the work is finished, its callback is queued, and the event loop picks it up and runs it on the main thread.
>
> So the correct way to describe it is a single-threaded event loop for our code, with multi-threaded work in the background. That's why one Node process can handle thousands of concurrent connections, as long as we don't block the main thread with CPU-heavy work."

1. Your JS code runs on **one main thread** (V8).
2. When you do I/O (DB query, file read, HTTP call), Node hands it off and does **not wait**.
   - Network I/O is handled by the operating system (epoll/kqueue).
   - File system, DNS lookup, crypto, zlib are handled by **libuv's thread pool** (default 4 threads, change with `UV_THREADPOOL_SIZE`).
3. When the work is done, its callback is put in a queue.
4. The **event loop** picks callbacks from the queue and runs them on the main thread.

So Node is "single-threaded" for your JavaScript, but uses other threads in the background for I/O.

**Q. Why is Node good for APIs?**

> 🎤 **Interview answer:** "In a typical API most of the time goes into waiting for the database, Redis or another service, not into CPU work. Node doesn't block a thread during that wait. It just registers a callback and serves other requests. So a single process can handle many concurrent connections with low memory, compared to a thread-per-request model. Using JavaScript or TypeScript on both frontend and backend, and the huge npm ecosystem, also make development faster."

Most API time is spent waiting for DB/network. Node does not block a thread while waiting, so one process can handle thousands of open connections with low memory.

**Q. When is Node NOT a good choice?**

> 🎤 **Interview answer:** "Node isn't a good fit for CPU-heavy work like video encoding, large image processing, heavy calculations or ML, if that work runs on the main thread, because while it runs the event loop is blocked and every other request waits. If a Node service has to do some of that, I move it to worker threads, a separate service, or a background queue. For example, I moved invoice and email work to Kafka consumers so the API stays responsive."

Heavy CPU work (video encoding, big image processing, complex calculations) on the main thread blocks all other requests. Fix: worker threads, a separate service, or a queue.

**Q. What is blocking vs non-blocking code?**

> 🎤 **Interview answer:** "Blocking code stops the thread until the operation finishes, like `fs.readFileSync` or a long synchronous loop. In Node that means the whole server stops serving other requests. Non-blocking code starts the operation and continues, and the result comes back later through a callback or promise. So in request handlers I never use Sync methods, I avoid heavy JSON parsing or CPU loops, and I keep everything async. Sync methods are fine only at startup, like reading a config file once."

```js
const data = fs.readFileSync("big.txt");   // blocking – whole server waits
fs.readFile("big.txt", (err, data) => {}); // non-blocking – server keeps serving others
```
Never use `Sync` methods or long loops inside request handlers.

---

### 2. Event-driven architecture

**Q. What does event-driven mean in Node?**

> 🎤 **Interview answer:** "Event-driven means the code reacts to events, like a request arriving, a file read finishing or a data chunk coming in, instead of running in a fixed sequence and waiting. Many Node core modules, like http, streams and net, are built on the EventEmitter class: you register listeners with `on` and something triggers them with `emit`. The big benefit is loose coupling. The code that places an order just emits 'orderPlaced', and email or stock listeners react without the order code knowing about them.
>
> I've used the same idea at system level with Kafka: the order service publishes an event, and separate consumers handle notifications and order receipts. One gotcha: EventEmitter listeners run synchronously, and an 'error' event with no listener crashes the process."

Code reacts to events (request arrived, file read finished, data chunk received) instead of running step by step and waiting. Many core modules (`http`, `fs streams`, `net`) are built on the **EventEmitter** class.

```js
const EventEmitter = require("events");
const orderEvents = new EventEmitter();

orderEvents.on("orderPlaced", (order) => sendEmail(order));
orderEvents.on("orderPlaced", (order) => updateStock(order));

orderEvents.emit("orderPlaced", { id: 1 });
```

**Benefit:** the code that places the order does not need to know about emails or stock. You can add new listeners without changing the main code (loose coupling).

**At system level** (between services) the same idea is done with Kafka / RabbitMQ / Redis pub/sub – I used Kafka for notification and order-receipt events. See [09 Kafka & Redis](09-kafka-redis-messaging.md).

**Gotcha:** EventEmitter listeners run **synchronously** in the order they were added. If a listener throws, `emit` throws. An `error` event with no listener crashes the process.

---

### 3. Event loop in Node.js

**Q. Explain the phases of the Node.js event loop.**

> 🎤 **Interview answer:** "Each iteration of the Node event loop goes through six phases. Timers runs expired setTimeout and setInterval callbacks. Pending callbacks runs some delayed system I/O callbacks. Idle/prepare is internal. Poll is the main one: it fetches new I/O events and runs their callbacks, and it waits here if nothing else is pending. Check runs setImmediate callbacks. And close callbacks handles things like socket close events.
>
> Between phases, and after each callback, Node drains the `process.nextTick` queue first and then the promise microtask queue. That's why nextTick and promises always run before the next timer or I/O callback."

Each round ("tick") goes through these phases:

| Phase | What runs |
|---|---|
| 1. **Timers** | `setTimeout`, `setInterval` callbacks whose time is over |
| 2. **Pending callbacks** | Some system I/O callbacks delayed from last round |
| 3. **Idle/prepare** | Internal use |
| 4. **Poll** | Gets new I/O events and runs their callbacks (most of your code runs here). Waits here if nothing else to do |
| 5. **Check** | `setImmediate` callbacks |
| 6. **Close callbacks** | `socket.on("close")` etc. |

**Between every phase (and after each callback)**, Node empties:
1. `process.nextTick` queue first
2. then the Promise microtask queue

**Q. process.nextTick vs setImmediate vs setTimeout(0)?**

> 🎤 **Interview answer:** "`process.nextTick` runs right after the current operation, before promise callbacks and before the event loop continues. It has the highest priority, and if used recursively it can starve I/O. Promise `.then` callbacks run right after the nextTick queue. `setTimeout(fn, 0)` runs in the timers phase of the next loop iteration, with a minimum of about 1 ms. `setImmediate` runs in the check phase, right after poll. In the main module the order of setTimeout 0 and setImmediate isn't guaranteed, but inside an I/O callback setImmediate always runs first. In practice I prefer setImmediate when I want to defer work without blocking I/O."

- `process.nextTick` – runs right after the current operation, before promises and before the loop moves on. Overusing it can starve I/O.
- `Promise.then` – microtask, right after nextTick queue.
- `setTimeout(fn, 0)` – timers phase of next loop (min ~1 ms).
- `setImmediate` – check phase, right after poll. Inside an I/O callback it always runs before `setTimeout(0)`.

**Q. How do you know the event loop is blocked?**

> 🎤 **Interview answer:** "The symptoms are that latency goes up for all endpoints at the same time, even simple ones like health checks, and one CPU core sits at 100%. To confirm it, I measure event loop lag with `perf_hooks.monitorEventLoopDelay`, or use tools like clinic.js, or APM tools like New Relic or Datadog. Then I profile with `--inspect` to find the hot function. Usually it's a synchronous operation, a big JSON.parse or stringify, a heavy loop or a bad regex, and I fix it by making it async, streaming or chunking it, or moving it to a worker thread."

API latency goes up for all endpoints, CPU at 100% on one core. Measure with `perf_hooks.monitorEventLoopDelay()`, clinic.js, or APM tools.

---

### 4. Middleware

**Q. What is middleware in Express?**

> 🎤 **Interview answer:** "Middleware is a function that runs between the request arriving and the response being sent. It gets `req`, `res` and `next`. It can read or modify the request and response, end the request by sending a response, call `next()` to pass control on, or call `next(err)` to jump straight to the error handler. Express is basically a pipeline of middleware.
>
> In my projects I use middleware for logging with request timing, JSON parsing, CORS and helmet, rate limiting, JWT authentication, role checks, request validation with Joi or zod, and finally a central error handler."

A function that runs **between the request coming in and the response going out**. It gets `(req, res, next)`. It can:
- read/change `req` and `res`
- end the request (send a response)
- call `next()` to pass control to the next middleware
- call `next(err)` to jump to the error handler

```js
const logger = (req, res, next) => {
  const start = Date.now();
  res.on("finish", () => console.log(req.method, req.url, res.statusCode, Date.now() - start + "ms"));
  next();
};
app.use(logger);
```

**Types of middleware:**
| Type | Example |
|---|---|
| Application-level | `app.use(logger)` |
| Router-level | `router.use(auth)` |
| Built-in | `express.json()`, `express.static()` |
| Third-party | `cors`, `helmet`, `morgan`, `express-rate-limit`, `multer` |
| Error-handling | 4 arguments: `(err, req, res, next)` |

**Q. Does the order of middleware matter?**

> 🎤 **Interview answer:** "Yes, very much. Middleware runs in the order it's registered. `express.json()` must come before any route that reads `req.body`. Security middleware like helmet, CORS and rate limiting should come early. Authentication must come before protected routes. The 404 handler and the error-handling middleware must come last, after all routes. A wrong order causes bugs like `req.body` being undefined or auth being skipped."

Yes. They run in the order they are added. `express.json()` must come before routes that read `req.body`. Auth must come before protected routes. The error handler must be **last**.

**Q. What if you forget to call next()?**

> 🎤 **Interview answer:** "If a middleware neither sends a response nor calls `next()`, the request just hangs. The client waits until it times out, and the connection stays open on the server. The opposite mistake is also common: calling `next()` after already sending a response, which causes the 'headers already sent' error. That's why I always write `return res.status(...).json(...)` when ending early."

The request hangs until the client times out (no response is sent).

**Middleware factory (closure) example – role check:**
```js
const allowRoles = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user.role)) return res.status(403).json({ message: "Forbidden" });
  next();
};
router.delete("/users/:id", auth, allowRoles("admin"), deleteUser);
```

---

### 5. Error handling

**Q. How do you handle errors in Express?**

> 🎤 **Interview answer:** "I follow a four-part pattern. First, a custom `AppError` class that carries a status code and an `isOperational` flag, so services can throw `new AppError('User not found', 404)`. Second, an `asyncHandler` wrapper, or Express 5, so rejected promises from async routes are passed to `next` automatically and I don't write try/catch in every controller. Third, one central error middleware with four arguments, registered last: it logs the full error with the request id and returns a consistent JSON response, and for unexpected errors it hides the internal details and returns a generic 500. And fourth, process-level handlers for `unhandledRejection` and `uncaughtException` that log the error and, for uncaught exceptions, exit so PM2 restarts the process in a clean state."

1. **Custom error class** so each error carries a status code.
2. **Async wrapper** (or Express 5, which catches rejected promises automatically) so we don't write try/catch in every route.
3. **One central error-handling middleware** at the end.
4. **Process-level handlers** for anything missed.

```js
// 1. Custom error
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // expected error, not a bug
  }
}

// 2. Async wrapper (Express 4)
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

router.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError("User not found", 404);
  res.json(user);
}));

// 404 for unknown routes
app.use((req, res, next) => next(new AppError("Route not found", 404)));

// 3. Central error handler – must have 4 args and be last
app.use((err, req, res, next) => {
  const status = err.statusCode || 500;
  logger.error({ err, path: req.path });                  // log full error
  res.status(status).json({
    success: false,
    message: err.isOperational ? err.message : "Something went wrong", // hide internals
  });
});

// 4. Process level
process.on("unhandledRejection", (reason) => { logger.error(reason); });
process.on("uncaughtException", (err) => { logger.error(err); process.exit(1); }); // PM2 restarts it
```

**Q. Operational errors vs programmer errors?**

> 🎤 **Interview answer:** "Operational errors are expected runtime problems in a correct program: invalid input, resource not found, DB timeout, a third-party API being down. We handle those gracefully, with a proper status code, retry or fallback. Programmer errors are bugs, like reading a property of undefined or wrong logic. We can't really handle those. We log them and fix the code, and if one reaches `uncaughtException`, the safest choice is to restart the process, because its state may be corrupted."

- Operational: expected problems – invalid input, not found, DB timeout. Handle them and send proper response.
- Programmer errors: bugs – undefined variable, wrong logic. Log it, and for `uncaughtException` restart the process because the app state may be broken.

**Q. Errors in Express 4 async routes?**

> 🎤 **Interview answer:** "In Express 4, if an async route handler throws or a promise rejects, Express doesn't catch it, because it only catches synchronous errors. The request hangs and we get an unhandledRejection. So we wrap handlers with an asyncHandler that does `.catch(next)`, or use the `express-async-errors` package. Express 5 fixed this: it automatically forwards rejected promises to the error handler."

In Express 4, an error thrown inside an `async` route is **not** caught automatically – the promise rejects and the request hangs (and you get an unhandledRejection). That is why we use `asyncHandler` or `express-async-errors`. Express 5 handles this by itself.

More in [11 Scenarios → Q5](11-scenario-questions.md#5-how-do-you-handle-errors-in-expressjs).

---

### 6. Authentication and authorization

**Q. Difference?**

> 🎤 **Interview answer:** "Authentication is verifying who the user is, for example login with password, OTP or Google. If it fails, we return 401 Unauthorized. Authorization is checking what an authenticated user is allowed to do, for example only an admin can delete users, or a user can only see their own orders. If that fails, we return 403 Forbidden. Authentication always comes first. In my projects, JWT middleware handles authentication, and role guards plus ownership checks handle authorization."

- **Authentication** = *Who are you?* (login with email/password, OTP, Google). Fails with **401 Unauthorized**.
- **Authorization** = *What are you allowed to do?* (admin can delete, user cannot). Fails with **403 Forbidden**.

**Q. Session-based vs token-based auth?**

> 🎤 **Interview answer:** "With session-based auth, the server stores the session, usually in Redis, and the client only holds a session id cookie. Logging out is easy, you just delete the session, but every server needs access to the shared session store. With token-based auth like JWT, the server stores nothing. The client sends a signed token, and any server or microservice can verify it with the key, so it scales well and fits mobile apps and SPAs. The downside is that a JWT can't easily be revoked before it expires.
>
> So I use short-lived access tokens plus refresh tokens stored in the DB or Redis, which gives a mix of both: stateless verification on every request, and revocation through the refresh token."

| Session | Token (JWT) |
|---|---|
| Server stores session (memory/Redis), client gets a session id cookie | Server stores nothing; client sends a signed token |
| Easy to log out (delete session) | Hard to cancel a token before it expires |
| Needs shared session store for multiple servers | Works across servers and services easily |
| Good for classic web apps | Good for mobile apps, SPAs, microservices |

**Q. How do you store passwords?**

> 🎤 **Interview answer:** "Never in plain text, and never with plain fast hashes like MD5 or SHA-256. I use bcrypt, or argon2. It adds a unique salt to every password, so the same passwords get different hashes and rainbow tables don't work. It's also deliberately slow, with a cost factor of about 10 to 12, which makes brute force expensive. At login I use `bcrypt.compare`. I also never return the hash in API responses: the field is excluded with `select: false` in Mongoose."

Never plain text. Hash with **bcrypt** (or argon2) which adds a salt and is slow on purpose.
```js
const hash = await bcrypt.hash(password, 10);        // 10 = salt rounds
const ok = await bcrypt.compare(inputPassword, hash);
```

**Q. Types of authorization**

> 🎤 **Interview answer:** "The common types are RBAC, ownership checks and ABAC. RBAC is role-based: permissions are attached to roles like admin, seller and user. I used Auth0 RBAC in the KindSwap NestJS microservices, where permissions come inside the access token and a guard checks them per route. Ownership checks make sure users can only access their own resources, like querying an order by both id and userId. ABAC is attribute-based: rules use attributes like department, region or time. In practice most apps combine RBAC with ownership checks."

- **RBAC (Role-Based)** – permissions by role: admin, seller, user. I used Auth0 RBAC in KindSwap microservices.
- **Ownership check** – user can edit only their own order: `if (order.userId !== req.user.id) 403`.
- **ABAC (Attribute-Based)** – rules based on attributes (department, time, region).

**Q. OTP login flow (used in Rider Perfume)**

> 🎤 **Interview answer:** "In Rider Perfume, the user enters their phone number. I validate it and rate-limit OTP requests per phone and IP, around 3 per 10 minutes, to stop SMS abuse. Then I generate a 6-digit OTP using crypto, store a hash of it in Redis with a 5-minute TTL, and send it via the SMS provider. On verify, I compare the hash, limit wrong attempts to about 5, and delete the OTP after a successful use so it can't be reused. Then I create the user if they're new and issue a short-lived access token and a refresh token in an httpOnly cookie. Redis is perfect here because of the built-in TTL."

1. User enters phone → generate 6-digit OTP → store **hashed** OTP in Redis with 5-minute TTL → send SMS.
2. Rate-limit OTP requests per phone/IP (e.g. 3 per 10 minutes).
3. User submits OTP → compare → delete from Redis (one-time use) → limit wrong attempts (e.g. 5).
4. Issue access + refresh tokens.

---

### 7. JWT and refresh tokens

**Q. What is a JWT?**

> 🎤 **Interview answer:** "A JWT, or JSON Web Token, is a compact token with three Base64URL-encoded parts separated by dots: header, payload and signature. The header says the algorithm, like HS256 or RS256. The payload holds claims like the user id, role and expiry. The signature is created with a secret or private key, so if anyone changes the payload, verification fails. The important point is that the payload is only encoded, not encrypted, so anyone can read it. I never put passwords or sensitive data in it, just ids and roles."

JSON Web Token – a string with 3 parts separated by dots: `header.payload.signature`.
- **Header** – algorithm (HS256 / RS256).
- **Payload** – data (userId, role, expiry `exp`). Only **Base64 encoded, not encrypted** – anyone can read it. Never put passwords or secrets in it.
- **Signature** – made with a secret key. If someone changes the payload, the signature does not match, so the server rejects it.

**Q. How does JWT auth work?**

> 🎤 **Interview answer:** "On login, after checking credentials, the server signs a token with the user id and role and a short expiry, like 15 minutes, and returns it. The client sends it on every request in the `Authorization: Bearer` header. An auth middleware extracts the token and verifies the signature and expiry with `jwt.verify`. If it's valid, it attaches the decoded user to `req.user` and calls next. If it's missing, invalid or expired, it returns 401. Since verification only needs the key, there's no DB lookup on every request, which keeps it fast and stateless."

```js
// Login
const accessToken = jwt.sign({ sub: user.id, role: user.role }, process.env.ACCESS_SECRET, { expiresIn: "15m" });

// Auth middleware
const auth = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1]; // "Bearer <token>"
  if (!token) return res.status(401).json({ message: "No token" });
  try {
    req.user = jwt.verify(token, process.env.ACCESS_SECRET);
    next();
  } catch {
    return res.status(401).json({ message: "Invalid or expired token" });
  }
};
```

**Q. Why refresh tokens?**

> 🎤 **Interview answer:** "It's a balance between security and user experience. Access tokens should be short-lived, around 10 to 15 minutes, so a stolen token is only useful for a short time. But we don't want users logging in every 15 minutes. So at login we also issue a long-lived refresh token, valid for 7 to 30 days, which is used only on the `/auth/refresh` endpoint to get a new access token. The refresh token is stored on the server side in the DB or Redis, so we can revoke it on logout. I also rotate it: each refresh issues a new refresh token and invalidates the old one."

Access tokens should be **short-lived** (10–15 min) so a stolen token is useful only for a short time. But we don't want users to log in every 15 minutes. So we also give a **long-lived refresh token** (7–30 days) that is used only to get a new access token.

**Flow:**
1. Login → server returns access token (15 min) + refresh token (7 days).
2. Client calls APIs with access token.
3. Access token expires → API returns 401 → client calls `/auth/refresh` with refresh token.
4. Server checks refresh token (signature + exists in DB/Redis + not revoked) → returns new access token (and a new refresh token – **rotation**).
5. Logout → delete refresh token from DB/Redis.

**Q. Where to store tokens on the client?**

> 🎤 **Interview answer:** "The refresh token goes in an httpOnly, Secure, SameSite cookie, so JavaScript can't read it, which protects it from XSS. The access token is best kept in memory, in a JS variable, or also in an httpOnly cookie. I avoid localStorage for sensitive tokens, because any XSS script can read it. If we use cookies, we also need CSRF protection, which SameSite mostly handles. For mobile apps, tokens go in secure storage like Keychain or Keystore."

- Refresh token: **httpOnly, Secure, SameSite cookie** (JavaScript cannot read it → safe from XSS).
- Access token: in memory (JS variable) or also an httpOnly cookie. Avoid localStorage for sensitive tokens (XSS can steal it).

**Q. How do you log out / revoke a JWT?**

> 🎤 **Interview answer:** "A JWT can't really be deleted, because it stays valid until it expires. So I combine a few approaches. Access tokens are short-lived, and logout deletes the refresh token from the DB, so the session ends within minutes. For immediate revocation, like a password change or a compromised account, I blacklist the token's `jti` in Redis with a TTL equal to its remaining lifetime, and the auth middleware checks that. For 'logout from all devices', I keep a `tokenVersion` on the user, include it in the token, and increment it, so all old tokens fail the check."

JWT cannot be "deleted". Options:
- Keep access tokens short-lived and delete the refresh token from DB → user is logged out within 15 min.
- Blacklist the token id (`jti`) in Redis until its expiry time.
- Store a `tokenVersion` on the user; increase it on "logout from all devices" and check it in middleware.

**Q. Refresh token rotation and reuse detection?**

> 🎤 **Interview answer:** "With rotation, every time a refresh token is used, the server issues a new refresh token and marks the old one as used. Reuse detection means that if an already-used refresh token is presented again, it means someone has a stolen copy, because a legitimate client would only have the latest one. In that case we revoke the whole token family for that user and force a fresh login. It limits the damage of a stolen refresh token."

Every time a refresh token is used, issue a new one and mark the old one as used. If an old (used) refresh token comes again, someone stole it → revoke all tokens of that user.

**Q. HS256 vs RS256?**

> 🎤 **Interview answer:** "HS256 is symmetric: the same secret signs and verifies the token. It's simple and fine when one service both issues and verifies tokens. RS256 is asymmetric: a private key signs and a public key verifies. That's better for microservices, because every service can verify tokens with the public key but none of them can create tokens. Auth0 uses RS256 and publishes its public keys at a JWKS endpoint, which is how our NestJS services in KindSwap verified tokens."

- HS256: one shared secret signs and verifies. Simple, good for one service.
- RS256: private key signs, public key verifies. Good for microservices – other services can verify without being able to create tokens (Auth0 uses RS256).

Full implementation answer: [11 Scenarios → Q4](11-scenario-questions.md#4-how-would-you-implement-jwt-authentication).

---

### 8. Streams and buffers

**Q. What is a Buffer?**

> 🎤 **Interview answer:** "A Buffer is a fixed-size chunk of raw binary data, allocated outside the V8 heap. Node uses it whenever it deals with bytes: reading files, network packets, images and crypto. We can convert between Buffers and strings with encodings like utf8, base64 or hex. For example, I've used it for base64 encoding and for verifying webhook signatures, where the raw request body as a Buffer must be used for the HMAC check."

A fixed-size chunk of raw binary data (bytes) in memory, outside the V8 heap. Used for files, images, network packets.
```js
const buf = Buffer.from("hello");
buf.toString("base64"); // "aGVsbG8="
```

**Q. What is a Stream?**

> 🎤 **Interview answer:** "A stream is a way to process data piece by piece, in chunks, instead of loading everything into memory at once. There are four types. Readable, like a file read stream or an incoming request. Writable, like a file write stream or the response. Duplex, which is both, like a TCP socket. And Transform, which changes data as it passes through, like gzip or a CSV parser. Streams are also EventEmitters, with events like data, end and error."

A way to process data **piece by piece (chunks)** instead of loading everything into memory at once.

**Types:**
| Type | Example |
|---|---|
| Readable | `fs.createReadStream`, `req` (incoming request) |
| Writable | `fs.createWriteStream`, `res` (response) |
| Duplex (both) | TCP socket |
| Transform (change data while passing) | `zlib.createGzip()`, CSV parser |

**Q. Why use streams?** Example: send a 2 GB file.

> 🎤 **Interview answer:** "With `readFile`, a 2 GB file is fully loaded into memory before sending. With a few such requests the server runs out of memory, and the user also waits for the full read before the first byte. With `fs.createReadStream` piped to the response using `pipeline`, data is sent in small chunks, around 64 KB, so memory stays low and the download starts immediately. I use the same approach for CSV exports of lakhs of rows: stream a DB cursor through a transform into the response, and for uploading files to S3."

```js
// Bad – loads 2 GB into memory
app.get("/file", async (req, res) => res.send(await fs.promises.readFile("big.zip")));

// Good – sends in chunks, low memory
const { pipeline } = require("stream/promises");
app.get("/file", async (req, res, next) => {
  try {
    await pipeline(fs.createReadStream("big.zip"), res);
  } catch (err) { next(err); }
});
```

**Q. What is backpressure?**

> 🎤 **Interview answer:** "Backpressure happens when the readable side produces data faster than the writable side can consume it, for example reading from a fast disk and writing to a slow client. If we ignore it, data piles up in memory. The writable signals it by returning false from `write()`, and we should pause until the 'drain' event. `pipe` and `pipeline` handle this automatically, which is a big reason to use them instead of manual 'data' handlers."

When the writable side is slower than the readable side, data piles up in memory. `pipe()` / `pipeline()` handle this automatically: they pause reading when the writer's buffer is full (`write()` returns `false`) and resume on the `drain` event.

**Q. pipe vs pipeline?**

> 🎤 **Interview answer:** "`pipe` connects streams and handles backpressure, but it doesn't forward errors. If one stream fails, the others aren't closed, so you can leak file handles or leave the response hanging. `pipeline` from `stream/promises` connects multiple streams, forwards errors, destroys all streams on failure, and gives a promise you can await in try/catch. So I always prefer `pipeline` in production code."

`pipeline` passes errors properly and closes all streams on error. `pipe` does not forward errors – you can leak file handles. Prefer `pipeline`.

**Real uses:** CSV export of lakhs of rows (stream DB cursor → CSV transform → response), file upload to S3, generating invoices/PDFs, reading large log files.

---

### 9. Worker threads vs child processes vs cluster

> 🎤 **Interview answer:** "They solve different problems. Worker threads run JavaScript in parallel threads inside the same process. They can share memory through SharedArrayBuffer and talk via postMessage, so they're for CPU-heavy JS work like image processing, PDF generation, hashing or parsing huge JSON, without blocking the event loop. Child processes start a completely separate process, which is right for running other programs like ffmpeg, Python scripts or shell commands. Spawn streams output, exec buffers it, and fork is for Node scripts with an IPC channel. Cluster creates multiple copies of the whole app, one per CPU core, sharing the same port, to handle more HTTP requests.
>
> In production I used PM2 cluster mode on EC2 to use all cores, and for CPU-heavy tasks I'd use worker threads or move them to a background job."

| | Worker threads | Child process | Cluster |
|---|---|---|---|
| What | New thread inside the same process | A completely new process | Many copies of your app (processes) |
| Memory | Can share memory (`SharedArrayBuffer`) | Separate memory | Separate memory |
| Communication | `postMessage` (fast) | IPC / stdin/stdout (slower) | IPC; OS shares the port |
| Use for | CPU-heavy JS work (image resize, PDF, hashing, big JSON parse) | Running other programs/scripts (ffmpeg, Python, shell commands) | Using all CPU cores to handle more HTTP requests |
| Crash effect | Can affect the process | Isolated | One worker dies, others continue |

```js
// Worker thread
const { Worker } = require("worker_threads");
const worker = new Worker("./heavy-task.js", { workerData: { n: 40 } });
worker.on("message", (result) => console.log(result));

// Child process
const { spawn } = require("child_process");
const ff = spawn("ffmpeg", ["-i", "in.mp4", "out.mp3"]);
```

**Child process methods:**
- `spawn` – streams output, good for long or large output.
- `exec` – runs in a shell, buffers output (small output only).
- `execFile` – like exec without shell (safer).
- `fork` – special spawn for Node scripts with an IPC channel.

**Q. How do you use all CPU cores?**

> 🎤 **Interview answer:** "Node runs our code on one core, so to use all of them we run multiple processes. On EC2 I used PM2 cluster mode, `pm2 start app.js -i max`, which forks one worker per core sharing the same port, restarts crashed workers automatically, and supports zero-downtime reload. In containerised setups we run multiple containers or pods behind a load balancer instead. Since the processes don't share memory, any shared state, like sessions, rate-limit counters, cache or Socket.io rooms, has to live in Redis."

- `cluster` module or **PM2 cluster mode** (`pm2 start app.js -i max`) – I used PM2 on EC2.
- Or run multiple containers behind a load balancer.

---

### 10. API rate limiting

**Q. What is rate limiting and why?**

> 🎤 **Interview answer:** "Rate limiting caps how many requests a client, identified by IP, user id or API key, can make in a time window. It protects against brute-force login attempts, OTP or SMS abuse, scraping and DDoS, and it protects expensive endpoints and third-party costs. When the limit is exceeded, we return 429 Too Many Requests with a Retry-After header. In Rider Perfume I added it on login and OTP routes, with stricter limits there and looser global limits elsewhere."

Limit how many requests a client (IP / user / API key) can make in a time window. Protects from brute force login, DDoS, scraping, and expensive endpoints (OTP, payments). Exceeded → **429 Too Many Requests** with `Retry-After` header.

**Algorithms (simple explanation):**
| Algorithm | How | Note |
|---|---|---|
| Fixed window | Count requests per minute block (12:00–12:01) | Simple; burst possible at window edges |
| Sliding window log | Store timestamp of each request, count last 60 s | Accurate, more memory |
| Sliding window counter | Mix of current + previous window counts | Good balance |
| Token bucket | Bucket refills N tokens/sec; each request takes one | Allows short bursts – most common |
| Leaky bucket | Requests leave at a fixed rate | Smooth output |

**Q. Why Redis for rate limiting?**

> 🎤 **Interview answer:** "Because in production we run multiple instances. If each server keeps its own in-memory counter, a user effectively gets the limit multiplied by the number of servers, and counters reset on every restart. Redis gives one shared, fast counter for all instances. `INCR` is atomic, so there are no race conditions, and `EXPIRE` cleans up keys automatically. For a fixed window I use INCR plus EXPIRE on a key like `rl:ip:minute`. For more accuracy, a sliding window with sorted sets or a token bucket in a Lua script. I'd also add broad limits at Nginx or the API gateway level."

With multiple servers, in-memory counters are separate per server, so a user gets limit × servers. Redis gives one shared counter. `INCR` is atomic.

```js
// Fixed window with Redis
async function rateLimit(req, res, next) {
  const key = `rl:${req.ip}:${Math.floor(Date.now() / 60000)}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 60);
  if (count > 100) return res.status(429).json({ message: "Too many requests" });
  next();
}
```
Or use `express-rate-limit` with `rate-limit-redis` store. I added rate limiting on login/OTP routes in Rider Perfume.

**Where to rate limit:** API gateway / Nginx (`limit_req`) for broad protection + app level for per-user or per-route rules.

---

### 11. Caching with Redis

**Q. What is Redis?**

> 🎤 **Interview answer:** "Redis is an in-memory key-value data store. Since data lives in RAM, reads and writes are sub-millisecond. It supports rich data types: strings, hashes, lists, sets, sorted sets and streams, plus TTL on keys, atomic operations, pub/sub and Lua scripting. I've used it for caching, OTP storage with TTL, rate limiting, pub/sub between microservices, distributed locks, and leader election across 4 to 5 instances in KindSwap. It can persist with RDB snapshots or AOF, but I mainly treat it as a fast layer, not the source of truth."

An in-memory key-value store. Very fast (sub-millisecond) because data is in RAM. Supports strings, hashes, lists, sets, sorted sets, TTL (expiry), pub/sub.

**Q. Cache-aside pattern (most common)**

> 🎤 **Interview answer:** "With cache-aside, the application manages the cache itself. On a read, I check Redis first. If it's a hit, I return it. If it's a miss, I read from the DB, store the result in Redis with a TTL, and return it. On a write, I update the DB and then delete the cache key, so the next read gets fresh data. I prefer deleting over updating the cache to avoid race conditions. The TTL is a safety net in case an invalidation is missed. I used this for product lists and categories in Rider Perfume."

1. Check cache.
2. Hit → return.
3. Miss → read DB → save in cache with TTL → return.
```js
async function getProduct(id) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const product = await Product.findById(id).lean();
  if (product) await redis.set(key, JSON.stringify(product), "EX", 300); // 5 min
  return product;
}

// On update – invalidate
async function updateProduct(id, data) {
  await Product.updateOne({ _id: id }, data);
  await redis.del(`product:${id}`);
}
```

**Q. Other caching strategies**

> 🎤 **Interview answer:** "Besides cache-aside, there's write-through, where every write goes to both the cache and the DB together, so the cache is always fresh but writes are slower. Write-back, or write-behind, writes to the cache first and saves to the DB later in batches. It's very fast, but you can lose data if the cache fails. And read-through, where the cache layer itself loads from the DB on a miss, so the app only talks to the cache. For most APIs cache-aside with TTL is the practical choice."

- **Write-through** – write to cache and DB together. Cache always fresh, writes slower.
- **Write-back (write-behind)** – write to cache, save to DB later. Fast, but risk of data loss.
- **Read-through** – cache library loads from DB itself on a miss.

**Q. What do you cache?** Data that is read often and changes rarely: product lists, categories, config, user profile, results of heavy queries, sessions, OTPs.

> 🎤 **Interview answer:** "I cache data that's read often, changes rarely and is expensive to compute: product lists, categories, configuration, user profiles, results of heavy aggregation queries, plus sessions and OTPs. I don't cache data that must be exactly correct right now, like wallet balance, payment status or stock during checkout. If I do cache something like that, it's with a very short TTL and strict invalidation."

**Don't cache:** wallet balance, stock count during checkout, anything that must be exactly correct right now (or use very short TTL + invalidation).

**Q. Cache invalidation problems**

> 🎤 **Interview answer:** "The main problems are stale data, stampede, penetration and memory. Stale data, where the cache has old values: I fix it with a TTL plus deleting the key on update. Cache stampede, where a hot key expires and hundreds of requests hit the DB together: the fixes are a lock so only one request rebuilds it, random jitter on TTLs, or refreshing popular keys before they expire. Cache penetration, where requests for ids that don't exist always miss and hit the DB: I cache a 'null' result for a short time and validate input. And memory: I set `maxmemory` with an eviction policy like `allkeys-lru`."

- **Stale data** – fix with TTL + delete key on update.
- **Cache stampede** – popular key expires, 1000 requests hit DB together. Fix: lock (only one request rebuilds), random TTL jitter, or refresh before expiry.
- **Cache penetration** – requests for IDs that don't exist always miss. Fix: cache "null" for a short time, validate input.
- **Memory full** – set `maxmemory` and eviction policy like `allkeys-lru`.

More Redis (pub/sub, leader election, locks) in [09 Kafka & Redis](09-kafka-redis-messaging.md).

---

## Extra Node/Express questions

### Q. What is libuv?

> 🎤 **Interview answer:** "libuv is a C library that Node uses for its asynchronous, cross-platform I/O. It provides the event loop itself, a thread pool of 4 threads by default for file system, DNS, crypto and zlib work, and abstractions over OS mechanisms like epoll, kqueue and IOCP for network I/O. It's what makes non-blocking I/O possible in Node."

A C library used by Node that provides the event loop, thread pool and async I/O across operating systems.

### Q. Is Node.js single-threaded?

> 🎤 **Interview answer:** "Partly. Our JavaScript runs on a single main thread with one call stack and one event loop. But Node itself isn't fully single-threaded: libuv uses a thread pool for file, crypto and DNS work, V8 uses background threads for garbage collection, and we can create worker threads ourselves. So the best answer is 'a single-threaded event loop with multi-threaded I/O in the background'."

Your JavaScript runs on one main thread. But libuv uses a thread pool for file/crypto/DNS work, and you can create worker threads. So: "single-threaded event loop, multi-threaded in the background".

### Q. What is package.json vs package-lock.json?

> 🎤 **Interview answer:** "package.json describes the project: name, scripts, and dependency version ranges, like `^4.18.0`. package-lock.json records the exact version of every installed package, including nested dependencies, so every developer and the CI server install exactly the same tree. We always commit the lock file and use `npm ci` in CI and production, because it installs strictly from the lock file. Caret allows minor and patch updates, and tilde allows only patch updates."

- `package.json` – project info, scripts, dependency **ranges** (`^4.18.0`).
- `package-lock.json` – **exact** installed versions of every dependency (and their dependencies), so every machine installs the same thing. Commit it. Use `npm ci` in CI/production.
- `^1.2.3` allows minor+patch updates; `~1.2.3` allows only patch updates.

### Q. dependencies vs devDependencies?

> 🎤 **Interview answer:** "Dependencies are needed at runtime in production, like express, mongoose or ioredis. devDependencies are only for development and build, like jest, eslint, typescript, nodemon and type definitions. In production builds or Docker images, I install with `npm ci --omit=dev` to keep the image smaller and reduce the attack surface."

`dependencies` are needed at runtime (express, mongoose). `devDependencies` only for development (jest, eslint, typescript, nodemon). `npm ci --omit=dev` for production.

### Q. How do you manage environment config?

> 🎤 **Interview answer:** "Locally I use a `.env` file with dotenv, and it's always in .gitignore, with a `.env.example` committed for reference. In production, values come from real environment variables or AWS Secrets Manager or SSM Parameter Store, so secrets are never in code. I also validate the config at startup with zod or Joi. If a required variable like the DB URL or JWT secret is missing, the app fails immediately with a clear error instead of breaking later at runtime."

`.env` file with `dotenv` locally (never commit it), real env variables / AWS Secrets Manager / SSM in production. Validate env on startup (e.g. with `joi` or `zod`) so the app fails fast if something is missing.

### Q. What is CORS?

> 🎤 **Interview answer:** "CORS, Cross-Origin Resource Sharing, is a browser security rule. By default a page from one origin, meaning protocol plus domain plus port, can't call an API on a different origin unless the server explicitly allows it with headers like `Access-Control-Allow-Origin`. For non-simple requests, like PUT or DELETE, JSON bodies or custom headers, the browser first sends an OPTIONS preflight request. In Express I use the `cors` middleware with a whitelist of our frontend domains, and `credentials: true` if cookies are involved. It's only a browser rule, so Postman and server-to-server calls aren't affected."

Browsers block a page from one origin (domain+port+protocol) calling an API on another origin unless the server allows it with headers like `Access-Control-Allow-Origin`. For non-simple requests (PUT, custom headers, JSON) the browser first sends an **OPTIONS preflight** request.
```js
app.use(cors({ origin: ["https://riderbeauty.com"], credentials: true }));
```
CORS is a browser rule – Postman or server-to-server calls are not affected.

### Q. req.params vs req.query vs req.body

> 🎤 **Interview answer:** "`req.params` holds route path variables, like `id` in `/users/:id`, used to identify a resource. `req.query` holds the query string after the question mark, used for filters, sorting and pagination, like `?page=2&limit=20`. `req.body` holds the payload sent in POST, PUT or PATCH, like JSON or form data, and it needs a body parser like `express.json()`. All three come from the client, so I validate all of them."

- `req.params` – path values: `/users/:id` → `req.params.id`
- `req.query` – after `?`: `/users?page=2` → `req.query.page`
- `req.body` – JSON/form data sent in POST/PUT (needs `express.json()`)

### Q. app.use vs app.get?

> 🎤 **Interview answer:** "`app.use` mounts middleware or routers. It matches every HTTP method and any path that starts with the given prefix, so `app.use('/api', router)` handles everything under /api. `app.get` registers a route handler only for GET requests matching that exact path pattern. So `use` is for cross-cutting concerns and mounting, and `get`, `post` and so on are for actual endpoints."

`app.use` matches any HTTP method and any path that **starts with** the given path (used for middleware and routers). `app.get` matches only GET on the exact path pattern.

### Q. Express Router

> 🎤 **Interview answer:** "Express Router is like a mini-app for grouping related routes. I create a router per module, like orders, users or products, define its routes and route-specific middleware there, and mount it with `app.use('/api/v1/orders', orderRouter)`. That keeps the app modular, makes versioning easy, and lets me apply middleware like auth to a whole group of routes."

Split routes into modules:
```js
const router = express.Router();
router.get("/", listOrders);
router.post("/", auth, createOrder);
app.use("/api/v1/orders", router);
```

### Q. How do you handle file uploads?

> 🎤 **Interview answer:** "For normal uploads I use multer for multipart form-data, with file type and size validation, and then upload to S3 and save only the S3 key in the DB. For large files, I generate a pre-signed S3 URL on the backend, and the client uploads directly to S3, so the file never passes through our Node server. That saves bandwidth and memory. I also validate on the backend after upload, and I never trust the file extension alone."

`multer` middleware for multipart/form-data. Validate type and size. For large files, upload directly from client to S3 using a **pre-signed URL** so the file does not pass through your server.

### Q. Graceful shutdown

> 🎤 **Interview answer:** "When PM2, Docker or Kubernetes deploys a new version, it sends SIGTERM. In the handler I first stop accepting new connections with `server.close`, let the in-flight requests finish, then close the DB, Redis and Kafka connections, and exit with code 0. I also set a force-exit timeout of around 10 seconds in case something hangs. For Kafka consumers I commit offsets before disconnecting. Without this, deploys can cut off requests midway or lose messages."

On `SIGTERM` (sent by PM2/Docker/Kubernetes during deploy): stop accepting new requests, finish running ones, close DB/Redis/Kafka connections, then exit.
```js
process.on("SIGTERM", () => {
  server.close(async () => {
    await mongoose.connection.close();
    await redis.quit();
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10000); // force after 10s
});
```

### Q. How to find a memory leak in Node?

> 🎤 **Interview answer:** "First I confirm it: memory grows steadily over time and never drops after GC, visible in `process.memoryUsage`, PM2 monit or CloudWatch, often ending in out-of-memory restarts. Then I run the app with `--inspect`, take heap snapshots in Chrome DevTools at two different times under load, and compare which objects keep growing and what is retaining them. The usual causes are global maps or arrays used as unbounded caches, event listeners that are never removed, timers that are never cleared, and closures holding large objects. The fix is usually adding a limit or TTL, cleaning up listeners, or moving the cache to Redis."

Watch memory growing over time (`process.memoryUsage()`, PM2 monit, CloudWatch). Take heap snapshots with `--inspect` + Chrome DevTools at two times and compare which objects grow. Common causes: global arrays/maps that only grow, listeners not removed, unclosed timers, caching without limits.

### Q. What is the difference between Express and NestJS?

> 🎤 **Interview answer:** "Express is a minimal, unopinionated framework. It gives you routing and middleware, and you decide the structure, which is flexible but means every team organises it differently. NestJS is a framework built on top of Express, or Fastify, with a fixed architecture: modules, controllers, providers, dependency injection, guards, pipes and interceptors, with TypeScript first. It's more boilerplate, but it's very consistent and testable for large teams and microservices. I've used Express for most of my projects and NestJS for the KindSwap Referral and Staking microservices."

Express is minimal – you decide the structure. NestJS is a framework on top of Express (or Fastify) with a fixed structure: modules, controllers, services, dependency injection, guards, pipes. Better for large teams. See [08 NestJS](08-nestjs.md).

### Q. Logging in production

> 🎤 **Interview answer:** "I use a structured logger like pino or winston with levels: error, warn, info and debug. Logs are in JSON format, so they can be searched and filtered. Every log line includes a request or correlation id, so I can trace one request across services. Logs go to CloudWatch or an ELK stack, with alarms on error spikes. And I'm strict that passwords, tokens, OTPs and card data are never logged. I mask or redact them."

Use a structured logger (`winston` / `pino`) with levels (error, warn, info, debug), JSON output, request id in every log line, and ship logs to CloudWatch/ELK. Never log passwords, tokens, card data.

### Q. What is `module.exports` vs `exports`?

> 🎤 **Interview answer:** "`exports` is just a shortcut reference to `module.exports`, and `require` always returns `module.exports`. Adding properties like `exports.foo = ...` works fine. But if I assign `exports = {...}`, I only change the local variable and break the link, so nothing gets exported. So when exporting a single function or class, I always use `module.exports = ...`."

`exports` is a shortcut reference to `module.exports`. If you assign `exports = {...}` you break the link; always use `module.exports = ...` when exporting a single thing.

### Q. What is the `__dirname`?

> 🎤 **Interview answer:** "`__dirname` gives the absolute path of the directory of the current file in CommonJS, and `__filename` gives the full file path. I use it to build reliable paths with `path.join(__dirname, 'templates')`, because relative paths depend on where the process was started from. In ES modules it doesn't exist, so we derive it from `import.meta.url`, or use `import.meta.dirname` in newer Node versions."

Absolute path of the folder of the current file (CommonJS). In ES modules use `path.dirname(fileURLToPath(import.meta.url))`.

### Q. How does `require` caching work?

> 🎤 **Interview answer:** "The first time a module is required, Node resolves the path, runs the file and caches its `module.exports`. Every later `require` of the same resolved path returns that same cached object without running the file again. That's why a DB connection or config module behaves like a singleton across the app. It also means module-level state is shared, which can surprise people in tests, where we sometimes reset modules."

The first `require` of a file runs it and caches the result. Later `require` calls return the same cached object. That is why a DB connection module is effectively a singleton.

### Q. Helmet?

> 🎤 **Interview answer:** "Helmet is an Express middleware that sets secure HTTP response headers in one line. It adds Content-Security-Policy, X-Frame-Options to stop clickjacking, Strict-Transport-Security for HTTPS, X-Content-Type-Options to stop MIME sniffing, and removes `X-Powered-By` so we don't advertise Express. It's not a complete security solution, but it's a cheap, standard first layer I add to every Express app."

Middleware that sets security HTTP headers (Content-Security-Policy, X-Frame-Options, HSTS, hides `X-Powered-By`). See [07 API Security](07-apis-backend.md#10-api-security).

---

[Home](README.md) | [← 02 JavaScript Coding](02-javascript-coding.md) | Next: [04 MongoDB & SQL →](04-mongodb-sql.md)
