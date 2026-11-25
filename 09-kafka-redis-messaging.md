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

> 🎤 **Interview answer:** "A message queue lets one service hand off work and move on, while another service processes it asynchronously. There are four main benefits. Faster responses, because slow work like emails, SMS, PDF invoices and notifications happens in the background. Decoupling, because the producer doesn't need to know who consumes the event. Absorbing traffic spikes, because messages wait in the queue and consumers process at their own pace. And reliability, because if a consumer is down, messages are kept and retried.
>
> My real example: at Appsinvo, notifications and order receipts were sent inside the order API request. That made the API slow, and if the email or SMS provider failed, either the order request failed or the message was lost. I moved them to Kafka: the order API just publishes an `order.placed` event and returns, and consumers handle notifications with retries and a dead-letter queue."

A queue lets one service **send a message and move on**, while another service processes it later.

Benefits:
- **Faster API response** – the slow work (email, SMS, PDF invoice, notification) is done in the background. I moved notification and order-receipt events to Kafka so the order API returned faster.
- **Decoupling** – order service doesn't need to know who listens.
- **Handles traffic spikes** – messages wait in the queue; consumers process at their own speed.
- **Reliability** – if a consumer is down, messages stay and are processed later; retries are possible.

### Queue vs pub/sub

> 🎤 **Interview answer:** "In a point-to-point queue, each message is processed by exactly one consumer. It's a job queue, where multiple workers share the load, like SQS or a RabbitMQ queue. In pub/sub, each message is delivered to every subscriber, so multiple independent services can react to the same event, like Redis pub/sub, SNS, or a Kafka topic read by several consumer groups. Kafka actually gives both: within one consumer group it behaves like a queue, because each partition goes to one consumer, and across groups it behaves like pub/sub."

- **Queue (point-to-point)** – each message is processed by **one** consumer (job queue). SQS, RabbitMQ queue.
- **Pub/sub** – each message goes to **every** subscriber group. Kafka topic with multiple consumer groups, Redis pub/sub, SNS.

---

## Kafka

### Kafka core concepts

> 🎤 **Interview answer:** "Kafka is a distributed, durable, append-only commit log. Producers write messages to topics. Each topic is split into partitions, which are ordered logs and the unit of parallelism, and each message in a partition has an offset. Partitions are replicated across brokers, typically with a replication factor of 3, with one leader and followers, so data survives broker failures. Messages stay on disk for the retention period even after they're consumed, so they can be replayed.
>
> Consumers read in consumer groups: each partition is assigned to only one consumer within a group, so the maximum parallelism equals the number of partitions, and consumers commit offsets to remember where they stopped. Different consumer groups each get all the messages independently. For example, our notification service and an analytics service could both read `order.placed`. Newer Kafka uses KRaft instead of ZooKeeper for cluster metadata."

Kafka is a distributed, durable **log** of messages. Messages are stored on disk for a retention period (e.g. 7 days) even after being read.

| Term | Simple meaning |
|---|---|
| **Broker** | A Kafka server. A cluster has many brokers |
| **Topic** | A named stream of messages, e.g. `order.placed` |
| **Partition** | A topic is split into partitions for parallel processing. Each partition is an ordered log |
| **Offset** | Position number of a message inside a partition |
| **Producer** | Sends messages to a topic |
| **Consumer** | Reads messages |
| **Consumer group** | Consumers sharing the work. Each partition is read by **only one** consumer in a group |
| **Replication factor** | Copies of each partition on different brokers (e.g. 3) for safety |
| **Leader / follower** | One broker is the leader for a partition; followers copy it |
| **Commit offset** | Consumer saves "I've processed up to offset X" so it continues from there after restart |
| **ZooKeeper / KRaft** | Cluster metadata management (newer Kafka uses KRaft, no ZooKeeper) |

**Scaling rule:** max useful consumers in a group = number of partitions. 6 partitions → up to 6 consumers in parallel.
**Different consumer groups** each get **all** messages: e.g. `email-service` group and `analytics` group both read `order.placed`.

### How ordering works

> 🎤 **Interview answer:** "Kafka only guarantees order within a single partition, not across the whole topic. The producer chooses the partition by hashing the message key, so all messages with the same key go to the same partition, in order. So I use a key like `orderId` or `userId`, which means all events for one order, like placed, paid and shipped, are processed in sequence, while different orders still run in parallel across partitions. If I needed strict global order, I'd need a single partition, which kills parallelism, so I design around per-entity ordering instead."

