[Home](README.md) | [← 08 NestJS](08-nestjs.md) | Next: [10 System Design →](10-system-design.md)

# 09 – Kafka, Redis & Messaging (added – heavy on your resume)

Your resume mentions Kafka with dead-letter queues, Redis leader election, Redis pub/sub, RabbitMQ and SQS. Interviewers **will** go deep here. Know these well.

## Contents
- **Message queues – basics**
  - [Why use a message queue](#why-use-a-message-queue)
  - [Queue vs pub/sub](#queue-vs-pubsub)
- **Kafka**
  - [Kafka core concepts](#kafka-core-concepts)
  - [How ordering works](#how-ordering-works)
  - [Delivery guarantees](#delivery-guarantees)
  - [Retries and dead-letter queue (DLQ)](#retries-and-dead-letter-queue-dlq)
  - [Idempotent consumers](#idempotent-consumers)
  - [Kafka with Node.js (kafkajs)](#kafka-with-nodejs-kafkajs)
  - [Kafka vs RabbitMQ vs SQS](#kafka-vs-rabbitmq-vs-sqs)
- **Redis**
  - [Redis data types and uses](#redis-data-types-and-uses)
  - [Redis pub/sub](#redis-pubsub)
  - [Leader election with Redis](#leader-election-with-redis)
  - [Distributed lock](#distributed-lock)
  - [Other Redis questions](#other-redis-questions)
- [Outbox pattern](#outbox-pattern)

---

## Message queues – basics

### Why use a message queue

> 🎤 **Interview answer:** "A message queue lets one service **drop a message and move on**, while another service picks it up and does the work later. It's like leaving a note in a box for someone else to handle.
>
> Benefits:
> - **Faster APIs** – slow work like emails, SMS, invoices and notifications is done in the background.
> - **Services don't depend on each other** – the order service doesn't need to know who reads the message.
> - **Handles traffic spikes** – messages wait in the queue, and workers process them at their own speed.
> - **Nothing is lost** – if a worker is down, messages wait and are processed later.
>
> My real example: at Appsinvo, the order API was sending emails and SMS itself. It was slow, and if the email provider failed, the message was lost. I moved this to Kafka. Now the order API just sends an 'order placed' event and returns. A separate consumer sends the notifications, with retries and a dead-letter queue."

### Queue vs pub/sub

> 🎤 **Interview answer:**
> - "**Queue** – each message is handled by **only one** worker. Many workers share the load. Like a ticket counter: each customer goes to one counter. Examples: SQS, RabbitMQ queue.
> - **Pub/sub** – each message goes to **every** subscriber. Like a WhatsApp group: everyone gets the message. Examples: Redis pub/sub, SNS.
>
> Kafka can do both: inside one consumer group it acts like a queue, and different consumer groups each get every message, like pub/sub."

---

## Kafka

### Kafka core concepts

> 🎤 **Interview answer:** "Kafka is like a **long, saved list of messages** (a log). Messages stay saved on disk for some days, even after they're read, so they can be read again later.
>
> - **Producer** – sends messages.
> - **Topic** – a named channel, like `order.placed`.
> - **Partition** – a topic is split into parts, so many consumers can work in parallel. Each partition keeps messages in order.
> - **Offset** – the position number of a message in a partition.
> - **Consumer group** – a team of consumers sharing the work. Each partition is read by **only one** consumer in the team. So 6 partitions = max 6 consumers working at once.
> - **Commit offset** – the consumer saves 'I've finished up to message X', so after a restart it continues from there.
>
> Different consumer groups each get **all** the messages. So our notification service and an analytics service can both read `order.placed` independently."

| Term | Simple meaning |
|---|---|
| **Broker** | A Kafka server. A cluster has many |
| **Topic** | A named channel of messages, e.g. `order.placed` |
| **Partition** | A part of a topic. Lets many consumers work in parallel. Ordered |
| **Offset** | Position number of a message inside a partition |
| **Producer** | Sends messages |
| **Consumer** | Reads messages |
| **Consumer group** | A team of consumers sharing the work. Each partition → one consumer in the team |
| **Replication factor** | How many copies of each partition are kept on different servers (e.g. 3) |
| **Commit offset** | Consumer saves "done up to X" so it continues from there after restart |

**Scaling rule:** max useful consumers in a group = number of partitions. 6 partitions → up to 6 consumers.

### How ordering works

> 🎤 **Interview answer:** "Kafka keeps order **only inside one partition**, not across the whole topic. When sending, I give each message a **key**. All messages with the same key go to the same partition, so they stay in order. I use `orderId` as the key. So all events of one order, like placed → paid → shipped, are processed in the right order. Different orders still run in parallel on different partitions."

### Delivery guarantees

> 🎤 **Interview answer:** "There are three levels:
> - **At-most-once** – a message may be lost, but never repeated. (Save progress first, then process.)
> - **At-least-once** – a message is never lost, but may come **twice**. (Process first, then save progress. If the consumer crashes in between, the message comes again.) This is the most common, and what I used. So the consumer must handle duplicates safely.
> - **Exactly-once** – Kafka supports it inside Kafka. But for outside actions like sending an email or writing to a DB, we still need to handle duplicates ourselves.
>
> On the sending side, I use `acks=all` (wait until all copies are saved) and turn on the idempotent producer, so messages aren't lost or duplicated on the way to Kafka."

| Type | Meaning | How |
|---|---|---|
| At-most-once | May lose, never repeat | Save progress before processing |
| **At-least-once** (most common) | Never lose, may repeat | Process first, then save progress → **consumer must handle duplicates** |
| Exactly-once | No loss, no repeat | Kafka transactions (only inside Kafka) |

### Retries and dead-letter queue (DLQ)

> 🎤 **Interview answer:** "Some messages fail, for example the email provider is down, or the data is bad.
> - If the consumer **keeps retrying forever**, all messages behind it are stuck.
> - If it **skips** the message, the message is lost.
>
> So at Appsinvo I did this:
> 1. When a message fails, I **retry it a few times** (about 3), waiting a bit longer each time. I do this by sending it to a separate **retry topic** with an 'attempts' count, so the main topic isn't blocked.
> 2. If it still fails after 3 tries, I send it to a **dead-letter queue (DLQ)** – a separate topic for failed messages – together with the error, the attempt count and the time.
> 3. Then the main flow continues.
> 4. We get an **alert** when the DLQ is not empty. We find the cause, fix it, and **send the messages again** to the main topic.
>
> I also treat errors differently: temporary errors like timeouts are retried, but permanent errors like bad data go straight to the DLQ, because retrying won't help."

```js
await consumer.run({
  eachMessage: async ({ topic, message }) => {
    const attempts = Number(message.headers?.attempts?.toString() || 0);

    try {
      await handleNotification(JSON.parse(message.value.toString()));
    } catch (err) {
      if (attempts < 3) {
        // try again later
        await producer.send({
          topic: "notifications.retry",
          messages: [{
            key: message.key,
            value: message.value,
            headers: { attempts: String(attempts + 1) },
          }],
        });
      } else {
        // give up → dead-letter queue
        await producer.send({
          topic: "notifications.dlq",
          messages: [{
            key: message.key,
            value: message.value,
            headers: { error: err.message, originalTopic: topic, failedAt: new Date().toISOString() },
          }],
        });
      }
    }
  },
});
```
**Poison message** = a message that will **always** fail (e.g. bad format). The DLQ stops it from blocking everything else.

### Idempotent consumers

> 🎤 **Interview answer:** "With at-least-once delivery, the same message can arrive twice. So every consumer must be **safe to run twice**. How I do it:
> - Every event has a unique `eventId`.
> - Before processing, I check if that id was already processed (a DB table with a unique rule, or a Redis set).
> - Where possible, I use a natural unique field, like a unique `orderId` in the receipts table, so a second insert simply fails.
>
> That's how we made sure a customer never got two emails for the same order."

```js
async function handle(event) {
  const isNew = await redis.set(`processed:${event.eventId}`, 1, "NX", "EX", 86400);
  if (!isNew) return;          // already processed → skip
  await sendOrderEmail(event);
}
```

### Kafka with Node.js (kafkajs)

> 🎤 **Interview answer:** "In Node I used `kafkajs`. I create a Kafka client with the broker addresses. The **producer** sends messages with a key (order id, for ordering) and a JSON value that includes an `eventId`. The **consumer** joins a group, like `notification-service`, subscribes to the topic, and processes each message inside try/catch, which leads to the retry and DLQ logic. On shutdown, I disconnect the consumer properly so its progress is saved."

```js
const { Kafka } = require("kafkajs");
const kafka = new Kafka({ clientId: "order-service", brokers: ["broker1:9092"] });

// Producer
const producer = kafka.producer({ idempotent: true });
await producer.connect();
await producer.send({
  topic: "order.placed",
  messages: [{
    key: String(order.id),
    value: JSON.stringify({ eventId, orderId: order.id, userId }),
  }],
});

// Consumer
const consumer = kafka.consumer({ groupId: "notification-service" });
await consumer.connect();
await consumer.subscribe({ topic: "order.placed", fromBeginning: false });
await consumer.run({ eachMessage: async ({ message }) => { /* process */ } });
```

**Consumer lag:**

> 🎤 **Interview answer:** "Consumer lag means **how far behind** the consumer is: how many messages are waiting that it hasn't processed yet. It's the main health number for consumers. If lag keeps growing, the consumers are too slow. Then I add more consumers (up to the number of partitions), or make processing faster. I set alerts on lag, so we know before users notice late notifications."

**Rebalancing:**

> 🎤 **Interview answer:** "A rebalance happens when a consumer joins or leaves the group. Kafka then re-assigns the partitions among the consumers, and reading pauses for a moment. Too many rebalances slow things down and can cause duplicate processing. So I keep processing fast, shut down consumers properly, and make handlers safe to run twice."

### Kafka vs RabbitMQ vs SQS

> 🎤 **Interview answer:**
> - "**Kafka** – a saved log. Messages stay for days and can be **read again**. Very high speed. Many consumer groups can read the same events. Best for event streaming and when many services need the same events. Hardest to run yourself.
> - **RabbitMQ** – a classic message broker. Messages are **deleted after they're processed**. Very flexible routing, and built-in dead-letter support. Great for task queues.
> - **SQS** – AWS's fully managed queue. **Nothing to run or maintain**. Simple, with built-in DLQ. Great for simple background jobs on AWS.
>
> At Appsinvo we chose Kafka because many consumers needed the same order events, and being able to replay them was useful. For a simple job queue on AWS, I'd pick SQS."

| | Kafka | RabbitMQ | AWS SQS |
|---|---|---|---|
| Model | Saved log (can replay) | Message broker with queues | Managed simple queue |
| After reading | Kept until retention ends | Deleted after ack | Deleted after consumer deletes it |
| Speed | Very high | High | High |
| Ordering | Per partition | Per queue | Only in FIFO queues |
| DLQ | Build yourself (DLQ topic) | Built-in | Built-in |
| Work to run it | High | Medium | None |
| Good for | Event streaming, many consumers | Task queues, complex routing | Simple background jobs on AWS |

**SQS key points:**

> 🎤 **Interview answer:** "When a worker takes a message from SQS, the message becomes **hidden** for some time (visibility timeout). If the worker finishes, it deletes the message. If the worker crashes and doesn't delete it, the message **appears again** after the timeout and is retried. So processing must be safe to repeat, and the timeout must be longer than the processing time. After a set number of failures (`maxReceiveCount`), SQS moves the message to a DLQ automatically. FIFO queues keep order and remove duplicates."

---

## Redis

### Redis data types and uses

> 🎤 **Interview answer:** "Redis has different data types, and picking the right one is the main design choice:
> - **String** – simple value. For cache, counters, rate limits, OTPs with expiry.
> - **Hash** – an object with fields, like a session.
> - **List** – a simple queue or 'latest 10 items'.
> - **Set** – unique values, like online users.
> - **Sorted set** – values with a score, always sorted. For leaderboards.
> - **Stream** – a saved message queue with consumer groups.
> - **Pub/sub** – live broadcast.
>
> I've used strings for cache, OTP, rate limiting and the leader lock, and pub/sub for events between services and price updates."

| Type | Commands | Use case |
|---|---|---|
| String | `SET GET INCR EXPIRE` | Cache, counters, rate limit, OTP |
| Hash | `HSET HGET HGETALL` | Session / object fields |
| List | `LPUSH RPOP LRANGE` | Simple queue, latest N items |
| Set | `SADD SISMEMBER SMEMBERS` | Unique visitors, online users |
| Sorted set | `ZADD ZRANGE ZINCRBY` | Leaderboards, sliding-window rate limit |
| Stream | `XADD XREADGROUP XACK` | Saved message queue with consumer groups |
| Pub/Sub | `PUBLISH SUBSCRIBE` | Live broadcast |
| Geo | `GEOADD GEOSEARCH` | Nearby search |

**Q. Why is Redis fast?**

> 🎤 **Interview answer:** "Four reasons: all data is in **RAM**, not on disk. It uses simple, efficient data structures. It runs commands **one at a time** on a single thread, so there's no time wasted on locks. And it handles thousands of connections efficiently. Most operations take less than a millisecond."

**Q. Redis is single-threaded – is that a problem?**

> 🎤 **Interview answer:** "Running commands one at a time is actually a **benefit**: each command runs fully before the next one starts, so it can't be interrupted. That's why `INCR` for rate limiting and `SET NX` for locks are safe without extra locking. The real risk is **slow commands**, like `KEYS *` or reading a huge set. While they run, every other client waits. So I avoid them and use `SCAN` instead of `KEYS`."

### Redis pub/sub

> 🎤 **Interview answer:** "With Redis pub/sub, a publisher sends a message to a **channel**, and everyone subscribed to that channel **right now** gets it instantly.
>
> The big limitation: it's **fire and forget**. Messages are **not saved**. If a subscriber is offline or reconnecting at that moment, it simply misses the message. There's no retry.
>
> So it's great when **the next message replaces the old one**, like live crypto prices, typing indicators, or telling all servers to clear a cache. At KindSwap we also used it between the Referral and Staking services. For events that must never be lost, like rewards, I'd recommend Redis Streams, Kafka or RabbitMQ, or saving the event in the DB with a retry job."

```js
// Subscriber (needs its own connection)
const sub = redis.duplicate();
await sub.subscribe("price-updates");
sub.on("message", (channel, msg) => io.emit("price", JSON.parse(msg)));

// Publisher
await redis.publish("price-updates", JSON.stringify({ symbol: "SOL", price: 182.4 }));
```
**Limitations:**
- No storage – if a subscriber is offline, it misses the message.
- No acknowledgement, no retry.
- Need guaranteed delivery → **Redis Streams**, Kafka or RabbitMQ.

### Leader election with Redis

> 🎤 **Interview answer:** "This is one of my favourite problems. At KindSwap, the price service ran on 4–5 servers for safety. **Every** server fetched crypto prices and sent updates. So users got **duplicate** price updates, and we called the price provider 5 times more than needed.
>
> The goal: only **one** server (the 'leader') should send updates, and if it dies, another one should take over automatically.
>
> How I solved it with Redis:
> 1. Every server tries to create the same Redis key with its own id: `SET price:leader <myId> NX PX 10000`.
>    - `NX` means 'only create it if it doesn't exist'. So **only one server can win**. That one is the leader.
>    - `PX 10000` means the key **expires in 10 seconds**. So if the leader crashes, the key disappears by itself.
> 2. The leader **renews** the key every 3 seconds, to say 'I'm still alive'. It renews only if the key still has **its own id**, so it can never extend another server's lock.
> 3. The other servers keep trying to create the key every few seconds. When the leader dies, the key expires within 10 seconds, and one of them becomes the new leader.
> 4. If renewal fails, the old leader **stops sending immediately**.
> 5. On a normal shutdown, the leader deletes its key, so another server takes over instantly.
>
> We logged leader changes in CloudWatch, with an alert if there was no leader. Result: zero duplicate updates and automatic failover."

**Steps in short:**
1. Every server has a unique id.
2. Each tries: `SET price:leader <serverId> NX PX 10000`
   - `NX` – only one server can create it → that one is the leader.
   - `PX 10000` – expires in 10 s, so a dead leader frees the lock automatically.
3. The leader renews every ~3 s, but **only if the key still has its id**. This check + renew is done in one step using a small Lua script, so nothing can happen in between:
   ```lua
   -- "if the key still has my id, extend it; otherwise do nothing"
   if redis.call("GET", KEYS[1]) == ARGV[1] then
     return redis.call("PEXPIRE", KEYS[1], ARGV[2])
   else
     return 0
   end
   ```
4. Followers keep trying step 2. If the leader dies, one of them takes over in ≤10 s.
5. If renewal returns 0 → the old leader stops publishing.
6. On shutdown → the leader deletes its own key → instant takeover.

```ts
async function tryBecomeLeader() {
  const ok = await redis.set(KEY, SERVER_ID, "PX", TTL_MS, "NX");
  if (ok === "OK") isLeader = true;
}

async function renew() {
  const res = await redis.eval(RENEW_LUA, 1, KEY, SERVER_ID, TTL_MS);
  if (res === 0) isLeader = false;   // lost leadership
}

setInterval(async () => {
  if (isLeader) await renew();
  else await tryBecomeLeader();
}, 3000);
```

**Follow-up questions you may get:**

> 🎤 **Interview answer:**
> - "**What if the leader freezes for a while and its key expires?** Then for a moment two servers might send updates. For price updates this is low risk. I keep the renewal time (3 s) much shorter than the expiry (10 s). For critical work, I'd add a 'fencing token': a number that increases with every new leader, and receivers ignore messages with an older number.
> - **What if Redis goes down?** Nobody can become leader, so prices pause. That's safer than duplicates. We use managed Redis with automatic failover to reduce this risk.
> - **Why not just run one server?** Then if it dies, there are no prices at all. Also, the other servers also handle API and socket traffic.
> - **Other options?** etcd, ZooKeeper, Consul, Kubernetes Leases, or Postgres advisory locks."

### Distributed lock

> 🎤 **Interview answer:** "A distributed lock makes sure **only one server does something at a time**. It's the same Redis idea: `SET lock:key <myId> NX PX <time>`. If I get OK, I have the lock.
>
> I use it for:
> - A cron job that should run on **only one** server, not all of them.
> - Stopping two requests from processing the **same payment** or the **same coupon** at the same time.
> - Letting only one request rebuild a popular cache key.
>
> Two rules: always set an expiry, so a crashed server doesn't hold the lock forever. And when releasing, delete the key **only if it still has my id**, so I never delete someone else's lock."

```js
const lockId = crypto.randomUUID();
const got = await redis.set("lock:daily-report", lockId, "NX", "PX", 60000);
if (got === "OK") {
  try {
    await runDailyReport();
  } finally {
    // release only if it's still my lock (done with a small Lua script in real code)
    if ((await redis.get("lock:daily-report")) === lockId) {
      await redis.del("lock:daily-report");
    }
  }
}
```

### Other Redis questions

> 🎤 **Interview answer:**
> - "**Saving to disk:** RDB takes a snapshot every few minutes (fast restart, but can lose the last few minutes). AOF logs every write (safer, bigger files). You can use both.
> - **High availability:** replicas keep copies. Sentinel watches and promotes a replica if the main one dies. Redis Cluster splits data across many servers.
> - **Redis vs Memcached:** Redis has many data types, saving to disk, pub/sub and replication. Memcached is only a simple key-value cache.
> - **Transactions:** `MULTI` / `EXEC` run commands together. Lua scripts also run as one step.
> - **Pipelining:** send many commands at once without waiting for each reply, to save time.
> - **Big keys / hot keys:** a very large value, or one key with huge traffic, slows Redis down."

---

## Outbox pattern

> 🎤 **Interview answer:** "The outbox pattern solves this problem: I save an order in the DB, and then send an 'order placed' event to Kafka. These are **two separate steps**. If the app crashes **between** them, the order is saved but the event is never sent. Or the event is sent but the DB save failed.
>
> The fix:
> 1. In the **same DB transaction** as the order, I also save the event in an `outbox` table. Since it's one transaction, both are saved, or neither.
> 2. A separate background worker reads new rows from the outbox table, sends them to Kafka, and marks them as 'sent'.
> 3. A row might be sent twice if the worker crashes at the wrong moment, so consumers must handle duplicates.
>
> This guarantees: if the order is saved, the event **will** be sent."

```
BEGIN
  INSERT INTO orders ...
  INSERT INTO outbox (event_type, payload, status) VALUES ('order.placed', {...}, 'pending')
COMMIT

Worker (every few seconds):
  read outbox rows with status = 'pending'
  → send to Kafka
  → mark as 'sent'
```

---

[Home](README.md) | [← 08 NestJS](08-nestjs.md) | Next: [10 System Design →](10-system-design.md)
