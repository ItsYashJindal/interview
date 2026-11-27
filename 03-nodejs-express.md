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

> 🎤 **Interview answer:** "Node.js lets us run JavaScript on the server, outside the browser. It is made of two main parts:
> - **V8** – Google's engine that runs the JavaScript code.
> - **libuv** – a library that handles slow work like files and network in the background, and runs the event loop.
>
> Node is very good for APIs, real-time chat and microservices, where most of the time is spent waiting for the database or other services. I have been building backends with it for 4.5 years."

**Q. Explain Node.js architecture.**

> 🎤 **Interview answer:** "My JavaScript code runs on **one main thread**. When the code needs slow work, like a DB query, an API call or reading a file, Node does **not wait** for it. It hands the work to the background:
> - Network work goes to the operating system.
> - File, crypto and DNS work goes to libuv's **thread pool** (4 threads by default).
>
> When the work is done, its callback is put in a queue. The **event loop** picks it from the queue and runs it on the main thread.
>
> So: our code runs on one thread, but the slow work happens in the background. That's why one Node server can handle thousands of users at the same time, as long as we don't block the main thread with heavy CPU work."

```
Request → Main thread (your JS) → slow work? → hand to background (OS / thread pool)
                    ↑                                         |
                    └──── event loop runs the callback ←──────┘  (when work is done)
```

**Q. Why is Node good for APIs?**

> 🎤 **Interview answer:** "In an API, most of the time is spent **waiting** for the database, Redis or another service. Node doesn't sit idle during that wait. It serves other requests. So one process can handle many users with little memory. Also, the same language (JavaScript/TypeScript) on frontend and backend, and the huge npm library, make development faster."

**Q. When is Node NOT a good choice?**

> 🎤 **Interview answer:** "Node is not good for heavy CPU work like video encoding, big image processing or heavy calculations, if that work runs on the main thread. While it runs, every other request has to wait. If a Node service must do such work, I move it to a worker thread, a separate service, or a background queue. For example, I moved invoice and email work to Kafka consumers so the API stays fast."

**Q. What is blocking vs non-blocking code?**

> 🎤 **Interview answer:** "Blocking code makes the thread wait until the work is finished, like `fs.readFileSync`. In Node that means the **whole server** stops serving other users. Non-blocking code starts the work and moves on, and the result comes later through a callback or promise. So inside API handlers I never use `Sync` methods. `Sync` is fine only at startup, like reading a config file once."

```js
const data = fs.readFileSync("big.txt");   // blocking – whole server waits
fs.readFile("big.txt", (err, data) => {}); // non-blocking – server keeps serving others
```

---

### 2. Event-driven architecture

**Q. What does event-driven mean in Node?**

> 🎤 **Interview answer:** "Event-driven means the code **reacts when something happens**, like a request arrives or a file finishes reading, instead of waiting in one place.
>
> Node has an `EventEmitter` class for this. We register a listener with `on`, and trigger it with `emit`.
>
> The big benefit: the code that places an order just says 'order placed'. The email code and stock code listen and react. The order code doesn't need to know about them. Adding a new listener doesn't change the order code.
>
> Between services, I use the same idea with Kafka: the order service sends an event, and other services react to it."

```js
const EventEmitter = require("events");
const orderEvents = new EventEmitter();

orderEvents.on("orderPlaced", (order) => sendEmail(order));
orderEvents.on("orderPlaced", (order) => updateStock(order));

orderEvents.emit("orderPlaced", { id: 1 });
```

**At system level** (between services) the same idea is done with Kafka / RabbitMQ / Redis pub/sub. See [09 Kafka & Redis](09-kafka-redis-messaging.md).

**Be careful:** Listeners run one by one, right away, in the order they were added. If nobody listens to an `error` event and it is emitted, the app crashes.

---

### 3. Event loop in Node.js

**Q. Explain the phases of the Node.js event loop.**