- Order is guaranteed **only inside one partition**, not across the whole topic.
- Messages with the **same key** go to the **same partition**. So use `orderId` or `userId` as the key → all events of one order are processed in order.

### Delivery guarantees

> 🎤 **Interview answer:** "There are three levels. At-most-once: you commit the offset before processing, so messages can be lost but never duplicated. At-least-once: you process first and commit after, so nothing is lost, but if the consumer crashes after processing and before committing, the message is redelivered. That's the most common choice, and it's what I used, and it requires idempotent consumers. Exactly-once: Kafka supports it with idempotent producers and transactions, but only within Kafka. As soon as the side effect is external, like sending an email or writing to a database, you still need idempotent handling on your side. On the producer side I use `acks=all`, the idempotent producer and retries, so a message isn't lost before it reaches Kafka."

| Type | Meaning | How |
|---|---|---|
| At-most-once | May lose, never duplicate | Commit offset before processing |
| **At-least-once** (most common) | Never lose, may duplicate | Process first, then commit offset → **consumer must be idempotent** |
| Exactly-once | No loss, no duplicate | Kafka idempotent producer + transactions (works within Kafka); for external side effects (email, DB) you still need idempotent handling |

Producer safety: `acks=all` (wait for all in-sync replicas), idempotent producer enabled, retries.

### Retries and dead-letter queue (DLQ)

> 🎤 **Interview answer:** "The problem is that some messages fail: the email provider is down, or the data is bad. If the consumer keeps retrying forever, it blocks the whole partition and every message behind it waits. If it just skips the message, the message is lost. So here's what I implemented at Appsinvo.
>
> On failure, the consumer retries a limited number of times with backoff, about three attempts. I did that by republishing to a retry topic with an attempt-count header, so the main partition isn't blocked. If it still fails after the maximum attempts, the message goes to a dead-letter topic like `notifications.dlq`, together with the error message, attempt count, original topic and offset, and the timestamp. Then the offset is committed and the main flow continues. We monitor the DLQ and alert when it's not empty, investigate the cause, fix it, and replay the messages back to the main topic.
>
> I also separate error types: transient errors like timeouts or 5xx get retried, while permanent ones like validation errors go straight to the DLQ, because retrying a poison message never helps."

**Problem:** a message fails processing (bad data, email provider down). If we keep retrying forever, the partition is stuck and later messages wait. If we skip it, we lose it.

**Solution I used:**
1. Try to process.
2. On failure, **retry a few times** with backoff (e.g. 3 times: 1 s, 5 s, 30 s). Can be in-process or by sending to a `topic.retry` topic.
3. After max retries, send the message to a **dead-letter topic** (`notifications.dlq`) with error details (error message, attempt count, original topic/partition/offset, timestamp).
4. Commit the offset so the main flow continues.
5. **Monitor the DLQ** (alert when count > 0), check the cause, fix, and **replay** messages back to the main topic.