> 🎤 **Interview answer:** "The Node event loop goes round and round in fixed steps called phases. The important ones are:
> 1. **Timers** – runs `setTimeout` and `setInterval` callbacks whose time is up.
> 2. **Poll** – runs callbacks for finished I/O, like a DB result or a file read. Most of our code runs here.
> 3. **Check** – runs `setImmediate` callbacks.
> 4. **Close** – runs close events, like a socket closing.
>
> Between every step, Node first runs all `process.nextTick` callbacks, and then all promise callbacks. That's why nextTick and promises always run before the next timer."

| Phase | What runs |
|---|---|
| 1. **Timers** | `setTimeout`, `setInterval` callbacks whose time is over |
| 2. **Pending callbacks** | Some system callbacks left from the last round |
| 3. **Idle/prepare** | Internal use only |
| 4. **Poll** | Callbacks of finished I/O (DB, file, network). Most of your code runs here |
| 5. **Check** | `setImmediate` callbacks |
| 6. **Close callbacks** | `socket.on("close")` etc. |

**Between every phase**, Node first empties:
1. the `process.nextTick` queue,
2. then the promise queue.

**Q. process.nextTick vs setImmediate vs setTimeout(0)?**

> 🎤 **Interview answer:** "From highest to lowest priority:
> 1. `process.nextTick` – runs right after the current code, before anything else.
> 2. Promise `.then` – runs right after nextTick.
> 3. `setTimeout(fn, 0)` – runs in the timers phase of the next round.
> 4. `setImmediate` – runs in the check phase, after I/O.
>
> If you keep calling `nextTick` again and again, I/O never gets a chance to run. So I prefer `setImmediate` when I just want to run something 'a bit later'."

**Q. How do you know the event loop is blocked?**

> 🎤 **Interview answer:** "The signs are: **all** APIs become slow at the same time, even a simple health check, and one CPU core is at 100%. To confirm, I measure event loop delay with `perf_hooks.monitorEventLoopDelay`, or use tools like clinic.js, New Relic or Datadog. Then I find the slow function with a profiler. Usually it's sync code, a huge `JSON.parse`, or a heavy loop. I fix it by making it async, splitting it into smaller pieces, or moving it to a worker thread."

---

### 4. Middleware

**Q. What is middleware in Express?**

> 🎤 **Interview answer:** "Middleware is a function that runs **between the request coming in and the response going out**. It gets `req`, `res` and `next`. It can:
> - read or change the request,
> - send a response and stop there,
> - call `next()` to go to the next middleware,
> - call `next(err)` to jump to the error handler.
>
> An Express app is basically a line of middleware functions. I use middleware for logging, parsing JSON, CORS, rate limiting, checking the JWT token, checking roles, validating input, and handling errors."

```js
const logger = (req, res, next) => {
  const start = Date.now();
  res.on("finish", () => {
    console.log(req.method, req.url, res.statusCode, Date.now() - start + "ms");
  });
  next(); // go to the next middleware
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

> 🎤 **Interview answer:** "Yes, a lot. Middleware runs in the order we add it.
> - `express.json()` must come before routes that read `req.body`, otherwise `req.body` is undefined.
> - Auth must come before protected routes.
> - The error handler must come **last**, after all routes."

**Q. What if you forget to call next()?**

> 🎤 **Interview answer:** "If a middleware doesn't send a response and doesn't call `next()`, the request just **hangs**. The client keeps waiting until it times out. The opposite mistake is calling `next()` after already sending a response, which gives the 'headers already sent' error. So I always write `return res.status(...).json(...)` when I stop early."

**Middleware factory example – role check:**
```js
const allowRoles = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user.role)) {
    return res.status(403).json({ message: "Forbidden" });
  }
  next();
};
router.delete("/users/:id", auth, allowRoles("admin"), deleteUser);
```

---

### 5. Error handling

**Q. How do you handle errors in Express?**

> 🎤 **Interview answer:** "I use four parts:
> 1. **My own error class**, `AppError`, which carries a message and a status code. So a service can do `throw new AppError('User not found', 404)`.
> 2. **An async wrapper** (`asyncHandler`), so errors from async routes go to the error handler automatically. I don't need try/catch in every controller. Express 5 does this by itself.
> 3. **One central error middleware**, added last. It logs the full error and sends a clean JSON response. For unknown errors, it hides the details and sends 'Something went wrong' with 500.
> 4. **Process-level handlers** for anything missed: `unhandledRejection` and `uncaughtException`. They log the error, and PM2 restarts the app if needed."

```js
// 1. Custom error
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // expected error, not a bug
  }
}