```js
await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const attempts = Number(message.headers?.attempts?.toString() || 0);
    try {
      await handleNotification(JSON.parse(message.value.toString()));
    } catch (err) {
      if (attempts < 3) {
        await producer.send({
          topic: "notifications.retry",
          messages: [{ key: message.key, value: message.value, headers: { attempts: String(attempts + 1) } }],
        });
      } else {
        await producer.send({
          topic: "notifications.dlq",
          messages: [{ key: message.key, value: message.value,
            headers: { error: err.message, originalTopic: topic, failedAt: new Date().toISOString() } }],
        });
      }
    }
  },
});
```
**Poison message** = a message that will always fail (bad format). DLQ stops it from blocking the queue.
**Transient vs permanent errors:** retry on timeouts / 5xx; send directly to DLQ on validation errors (retrying won't help).

### Idempotent consumers

> 🎤 **Interview answer:** "Since at-least-once delivery means the same message can arrive twice, for example after a consumer crash or a rebalance before the offset commit, every consumer must be safe to run twice. The way I do it: every event carries a unique `eventId`. Before processing, the consumer checks whether that id has already been processed, using a processed-events table with a unique constraint or a Redis set with a TTL, and records it after success. Where possible I rely on natural uniqueness, like a unique `orderId` in the receipts table, and use upserts instead of inserts. That's how we made sure a customer never got two order emails for the same order."

Because of at-least-once delivery, the same message can come twice (consumer crashed after processing but before committing).
Fix:
- Every event has a unique `eventId`.
- Before processing, check a `processed_events` table / Redis set; or use a unique DB constraint (e.g. unique `orderId` in receipts table).
- Use upserts instead of inserts.

### Kafka with Node.js (kafkajs)

> 🎤 **Interview answer:** "In Node I used kafkajs. I create a Kafka client with a clientId and the broker list. The producer is created with `idempotent: true`, and I send messages with a key, like the order id for ordering, and a JSON value containing an eventId. The consumer joins a groupId, like 'notification-service', subscribes to the topic and processes messages in `eachMessage`, with try/catch that leads into the retry and DLQ logic. I also handle graceful shutdown: on SIGTERM, I disconnect the consumer so offsets are committed and the group rebalances cleanly."

```js
const { Kafka } = require("kafkajs");
const kafka = new Kafka({ clientId: "order-service", brokers: ["broker1:9092"] });

// Producer
const producer = kafka.producer({ idempotent: true });
await producer.connect();
await producer.send({
  topic: "order.placed",
  messages: [{ key: String(order.id), value: JSON.stringify({ eventId, orderId: order.id, userId }) }],
});

// Consumer
const consumer = kafka.consumer({ groupId: "notification-service" });
await consumer.connect();
await consumer.subscribe({ topic: "order.placed", fromBeginning: false });
await consumer.run({ eachMessage: async ({ message }) => { /* process */ } });
```

**Consumer lag** = how far behind the consumer is (latest offset – committed offset). Monitor it. High lag → add consumers (up to partition count) or make processing faster.

> 🎤 **Interview answer:** "Consumer lag is the difference between the latest offset in a partition and the offset the consumer group has committed, basically how far behind the consumer is. It's the key health metric for consumers. If lag keeps growing, consumers can't keep up, and I either scale out consumers, up to the number of partitions, or make processing faster, with batching or by removing slow calls. I set alerts on lag so we find out before users notice delayed notifications."

**Rebalancing** = when a consumer joins/leaves, partitions are re-assigned. During this, consumption pauses briefly.

> 🎤 **Interview answer:** "A rebalance happens when a consumer joins or leaves the group, or is considered dead because it missed heartbeats or took too long to process. Kafka then reassigns partitions among the remaining consumers, and consumption pauses briefly. Frequent rebalances hurt throughput and can cause duplicate processing, so I keep message processing well within the session and poll timeouts, shut consumers down gracefully, and make handlers idempotent."

### Kafka vs RabbitMQ vs SQS

> 🎤 **Interview answer:** "Kafka is a distributed log: messages are kept for the retention period and can be replayed, it handles very high throughput, it supports many independent consumer groups, and it orders per partition. It's best for event streaming, multiple consumers of the same events, and analytics, but it takes the most effort to operate, unless you use MSK or Confluent. RabbitMQ is a traditional broker: messages are deleted after acknowledgement, it has very flexible routing with direct, topic and fanout exchanges, and built-in dead-letter exchanges. It's great for task queues and complex routing. SQS is fully managed by AWS with zero ops: a simple queue with visibility timeouts and built-in DLQ redrive, and FIFO queues when you need ordering. It's ideal for simple background jobs on AWS.
>
> At Appsinvo we chose Kafka because several consumers needed the same order events and replay was valuable. For a simple job queue on AWS, I'd pick SQS."

| | Kafka | RabbitMQ | AWS SQS |
|---|---|---|---|
| Model | Distributed log (pub/sub with replay) | Message broker with queues and exchanges | Managed simple queue |
| Message after read | Kept until retention ends (can replay) | Deleted after ack | Deleted after consumer deletes it |
| Throughput | Very high (millions/sec) | High | High, managed |
| Ordering | Per partition | Per queue | Standard: no; FIFO queues: yes |
| Routing | By topic/partition key | Flexible (direct, topic, fanout exchanges) | None (use SNS for fanout) |
| DLQ | Build yourself (DLQ topic) | Built-in (dead-letter exchange) | Built-in (redrive policy, `maxReceiveCount`) |
| Ops effort | High (or MSK/Confluent) | Medium | None |
| Good for | Event streaming, many consumers, analytics, event sourcing | Task queues, complex routing, RPC | Simple background jobs on AWS |

**SQS points:** visibility timeout (message hidden while a consumer works; if not deleted in time it appears again), long polling, FIFO with deduplication id.

> 🎤 **Interview answer:** "The key SQS concepts are the following. When a consumer receives a message, it becomes invisible for the visibility timeout. If the consumer doesn't delete it within that time, for example because it crashed, the message becomes visible again and is retried, so processing must be idempotent and the timeout must be longer than the processing time. Long polling reduces empty responses and cost. A redrive policy with `maxReceiveCount` moves repeatedly failing messages to a DLQ automatically. FIFO queues give ordering per message group and deduplication with a deduplication id."

---

## Redis

### Redis data types and uses

> 🎤 **Interview answer:** "Redis has several data structures, and choosing the right one is the main design decision. Strings are for cache values, counters with INCR, rate limits and OTPs with TTL. Hashes are for objects with fields, like a session. Lists are for simple queues or the latest N items. Sets are for unique things like online users. Sorted sets are for leaderboards, sliding-window rate limiting and delayed jobs, sorted by timestamp. Streams are for a persistent log with consumer groups and acknowledgements. Pub/sub is for real-time broadcast. And there are geo commands and HyperLogLog for approximate unique counts. In my projects I've used strings for cache, OTP, rate limiting and the leader lock, and pub/sub for inter-service events and price broadcasts."

| Type | Commands | Use case |
|---|---|---|
| String | `SET GET INCR EXPIRE` | Cache, counters, rate limit, OTP |
| Hash | `HSET HGET HGETALL` | User session/object fields |
| List | `LPUSH RPOP LRANGE` | Simple queue, latest N items |
| Set | `SADD SISMEMBER SMEMBERS` | Unique visitors, online users, tags |
| Sorted set | `ZADD ZRANGE ZREVRANGE ZINCRBY` | Leaderboards, sliding-window rate limit, delayed jobs |
| Stream | `XADD XREADGROUP XACK` | Persistent message queue with consumer groups |
| Pub/Sub | `PUBLISH SUBSCRIBE` | Real-time broadcast |
| Geo | `GEOADD GEOSEARCH` | Nearby search |
| Bitmap / HyperLogLog | `SETBIT`, `PFADD` | Daily active flags, approximate unique counts |

**Q. Why is Redis fast?**

> 🎤 **Interview answer:** "There are four reasons. All data lives in RAM, so there's no disk I/O on reads and writes. It uses efficient, purpose-built data structures. Commands execute on a single thread, so there's no locking or context-switching overhead. And it uses I/O multiplexing with epoll to handle thousands of client connections efficiently. Most operations are O(1) or O(log n), so typical latency is sub-millisecond."

Data in RAM, simple data structures, single-threaded command execution (no lock overhead), efficient I/O multiplexing.

**Q. Redis is single-threaded – is that a problem?**

> 🎤 **Interview answer:** "Command execution is single-threaded, which is actually an advantage: every individual command is atomic, which is why INCR for rate limiting or SET NX for locks is safe without extra locking. It's fast enough for most workloads, since the bottleneck is usually network or memory, not CPU. Newer versions also use I/O threads for networking. The real risk is slow commands: `KEYS *`, fetching huge sets or long Lua scripts block every other client. So I use SCAN, keep values small and scripts short, and scale with Redis Cluster if needed."

Commands run one at a time, so each command is atomic. It's fast enough for most cases. Slow commands (`KEYS *`, huge `SMEMBERS`, big Lua scripts) block everyone – avoid them.

### Redis pub/sub

> 🎤 **Interview answer:** "Redis pub/sub lets publishers send messages to a channel, and every client currently subscribed to that channel receives them instantly. A subscribing connection is dedicated, so I use a separate, duplicated connection for subscribing. The critical limitation is that it's fire and forget: messages aren't stored and there's no acknowledgement, so if a subscriber is offline or reconnecting, it simply misses them.
>
> That makes it great for real-time data where the latest value replaces the old one, like crypto price updates, typing indicators, cache invalidation broadcasts, or the Socket.io Redis adapter. At KindSwap we also used it between the Referral and Staking services. For events that must not be lost, the right tools are Redis Streams with consumer groups, Kafka or RabbitMQ, or a DB-backed outbox with a reconciliation job, and that's what I'd recommend for critical reward events."

```js
const sub = redis.duplicate();             // subscriber needs its own connection
await sub.subscribe("price-updates");
sub.on("message", (channel, msg) => io.emit("price", JSON.parse(msg)));

await redis.publish("price-updates", JSON.stringify({ symbol: "SOL", price: 182.4 }));
```
**Important limitations:**
- **Fire and forget** – no storage. If a subscriber is offline, it misses the message.
- No acknowledgement, no retry.
- Good for: real-time updates where the next message replaces the old one (prices, typing indicators, cache invalidation broadcasts, Socket.io scaling).
- If you need guaranteed delivery → **Redis Streams**, Kafka, or RabbitMQ.

### Leader election with Redis

> 🎤 **Interview answer:** "This is one of my favourite problems I solved. At KindSwap, the price service ran on 4 to 5 instances for availability, and every instance fetched crypto prices and published updates. So users received duplicate price updates, and we were calling the price provider five times more than needed. The goal was that exactly one instance publishes, with automatic failover.
>
> I implemented leader election with Redis. Every instance has a unique id. Each one tries `SET price:leader <myId> NX PX 10000`. NX means only one instance can create the key, so it becomes the leader, and the 10-second expiry means that if the leader crashes, the lock frees itself. The leader renews the TTL every 3 seconds using a small Lua script that extends the expiry only if the value is still its own id, so the check and the renewal are atomic and it can never extend someone else's lock. Followers keep retrying the SET NX every few seconds, so when the leader dies, one of them takes over within about 10 seconds. If a renewal fails, the old leader stops publishing immediately. On graceful shutdown it deletes the key, only if it owns it, so takeover is instant. We logged leadership changes and publish counts to CloudWatch, with alarms if there was no leader. The result was zero duplicate updates and automatic failover."

**My problem (KindSwap):** the price service ran on 4–5 instances. Each one fetched crypto prices and published updates → users got **duplicate** price updates and we called the price provider 5 times more than needed.
**Goal:** only **one** instance (the leader) publishes; if it dies, another takes over automatically.

**How it works:**
1. Every instance has a unique id (hostname + pid / uuid).
2. Each instance tries to take a lock key:
   ```
   SET price:leader <instanceId> NX PX 10000
   ```
   - `NX` – set only if the key doesn't exist → only one instance wins.
   - `PX 10000` – key expires in 10 s, so if the leader crashes, the lock frees itself.
3. The winner is the **leader** and starts publishing prices.
4. The leader **renews** the lock every ~3 s (heartbeat), but only if it is still the owner – done atomically with a Lua script:
   ```lua
   if redis.call("GET", KEYS[1]) == ARGV[1] then
     return redis.call("PEXPIRE", KEYS[1], ARGV[2])
   else
     return 0
   end
   ```
5. Followers retry `SET NX` every few seconds. If the leader dies, its key expires in ≤10 s and one follower becomes leader.
6. If renewal fails (returns 0), the old leader **stops publishing immediately** (it lost leadership).
7. On graceful shutdown, the leader deletes the key (only if it owns it) so takeover is instant.
8. Monitored with **CloudWatch** – logs/metrics for "became leader", "lost leadership", publish counts.

```ts
async function tryBecomeLeader() {
  const ok = await redis.set(KEY, INSTANCE_ID, "PX", TTL_MS, "NX");
  if (ok === "OK") isLeader = true;
}
async function renew() {
  const res = await redis.eval(RENEW_LUA, 1, KEY, INSTANCE_ID, TTL_MS);
  if (res === 0) isLeader = false;   // lost it
}
setInterval(async () => {
  if (isLeader) await renew();
  else await tryBecomeLeader();
}, 3000);
```

**Follow-up questions you may get:**

> 🎤 **Interview answer:** "For the common follow-ups, here's how I answer. If the leader pauses, for example during a long GC, its lock can expire while it still thinks it's the leader, so for a moment two instances could publish. For price updates that's low risk, and I keep the renewal interval well below the TTL. For critical work I'd add fencing tokens, an increasing number per leadership term that consumers check. If Redis itself goes down, no one can become leader, so prices pause instead of being duplicated, which is the safe failure. That's why we use managed Redis with replication and Multi-AZ failover. Redlock across independent nodes exists, but it's debated, and for our case it wasn't needed. Why not just run one instance? Because that's a single point of failure, and the other instances also serve API and socket traffic. Alternatives are etcd, ZooKeeper or Consul, Kubernetes Leases, or Postgres advisory locks."

- *What if the leader pauses (long GC / network issue) and its lock expires, then it wakes up and publishes?* → For a short time there could be two leaders. For price updates this is low-risk. For critical work use a **fencing token** (an increasing number stored with each leadership; the receiver ignores older tokens) and keep renewal interval well below TTL.
- *What if Redis goes down?* → No one can be leader → no price updates (fail safe instead of duplicates). Use Redis with replication/Sentinel/ElastiCache Multi-AZ. Redlock (multiple independent Redis nodes) is an option, but has known debates; for our case a single managed Redis was enough.
- *Why not just run one instance?* → Single point of failure; other instances also serve API/socket traffic.
- *Alternatives?* → ZooKeeper/etcd/Consul leader election, Kubernetes Lease objects, DB advisory locks (`pg_try_advisory_lock`), or a Kafka consumer group with a single-partition topic.

### Distributed lock

> 🎤 **Interview answer:** "A distributed lock makes sure only one process across all instances does something at a time. It's the same Redis pattern: `SET lock:key <uniqueId> NX PX <ttl>`. If it returns OK, I hold the lock. I use it so a cron job runs on only one instance, so two parallel requests can't process the same payment or redeem the same coupon, and to protect against cache stampedes, where only one request rebuilds a hot key. Two rules: always set a TTL, so a crashed process doesn't hold the lock forever, and release it safely with a Lua script that deletes the key only if the value is still my id, so I never delete a lock someone else acquired after mine expired."

Same `SET key value NX PX` idea, used to make sure only one process does something at a time:
- Cron job runs on all instances → lock so only one runs it.
- Prevent two parallel requests from processing the same payment / same coupon.
- Rebuild a cache key (stampede protection).

Release safely – delete only if you own it (Lua: `if GET == myId then DEL`). Always set a TTL so a crashed process doesn't hold the lock forever.

### Other Redis questions

> 🎤 **Interview answer:** "A few other points I'm ready for. Persistence: RDB snapshots are compact and restart fast but can lose the last few minutes, while AOF logs every write and is safer, and you can combine both. High availability: replicas plus Sentinel for automatic failover, or Redis Cluster, which shards data across 16,384 hash slots for horizontal scale. Compared to Memcached, Redis has rich data types, persistence, replication and pub/sub, while Memcached is just a multi-threaded key-value cache. Transactions use MULTI and EXEC, with WATCH for optimistic locking, and Lua scripts are atomic. Pipelining batches commands to cut round trips. And I watch for big keys and hot keys, because they cause latency spikes."

- **Persistence:** RDB (snapshots every N minutes – fast restart, may lose last minutes) vs AOF (logs every write – safer, bigger files). Can use both.
- **Replication / Sentinel / Cluster:** replicas for read scale + failover; Sentinel monitors and promotes a new master; Cluster shards data across nodes (16384 hash slots).
- **Redis vs Memcached:** Redis has many data types, persistence, pub/sub, replication. Memcached is only simple key-value cache, multi-threaded.
- **Transactions:** `MULTI/EXEC` runs commands together; `WATCH` for optimistic locking. Lua scripts are atomic too.
- **Pipelining:** send many commands without waiting for each reply – fewer round trips.
- **TTL** – `EXPIRE`, `TTL key` (-1 = no expiry, -2 = key doesn't exist).
- **Big keys / hot keys** – very large values or one key with huge traffic cause slowness; split or replicate.

---

## Outbox pattern

> 🎤 **Interview answer:** "The outbox pattern solves the dual-write problem. If I save an order in the database and then publish `order.placed` to Kafka as two separate steps, a crash in between leaves them inconsistent: the order exists but no event was sent, or the event was sent but the DB transaction rolled back.
>
> With the outbox pattern, in the same database transaction as the order, I also insert a row into an `outbox` table with the event type and payload. Since it's one transaction, both are saved or neither is. A separate relay worker polls pending outbox rows, publishes them to Kafka and marks them as sent. Change data capture with Debezium is an alternative to polling. Because a row might be published twice if the worker crashes after publishing, consumers stay idempotent. It guarantees that if the order is saved, the event will eventually be published."

**Problem:** You save an order in DB and then publish `order.placed` to Kafka. If the app crashes between the two, DB has the order but no event is sent (or the event is sent but DB transaction rolled back).

**Solution:**
1. In the **same DB transaction**, insert the order **and** a row in an `outbox` table (`eventType, payload, status = pending`).
2. A separate worker reads pending outbox rows, publishes them to Kafka, and marks them `sent`.
3. Consumers are idempotent because a row might be published twice.

This guarantees "if the order is saved, the event will be sent".

---

[Home](README.md) | [← 08 NestJS](08-nestjs.md) | Next: [10 System Design →](10-system-design.md)