// 2. Async wrapper (Express 4) – sends any error to next()
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

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
  logger.error({ err, path: req.path });
  res.status(status).json({
    success: false,
    message: err.isOperational ? err.message : "Something went wrong", // hide internal details
  });
});

// 4. Process level
process.on("unhandledRejection", (reason) => { logger.error(reason); });
process.on("uncaughtException", (err) => { logger.error(err); process.exit(1); }); // PM2 restarts it
```

**Q. Operational errors vs programmer errors?**

> 🎤 **Interview answer:**
> - "**Operational errors** are expected problems: wrong input, item not found, DB timeout, a third-party API is down. We handle them properly and send the right status code.
> - **Programmer errors** are bugs in our code, like reading a property of `undefined`. We can't 'handle' them. We log them and fix the code. If one crashes the app, it is safest to restart, because the app may be in a bad state."

**Q. Errors in Express 4 async routes?**

> 🎤 **Interview answer:** "In Express 4, if an `async` route throws an error, Express does **not** catch it. Express 4 only catches normal (non-async) errors. So the request hangs. That's why we wrap routes with `asyncHandler`, or use the `express-async-errors` package. Express 5 fixed this and catches async errors by itself."

More in [11 Scenarios → Q5](11-scenario-questions.md#5-how-do-you-handle-errors-in-expressjs).

---

### 6. Authentication and authorization

**Q. Difference?**

> 🎤 **Interview answer:**
> - "**Authentication** = *Who are you?* For example, login with password, OTP or Google. If it fails → **401 Unauthorized**.
> - **Authorization** = *What are you allowed to do?* For example, only an admin can delete users, or a user can only see their own orders. If it fails → **403 Forbidden**.
>
> Authentication always comes first. In my projects, JWT middleware does authentication, and role checks do authorization."

**Q. Session-based vs token-based auth?**

> 🎤 **Interview answer:** "With **sessions**, the server saves the login info, usually in Redis, and the browser only keeps a session id in a cookie. Logout is easy: just delete the session.
>
> With **tokens (JWT)**, the server saves nothing. The client sends a signed token on every request, and any server can check it with the secret key. This works well for mobile apps and microservices. The downside: we can't easily cancel a JWT before it expires.
>
> So I use both ideas: short-lived access tokens, plus refresh tokens saved in the DB or Redis, which I can delete on logout."

| Session | Token (JWT) |
|---|---|
| Server saves session (Redis), browser keeps a session id cookie | Server saves nothing; client sends a signed token |
| Easy to log out (delete session) | Hard to cancel a token before it expires |
| All servers need the same session store | Any server can check the token |
| Good for classic websites | Good for mobile apps, SPAs, microservices |

**Q. How do you store passwords?**

> 🎤 **Interview answer:** "Never as plain text. I use **bcrypt** to hash them. A hash is a one-way scramble: we can't turn it back into the password. bcrypt also adds a random 'salt' to every password, so two users with the same password get different hashes. And it is slow on purpose, which makes guessing passwords very slow for attackers. At login, I use `bcrypt.compare`. I also never send the hash back in API responses."

```js
const hash = await bcrypt.hash(password, 10);        // 10 = how slow (cost)
const ok = await bcrypt.compare(inputPassword, hash); // true / false
```

**Q. Types of authorization**

> 🎤 **Interview answer:**
> - "**RBAC (role-based)** – permissions are given to roles like admin, seller, user. I used Auth0 RBAC in the KindSwap NestJS microservices.
> - **Ownership check** – a user can only access their own data. For example, I look for an order by both `id` and `userId`.
> - **ABAC (attribute-based)** – rules based on things like department, region or time.
>
> Most apps use roles plus ownership checks together."

**Q. OTP login flow (used in Rider Perfume)**

> 🎤 **Interview answer:** "1. The user enters their phone number. I limit OTP requests to about 3 per 10 minutes, so nobody can spam SMS.
> 2. I create a 6-digit OTP, save a **hash** of it in Redis for 5 minutes, and send it by SMS.
> 3. When the user enters the OTP, I compare it. I allow only about 5 wrong tries. After success, I delete the OTP so it can't be used again.
> 4. Then I create the user if new, and give an access token and a refresh token.
>
> Redis is perfect here because keys can expire automatically."

---

### 7. JWT and refresh tokens

**Q. What is a JWT?**

> 🎤 **Interview answer:** "JWT means JSON Web Token. It is a string with 3 parts separated by dots: `header.payload.signature`.
> - **Header** – which algorithm was used to sign it.
> - **Payload** – the data, like user id, role and expiry time.
> - **Signature** – made with a secret key. If anyone changes the payload, the signature won't match and the server rejects it.
>
> Important: the payload is only **encoded, not encrypted**. Anyone can read it. So I never put passwords or private data in it, only ids and roles."

**Q. How does JWT auth work?**

> 🎤 **Interview answer:** "1. At login, after checking the password, the server creates a token with the user id and role, valid for about 15 minutes, and sends it to the client.
> 2. The client sends this token on every request in the header: `Authorization: Bearer <token>`.
> 3. An auth middleware checks the token with `jwt.verify`. If it's valid, it puts the user in `req.user` and calls `next()`. If it's missing, wrong or expired, it returns 401.
>
> There is no DB call to check the token, so it's fast."

```js
// Login
const accessToken = jwt.sign(
  { sub: user.id, role: user.role },
  process.env.ACCESS_SECRET,
  { expiresIn: "15m" }
);

// Auth middleware
const auth = (req, res, next) => {
  const header = req.headers.authorization;     // "Bearer <token>"
  const token = header ? header.split(" ")[1] : null;
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

> 🎤 **Interview answer:** "Access tokens should be short, about 15 minutes, so if someone steals one, it stops working soon. But users don't want to log in every 15 minutes. So at login we also give a **refresh token** that lasts 7 to 30 days. It is used only for one thing: getting a new access token. I save refresh tokens in the DB or Redis, so I can delete them on logout. Every time a refresh token is used, I give a new one and cancel the old one. This is called **rotation**."

**Flow:**
1. Login → server returns access token (15 min) + refresh token (7 days).
2. Client calls APIs with the access token.
3. Access token expires → API returns 401 → client calls `/auth/refresh` with the refresh token.
4. Server checks the refresh token (valid + present in DB/Redis) → returns a new access token and a new refresh token.
5. Logout → delete the refresh token from DB/Redis.

**Q. Where to store tokens on the client?**

> 🎤 **Interview answer:** "The refresh token goes in an **httpOnly cookie**. httpOnly means JavaScript in the page cannot read it, so if a hacker injects a script (XSS attack), they can't steal it. The access token can be kept in memory (a JS variable). I avoid localStorage for tokens, because any injected script can read localStorage. On mobile, tokens go in the phone's secure storage."

**Q. How do you log out / revoke a JWT?**

> 🎤 **Interview answer:** "A JWT can't be deleted. It stays valid until it expires. So:
> - Access tokens are short (15 min), and on logout I delete the refresh token. So the user is fully logged out within minutes.
> - If I need to block a token **right now** (like after a password change), I save its id in a Redis 'blocklist' until it expires, and the middleware checks that list.
> - For 'logout from all devices', I keep a `tokenVersion` number on the user and put it in the token. I increase the number, so all old tokens stop matching."

**Q. Refresh token rotation and reuse detection?**

> 🎤 **Interview answer:** "Every time a refresh token is used, I give a new one and mark the old one as used. If someone sends an **old, already-used** refresh token again, it means it was stolen, because the real user only has the newest one. Then I cancel all tokens of that user and ask them to log in again."

**Q. HS256 vs RS256?**

> 🎤 **Interview answer:**
> - "**HS256** – one shared secret is used to create and to check the token. Simple. Fine when only one service creates and checks tokens.
> - **RS256** – a private key creates the token, a public key checks it. Better for microservices: every service can check tokens with the public key, but none of them can create tokens. Auth0 uses RS256, and that's how our NestJS services in KindSwap checked tokens."

Full implementation answer: [11 Scenarios → Q4](11-scenario-questions.md#4-how-would-you-implement-jwt-authentication).

---

### 8. Streams and buffers

**Q. What is a Buffer?**

> 🎤 **Interview answer:** "A Buffer is a piece of raw binary data (bytes) in memory. Node uses it when working with files, images, network data and encryption. We can convert a Buffer to text, base64 or hex. I used it to check webhook signatures from payment gateways."

```js
const buf = Buffer.from("hello");
buf.toString("base64"); // "aGVsbG8="
```

**Q. What is a Stream?**

> 🎤 **Interview answer:** "A stream lets us handle data **piece by piece** (in chunks), instead of loading all of it into memory at once. Like watching a video online: it plays while it downloads, instead of waiting for the full file.
>
> There are 4 types:
> - **Readable** – we read from it, like reading a file or an incoming request.
> - **Writable** – we write to it, like writing a file or sending a response.
> - **Duplex** – both, like a network socket.
> - **Transform** – changes data as it passes, like zipping (gzip)."

| Type | Example |
|---|---|
| Readable | `fs.createReadStream`, `req` (incoming request) |
| Writable | `fs.createWriteStream`, `res` (response) |
| Duplex (both) | TCP socket |
| Transform (changes data while passing) | `zlib.createGzip()`, CSV parser |

**Q. Why use streams?** Example: send a 2 GB file.

> 🎤 **Interview answer:** "With `readFile`, the full 2 GB is loaded into memory first. A few such requests and the server runs out of memory. With a stream, the file is sent in small pieces (about 64 KB at a time). Memory stays low, and the download starts immediately. I use the same idea for CSV exports of lakhs of rows and for uploading files to S3."

```js
// Bad – loads 2 GB into memory
app.get("/file", async (req, res) => {
  res.send(await fs.promises.readFile("big.zip"));
});

// Good – sends in small pieces, low memory
const { pipeline } = require("stream/promises");
app.get("/file", async (req, res, next) => {
  try {
    await pipeline(fs.createReadStream("big.zip"), res);
  } catch (err) {
    next(err);
  }
});
```

**Q. What is backpressure?**

> 🎤 **Interview answer:** "Backpressure is when data comes in **faster** than we can send it out. For example, reading from a fast disk but sending to a user on a slow network. If we don't handle it, the extra data piles up in memory. `pipe` and `pipeline` handle this for us: they pause reading when the writer is full, and continue when it is ready again."

**Q. pipe vs pipeline?**

> 🎤 **Interview answer:** "Both connect streams. But if something fails, `pipe` does not pass the error along and does not close the other streams, so files can stay open. `pipeline` handles errors properly and closes all streams. So I always use `pipeline`."

**Real uses:** CSV export of lakhs of rows, file upload to S3, creating PDFs, reading big log files.

---

### 9. Worker threads vs child processes vs cluster

> 🎤 **Interview answer:** "They solve different problems:
> - **Worker threads** – run heavy JavaScript work in a separate thread, inside the same app. Use it for CPU-heavy work like image resizing, PDF creation or hashing, so the main thread stays free.
> - **Child process** – starts a completely separate program. Use it to run things like ffmpeg, a Python script or a shell command.
> - **Cluster** – runs many copies of the whole app, one per CPU core, all on the same port. Use it to handle more HTTP requests.
>
> In production I used PM2 in cluster mode on EC2 to use all CPU cores."

| | Worker threads | Child process | Cluster |
|---|---|---|---|
| What | New thread inside the same app | A completely new program | Many copies of your app |
| Memory | Can share memory | Separate memory | Separate memory |
| Use for | CPU-heavy JS work (image resize, PDF, hashing) | Running other programs (ffmpeg, Python, shell) | Using all CPU cores for more requests |
| If it crashes | Can affect the app | Isolated | Other copies keep working |

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
- `spawn` – gives output little by little. Good for big output.
- `exec` – runs in a shell, gives all output at the end. Small output only.
- `execFile` – like exec, but without a shell (safer).
- `fork` – for running another Node file, with a way to send messages between them.

**Q. How do you use all CPU cores?**

> 🎤 **Interview answer:** "Node runs our code on one core. To use all cores, we run many copies of the app. On EC2 I used PM2 cluster mode: `pm2 start app.js -i max`. It starts one copy per core on the same port and restarts any copy that crashes. With Docker we run many containers behind a load balancer instead. Since the copies don't share memory, anything shared, like sessions, rate-limit counts and cache, must be kept in Redis."

---

### 10. API rate limiting

**Q. What is rate limiting and why?**

> 🎤 **Interview answer:** "Rate limiting means limiting how many requests one user or IP can make in a given time, for example 100 per minute. It protects against password guessing, OTP/SMS spam, scraping and traffic floods. It also protects expensive APIs. When the limit is crossed, we return **429 Too Many Requests**. In Rider Perfume I added strict limits on login and OTP routes."

**Algorithms (simple explanation):**
| Algorithm | How it works | Note |
|---|---|---|
| Fixed window | Count requests in each minute (12:00–12:01) | Simple, but a burst is possible at the edge of two minutes |
| Sliding window | Count requests in the last 60 seconds from now | More accurate |
| Token bucket | A bucket gets N tokens per second; each request uses one token | Allows small bursts – most common |
| Leaky bucket | Requests go out at a fixed speed, like water from a hole | Smooth flow |

**Q. Why Redis for rate limiting?**

> 🎤 **Interview answer:** "In production we run many servers. If each server counts in its own memory, a user gets the limit multiplied by the number of servers. Redis gives **one shared counter** for all servers. `INCR` adds 1 safely, even when many requests come at the same moment, and `EXPIRE` deletes the key automatically when the time window ends."

```js
// Fixed window with Redis: max 100 requests per minute per IP
async function rateLimit(req, res, next) {
  const minute = Math.floor(Date.now() / 60000);
  const key = `rl:${req.ip}:${minute}`;

  const count = await redis.incr(key);           // add 1
  if (count === 1) await redis.expire(key, 60);  // delete after 60 s

  if (count > 100) return res.status(429).json({ message: "Too many requests" });
  next();
}
```
Or use the `express-rate-limit` package with the `rate-limit-redis` store.

**Where to rate limit:** Nginx / API gateway for general protection + inside the app for per-user or per-route rules.

---

### 11. Caching with Redis

**Q. What is Redis?**

> 🎤 **Interview answer:** "Redis is a very fast key-value store that keeps data in **RAM** (memory). Reads and writes take less than a millisecond. It supports strings, hashes, lists, sets and sorted sets, and keys can expire automatically (TTL). I have used it for caching, saving OTPs, rate limiting, sending messages between services (pub/sub), locks, and choosing a leader among 4–5 servers in KindSwap. I don't use it as the main database, only as a fast helper layer."

**Q. Cache-aside pattern (most common)**

> 🎤 **Interview answer:** "When reading:
> 1. First check Redis.
> 2. If found, return it.
> 3. If not found, read from the DB, save it in Redis with an expiry time, and return it.
>
> When updating: I update the DB and then **delete** the Redis key, so the next read gets fresh data. The expiry time is a safety net in case I miss a delete. I used this for product lists and categories in Rider Perfume."

```js
async function getProduct(id) {
  const key = `product:${id}`;

  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);              // found in cache

  const product = await Product.findById(id).lean();  // not found → DB
  if (product) await redis.set(key, JSON.stringify(product), "EX", 300); // keep 5 min
  return product;
}

// On update – delete the old cache
async function updateProduct(id, data) {
  await Product.updateOne({ _id: id }, data);
  await redis.del(`product:${id}`);
}
```

**Q. Other caching strategies**

> 🎤 **Interview answer:**
> - "**Write-through** – every write goes to the cache and the DB together. Cache is always fresh, but writes are slower.
> - **Write-back** – write to the cache first, save to the DB later. Very fast, but data can be lost if the cache crashes.
> - **Read-through** – the cache itself loads from the DB when data is missing.
>
> For most APIs, cache-aside with an expiry time is the practical choice."

**Q. What do you cache?**

> 🎤 **Interview answer:** "I cache data that is **read often, changes rarely, and is slow to get**: product lists, categories, settings, user profiles, results of heavy queries, plus sessions and OTPs. I don't cache things that must be exactly correct right now, like wallet balance, payment status, or stock during checkout."

**Q. Cache invalidation problems**

> 🎤 **Interview answer:**
> - "**Old data** – the cache has an old value. Fix: expiry time + delete the key on update.
> - **Cache stampede** – a popular key expires and 1000 requests hit the DB at the same time. Fix: let only one request rebuild the cache (using a lock), or add a small random extra time to each expiry so keys don't all expire together.
> - **Missing IDs** – requests for IDs that don't exist always miss the cache and hit the DB. Fix: cache an empty result for a short time.
> - **Memory full** – set a memory limit in Redis and let it remove the least-used keys (`allkeys-lru`)."

More Redis (pub/sub, leader election, locks) in [09 Kafka & Redis](09-kafka-redis-messaging.md).

---

## Extra Node/Express questions

### Q. What is libuv?

> 🎤 **Interview answer:** "libuv is a library written in C that Node uses. It gives Node the event loop and a thread pool (4 threads by default) for file, DNS and crypto work. It's what lets Node do slow work in the background without blocking."

### Q. Is Node.js single-threaded?

> 🎤 **Interview answer:** "Partly. **Our JavaScript** runs on one main thread. But Node itself uses other threads in the background: libuv's thread pool for files and crypto, and we can also create worker threads ourselves. So the best answer is: 'one thread runs our code, but slow work happens on other threads in the background'."

### Q. What is package.json vs package-lock.json?

> 🎤 **Interview answer:**
> - "`package.json` – project info, scripts, and which **range** of versions we accept, like `^4.18.0`.
> - `package-lock.json` – the **exact** version of every installed package, so every developer and every server installs exactly the same thing. We always commit it and use `npm ci` on servers.
>
> `^1.2.3` allows updates like 1.3.0. `~1.2.3` allows only small fixes like 1.2.4."

### Q. dependencies vs devDependencies?

> 🎤 **Interview answer:** "`dependencies` are needed when the app runs, like express and mongoose. `devDependencies` are needed only while developing, like jest, eslint, typescript and nodemon. For production I install with `npm ci --omit=dev` to keep it smaller."

### Q. How do you manage environment config?

> 🎤 **Interview answer:** "Locally I use a `.env` file with `dotenv`. It is never committed to Git. I commit a `.env.example` with empty values. In production, values come from real environment variables or AWS Secrets Manager. At startup, I check that all required values exist. If the DB URL or JWT secret is missing, the app stops immediately with a clear error, instead of failing later."

### Q. What is CORS?

> 🎤 **Interview answer:** "CORS is a browser security rule. By default, a website on one domain can't call an API on a different domain, unless the API says it's allowed by sending special headers like `Access-Control-Allow-Origin`. For some requests, like PUT or DELETE, the browser first sends a test request called OPTIONS (preflight) to ask permission. In Express I use the `cors` package with a list of our allowed frontend domains. CORS only applies in browsers. Postman and server-to-server calls are not affected."

```js
app.use(cors({ origin: ["https://riderbeauty.com"], credentials: true }));
```

### Q. req.params vs req.query vs req.body

> 🎤 **Interview answer:**
> - "`req.params` – values in the URL path. `/users/:id` → `req.params.id`.
> - `req.query` – values after `?`. `/users?page=2` → `req.query.page`. Used for filters and pagination.
> - `req.body` – data sent in POST or PUT, like JSON. Needs `express.json()`.
>
> All three come from the user, so I validate all of them."

### Q. app.use vs app.get?

> 🎤 **Interview answer:** "`app.use` works for **any** method (GET, POST, etc.) and any path that **starts with** the given path. It's used for middleware and routers. `app.get` works only for GET requests on that exact path."

### Q. Express Router

> 🎤 **Interview answer:** "Router is like a mini-app to group related routes. I make one router per module, like orders or users, and connect it with `app.use('/api/v1/orders', orderRouter)`. It keeps the code organised, and I can add middleware like auth to the whole group."

```js
const router = express.Router();
router.get("/", listOrders);
router.post("/", auth, createOrder);
app.use("/api/v1/orders", router);
```

### Q. How do you handle file uploads?

> 🎤 **Interview answer:** "For normal uploads I use `multer`. I check the file type and size, upload it to S3, and save only the S3 key in the DB. For big files, the backend creates a **pre-signed S3 URL** (a temporary upload link), and the client uploads straight to S3. The file never passes through our server, which saves memory and bandwidth."

### Q. Graceful shutdown

> 🎤 **Interview answer:** "When we deploy, PM2 or Docker sends a `SIGTERM` signal to the old app. In my handler I:
> 1. stop taking new requests,
> 2. let the running requests finish,
> 3. close DB, Redis and Kafka connections,
> 4. exit.
>
> I also add a 10-second timer to force exit if something gets stuck. Without this, deploys can cut requests in the middle."

```js
process.on("SIGTERM", () => {
  server.close(async () => {          // stop new requests, wait for running ones
    await mongoose.connection.close();
    await redis.quit();
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10000); // force exit after 10 s
});
```

### Q. How to find a memory leak in Node?

> 🎤 **Interview answer:** "First I confirm it: memory keeps growing and never comes down, which I can see in PM2 or CloudWatch. Then I run the app with `--inspect`, take two heap snapshots in Chrome DevTools a few minutes apart, and compare which objects keep growing. Common causes: a global array or map that keeps growing, event listeners never removed, timers never cleared. The fix is usually a size limit or expiry, removing listeners, or moving the cache to Redis."

### Q. What is the difference between Express and NestJS?

> 🎤 **Interview answer:** "Express is small and simple. It gives routing and middleware, and we decide the folder structure ourselves. NestJS is built on top of Express and gives a **fixed structure**: modules, controllers, services, dependency injection, guards and pipes, with TypeScript. More code to write, but very organised for big teams. I used Express in most projects and NestJS for the KindSwap Referral and Staking microservices."

See [08 NestJS](08-nestjs.md).

### Q. Logging in production

> 🎤 **Interview answer:** "I use a logger like `pino` or `winston` with levels: error, warn, info, debug. Logs are in JSON format so they are easy to search. Every log line has a request id, so I can follow one request across services. Logs go to CloudWatch, with alerts when errors increase. I never log passwords, tokens, OTPs or card details."

### Q. What is `module.exports` vs `exports`?

> 🎤 **Interview answer:** "`exports` is just a shortcut that points to `module.exports`. `require` always returns `module.exports`. Adding things like `exports.foo = ...` is fine. But if I write `exports = {...}`, I break the link and nothing is exported. So to export one function or class, I always use `module.exports = ...`."

### Q. What is the `__dirname`?

> 🎤 **Interview answer:** "`__dirname` gives the full folder path of the current file. I use it to build file paths safely, like `path.join(__dirname, 'templates')`. In ES modules it doesn't exist, so we use `import.meta.dirname` in newer Node versions."

### Q. How does `require` caching work?

> 🎤 **Interview answer:** "The first time a file is required, Node runs it and saves (caches) what it exports. Every later `require` of the same file gives back the **same saved object**, without running the file again. That's why a DB connection file is created only once and shared across the app."

### Q. Helmet?

> 🎤 **Interview answer:** "Helmet is an Express middleware that adds security headers to every response in one line. For example, it stops our site from being loaded inside another site's iframe, forces HTTPS, and hides the 'X-Powered-By: Express' header. It's a cheap first layer of security that I add to every Express app."

See [07 API Security](07-apis-backend.md#10-api-security).

---

[Home](README.md) | [← 02 JavaScript Coding](02-javascript-coding.md) | Next: [04 MongoDB & SQL →](04-mongodb-sql.md)
