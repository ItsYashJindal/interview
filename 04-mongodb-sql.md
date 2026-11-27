[Home](README.md) | [← 03 Node.js & Express](03-nodejs-express.md) | Next: [05 React →](05-react.md)

# 04 – MongoDB & SQL (PostgreSQL / MySQL)

## Contents
- **MongoDB**
  1. [CRUD operations](#1-crud-operations)
  2. [Indexing](#2-indexing)
  3. [Aggregation pipeline](#3-aggregation-pipeline)
  4. [Population vs lookup](#4-population-vs-lookup)
  5. [MongoDB schema design](#5-mongodb-schema-design)
- **SQL**
  6. [SQL joins](#6-sql-joins)
  7. [Primary and foreign keys](#7-primary-and-foreign-keys)
  8. [Transactions](#8-transactions)
  9. [Query optimisation](#9-query-optimisation)
- [Extra MongoDB questions (added)](#extra-mongodb-questions)
- [Extra SQL questions + must-practise queries (added)](#extra-sql-questions)
- [ORMs – Prisma, TypeORM, Mongoose (added)](#orms--prisma-typeorm-mongoose)

---

## MongoDB

### 1. CRUD operations

> 🎤 **Interview answer:** "CRUD means Create, Read, Update, Delete.
> - **Create** – `insertOne` / `insertMany`.
> - **Read** – `find` (many) or `findOne`, with a filter, the fields I want, and sort / skip / limit.
> - **Update** – `updateOne` / `updateMany` with operators like `$set` (change a field), `$inc` (add to a number), `$push` (add to an array). With `upsert: true`, it creates the document if none is found.
> - **Delete** – `deleteOne` / `deleteMany`.
>
> In real projects I use update operators instead of replacing the whole document. For example, to reduce stock safely I use `$inc: { stock: -qty }` with the filter `stock >= qty`, so stock can never go below zero."

```js
// Create
db.users.insertOne({ name: "Yash", age: 27 });
db.users.insertMany([{ name: "A" }, { name: "B" }]);

// Read
db.users.find({ age: { $gte: 18 } }, { name: 1, email: 1 })  // filter, fields to return
        .sort({ createdAt: -1 }).skip(0).limit(10);
db.users.findOne({ email: "a@x.com" });

// Update
db.users.updateOne({ _id: id }, { $set: { name: "New" }, $inc: { loginCount: 1 } });
db.users.updateMany({ active: false }, { $set: { archived: true } });
db.users.updateOne({ email }, { $set: { name } }, { upsert: true }); // create if not found

// Delete
db.users.deleteOne({ _id: id });
db.users.deleteMany({ archived: true });
```

**Common operators:**
- Compare: `$eq $ne $gt $gte $lt $lte $in $nin` (equal, not equal, greater than, ...)
- Logic: `$and $or $not`
- Field exists: `$exists`
- Arrays: `$all $elemMatch $size`
- Update: `$set $unset $inc $push $pull $addToSet`

**Q. updateOne vs findOneAndUpdate?**

> 🎤 **Interview answer:** "`updateOne` only tells me how many documents were matched and changed. `findOneAndUpdate` gives me the **document itself**, before or after the update (`{ new: true }` in Mongoose gives the updated one). I use `updateOne` when I just need to change something, and `findOneAndUpdate` when I need the updated document back."

**Q. Soft delete vs hard delete?**

> 🎤 **Interview answer:** "**Hard delete** removes the record forever. **Soft delete** just marks it, like `isDeleted: true` or `deletedAt: <date>`. The data can be restored later, and old orders that point to a deleted product don't break. The downside: every query must skip deleted records. I handle that in one common place, like a Mongoose `pre('find')` hook, and I add an index on that field."

---

### 2. Indexing

**Q. What is an index?**

> 🎤 **Interview answer:** "An index is like the index at the back of a book. Instead of reading every page to find a topic, you look it up in the index and jump to the right page.
>
> Without an index, MongoDB reads **every document** to find matches. This is called a COLLSCAN (collection scan), and it's slow. With an index, it jumps straight to the matching documents. This is called IXSCAN (index scan).
>
> Adding the right indexes was the main reason I could bring a user-listing API down from 4.5 seconds to 550 ms."

**Types of indexes:**
| Type | Example | Use |
|---|---|---|
| Single field | `{ email: 1 }` | Search by one field |
| Compound | `{ status: 1, createdAt: -1 }` | Filter + sort together |
| Unique | `{ email: 1 }, { unique: true }` | No duplicate emails |
| Multikey | index on an array field | Search inside arrays (tags) |
| Text | `{ title: "text" }` | Basic keyword search |
| TTL | `{ createdAt: 1 }, { expireAfterSeconds: 300 }` | Auto-delete old documents (OTPs, sessions) |
| 2dsphere | `{ location: "2dsphere" }` | Location search ("near me") – used in matrimony project |
| Partial | index only where `isDeleted: false` | Smaller index |

**Q. ESR rule for compound indexes**

> 🎤 **Interview answer:** "ESR tells the best order of fields in a compound index: **E**quality → **S**ort → **R**ange.
> - **Equality** – fields matched exactly, like `status: 'active'`.
> - **Sort** – the field we sort on, like `createdAt`.
> - **Range** – fields with `>`, `<`, like `age >= 25`.
>
> With this order, MongoDB finds the matches and reads them already sorted. If the order is wrong, MongoDB has to sort in memory, which is slow."

```js
// Query
find({ status: "active", age: { $gte: 25 } }).sort({ createdAt: -1 })

// Best index: Equality, Sort, Range
{ status: 1, createdAt: -1, age: 1 }
```

**Q. Prefix rule**

> 🎤 **Interview answer:** "A compound index on `a, b, c` can be used for queries on `a`, on `a + b`, and on `a + b + c`. But **not** for a query on only `b` or only `c`. It's like a phone book sorted by surname, then first name: you can find all 'Sharma's easily, but you can't find all 'Rahul's quickly."

**Q. How do you check if a query uses an index?**

> 🎤 **Interview answer:** "I run the query with `.explain('executionStats')` and check three things:
> 1. **stage** – IXSCAN is good, COLLSCAN on a big collection is bad.
> 2. **totalDocsExamined vs nReturned** – they should be close. If MongoDB checks 100,000 documents to return 10, the index is wrong or missing.
> 3. **executionTimeMillis** – how long it took.
>
> In SQL, the same tool is `EXPLAIN ANALYZE`."

```js
db.users.find({ email: "a@x.com" }).explain("executionStats");
```

**Q. Downsides of indexes?**

> 🎤 **Interview answer:** "Indexes make reads faster but writes slower, because every insert or update must also update every index. They also use RAM and disk. So I only add indexes for queries we really run, and I remove indexes that are never used."

**Covered query:** when all the fields in the filter and the result are inside the index, MongoDB answers from the index alone, without reading the documents. This is the fastest.

---

### 3. Aggregation pipeline

**Q. What is it?**

> 🎤 **Interview answer:** "The aggregation pipeline is MongoDB's way to do reports and calculations. Documents pass through a list of steps (stages), one after another, like water through a series of filters. Each step changes the data:
> - `$match` – filter (like WHERE),
> - `$group` – group and calculate totals or averages,
> - `$project` – choose which fields to show,
> - `$sort`, `$limit` – order and limit,
> - `$lookup` – join with another collection,
> - `$unwind` – turn an array into separate documents.
>
> I've used it for sales reports, top-selling products and dashboard numbers. Tip: I always put `$match` first, so it can use indexes and the next steps have less data to work on."

**Common stages:**
| Stage | Does |
|---|---|
| `$match` | Filter (like WHERE) – put it **first** so indexes are used |
| `$project` / `$addFields` | Choose fields / add calculated fields |
| `$group` | Group and calculate (`$sum`, `$avg`, `$max`) |
| `$sort`, `$skip`, `$limit` | Order and paginate |
| `$lookup` | Join with another collection |
| `$unwind` | Make one document per array item |
| `$facet` | Run several pipelines at once (e.g. data + total count) |
| `$count` | Count documents |

**Example – monthly sales report for delivered orders:**
```js
db.orders.aggregate([
  // 1. only delivered orders from this year
  { $match: { status: "delivered", createdAt: { $gte: ISODate("2026-01-01") } } },
  // 2. group by month and calculate
  { $group: {
      _id: { month: { $month: "$createdAt" } },
      totalSales: { $sum: "$amount" },
      orders: { $sum: 1 },
      avgOrder: { $avg: "$amount" },
  }},
  // 3. sort by month
  { $sort: { "_id.month": 1 } },
]);
```

**Example – top 5 selling products:**
```js
db.orders.aggregate([
  { $unwind: "$items" },                                                   // one doc per item
  { $group: { _id: "$items.productId", qty: { $sum: "$items.qty" } } },    // total qty per product
  { $sort: { qty: -1 } },                                                  // most sold first
  { $limit: 5 },                                                           // top 5
  { $lookup: { from: "products", localField: "_id", foreignField: "_id", as: "product" } }, // get product
  { $unwind: "$product" },
  { $project: { _id: 0, name: "$product.name", qty: 1 } },                 // show name + qty
]);
```

**Example – paginated list + total count in one query:**
```js
db.users.aggregate([
  { $match: { active: true } },
  { $facet: {
      data: [{ $sort: { createdAt: -1 } }, { $skip: 20 }, { $limit: 10 }],
      total: [{ $count: "count" }],
  }},
]);
```

**Tips:** `$match` and `$sort` early (to use indexes). `$project` early to remove unused fields. Each stage can use max 100 MB memory; for bigger data use `allowDiskUse: true`.

---

### 4. Population vs lookup

> 🎤 **Interview answer:** "Both bring in data from another collection, like getting the user details inside an order. The difference is **where** the join happens.
> - **`populate`** (Mongoose) – happens in Node. Mongoose first gets the orders, then runs a **second query** to get the users, and merges them. Very easy to use, but it's an extra query, and I can't filter or group on the user fields in the database.
> - **`$lookup`** (MongoDB) – happens inside the database, in one query. I can filter, sort and group on the joined data.
>
> I use `populate` for simple cases like 'order with user name', and `$lookup` for reports. If I need a join all the time, I rethink the schema."

| | Mongoose `populate()` | MongoDB `$lookup` |
|---|---|---|
| Where it runs | In Node (extra queries) | Inside MongoDB (one query) |
| Ease | Very easy, uses `ref` in schema | More code |
| Filter/sort/group on joined data | Limited | Full power |
| Best for | Simple "get order with user name" | Reports, filtering on joined fields |

```js
// populate
const order = await Order.findById(id).populate("user", "name email");

// $lookup
await Order.aggregate([
  { $match: { _id: id } },
  { $lookup: { from: "users", localField: "user", foreignField: "_id", as: "user" } },
  { $unwind: "$user" },
]);
```

---

### 5. MongoDB schema design

> 🎤 **Interview answer:** "My main rule: design the schema based on **how the app reads the data**.
>
> I **embed** (store inside the same document) when the data is always read together and is small, like a user's 2–3 addresses.
>
> I **reference** (store only the id, keep data in another collection) when the data can grow a lot or is used on its own, like orders, messages or payments.
>
> I remember that a document can be max 16 MB, so I never let an array grow forever.
>
> In my matrimony project, preferences and photos are embedded in the profile, but interests, messages and payments are separate collections, because they keep growing."

**Embed vs reference:**
| Embed (store inside) | Reference (store id) |
|---|---|
| Data is read together (user + address) | Data is used separately |
| One-to-few (a user has 3 addresses) | One-to-many (a user has 10,000 orders) |
| Small, doesn't grow much | Big or keeps growing |
| One read, simple update | Needs a join (populate / $lookup) |

**Limits to remember:** a document can be max **16 MB**. Don't keep all chat messages inside one conversation document – store messages in a separate collection.

**Patterns (simple):**
- **Subset pattern** – keep the last 5 reviews inside the product, the rest in a reviews collection.
- **Extended reference** – copy a few fields you always need (like `userName` inside the order) to avoid a join.
- **Computed pattern** – save counters like `totalOrders` or `rating`, instead of calculating them on every read.
- **Bucket pattern** – group time-based data (e.g. one document per sensor per hour).

**Example – matrimony profile (my project):**
```js
const profileSchema = new Schema({
  userId: { type: ObjectId, ref: "User", unique: true },
  name: String,
  gender: { type: String, enum: ["male", "female"] },
  dob: Date,
  religion: String, caste: String, education: String, income: Number,
  location: { type: { type: String, default: "Point" }, coordinates: [Number] }, // [lng, lat]
  preferences: { ageMin: Number, ageMax: Number, religion: [String] },          // embedded
  photos: [String],                                                               // few → embed
  isPremium: Boolean,
}, { timestamps: true });

profileSchema.index({ location: "2dsphere" });
profileSchema.index({ gender: 1, religion: 1, dob: 1 });
```
Messages, interests and payments are separate collections because they grow a lot.

---

## SQL

### 6. SQL joins

> 🎤 **Interview answer:** "A join combines rows from two tables using a common column.
> - **INNER JOIN** – only rows that match in both tables. Example: users who have orders.
> - **LEFT JOIN** – all rows from the left table, plus matching rows from the right. If no match, the right side is NULL. Example: all users with their order count, even 0.
> - **RIGHT JOIN** – the opposite of LEFT JOIN.
> - **FULL OUTER JOIN** – all rows from both tables.
> - **SELF JOIN** – a table joined with itself, like employee → manager.
>
> A useful trick: LEFT JOIN + `WHERE o.id IS NULL` finds users who never ordered."

Tables: `users(id, name)`, `orders(id, user_id, amount)`

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows that match in both tables |
| `LEFT JOIN` | All rows from left + matching from right (NULL if no match) |
| `RIGHT JOIN` | All rows from right + matching from left |
| `FULL OUTER JOIN` | All rows from both (not in MySQL) |
| `CROSS JOIN` | Every combination (rows × rows) |
| `SELF JOIN` | Table joined with itself (employee → manager) |

```sql
-- Users with their orders (only users who ordered)
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- All users and their order count (including 0)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- Users who never ordered
SELECT u.* FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- Employee with manager name (self join)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

### 7. Primary and foreign keys

> 🎤 **Interview answer:**
> - "**Primary key** – a column that uniquely identifies each row, like `id`. It must be unique and not empty. One per table. It is indexed automatically.
> - **Foreign key** – a column that points to the primary key of another table, like `orders.user_id → users.id`. The database makes sure it's valid, so you can't create an order for a user that doesn't exist.
>
> With `ON DELETE` we decide what happens to child rows when the parent is deleted: delete them too (CASCADE), set to NULL, or block the delete (RESTRICT).
>
> One practical point: Postgres does **not** index foreign key columns by itself, so I add those indexes myself."

- **Unique key** – values must be unique, can be many per table.
- **Composite key** – a primary key made of 2+ columns (e.g. `order_id + product_id`).

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  amount NUMERIC(10,2) NOT NULL CHECK (amount >= 0),
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_orders_user_id ON orders(user_id);  -- not automatic in Postgres
```

**Q. Natural key vs surrogate key?**

> 🎤 **Interview answer:** "A **natural key** is real data used as the id, like an email. A **surrogate key** is a generated id with no meaning, like `1, 2, 3` or a UUID. I prefer surrogate keys, because real data can change. If a user changes their email and email is the primary key, every table that points to it breaks. I still keep a unique rule on email."

**Q. UUID vs auto-increment?**

> 🎤 **Interview answer:**
> - "**Auto-increment** (1, 2, 3...) – small and fast, but easy to guess. If `/orders/105` exists, someone can try `/orders/106`.
> - **UUID** (long random string) – unique everywhere and not guessable, and the app can create it before saving. But it's bigger, and random values make inserts a bit slower. UUIDv7 is time-ordered and fixes this.
>
> A common approach: a number id inside the DB, and a UUID for public URLs."

---

### 8. Transactions

**Q. What is a transaction?**

> 🎤 **Interview answer:** "A transaction groups many database operations into **one unit**: either all of them succeed, or none of them happen. If any step fails, everything is undone (rolled back).
>
> Classic example: money transfer. Taking money from A and adding it to B must both happen, or neither.
>
> In the Air India project, I used a Prisma transaction to create a GRN (goods received note) and update the received quantity on the purchase order together. So we never have a GRN without the PO being updated."

**ACID:**

> 🎤 **Interview answer:** "ACID is four promises a transaction gives:
> - **Atomicity** – all or nothing.
> - **Consistency** – data always follows the rules (constraints, foreign keys).
> - **Isolation** – two transactions running at the same time don't see each other's half-done work.
> - **Durability** – once saved (committed), the data survives even a crash."

```sql
BEGIN;
UPDATE wallets SET balance = balance - 500 WHERE user_id = 1 AND balance >= 500;
UPDATE wallets SET balance = balance + 500 WHERE user_id = 2;
INSERT INTO transactions(from_id, to_id, amount) VALUES (1, 2, 500);
COMMIT;   -- or ROLLBACK on error
```

**Prisma (used at Air India project):**
```ts
await prisma.$transaction(async (tx) => {
  const grn = await tx.grn.create({ data: grnData });
  await tx.purchaseOrder.update({
    where: { id: poId },
    data: { receivedQty: { increment: qty } },
  });
});
```

**MongoDB (many documents, needs a replica set):**
```js
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    await Order.create([orderData], { session });
    await Product.updateOne(
      { _id: pid, stock: { $gte: qty } },
      { $inc: { stock: -qty } },
      { session }
    );
  });
} finally {
  session.endSession();
}
```
Note: an update on **one** MongoDB document is already all-or-nothing. Use transactions only when several documents must change together.

**Isolation levels (low → high):**

> 🎤 **Interview answer:** "Isolation level decides how much one transaction can see of another transaction's changes. From weakest to strongest:
> - **Read Uncommitted** – can see other transactions' unsaved changes (dirty read).
> - **Read Committed** (Postgres default) – sees only saved data, but if you read the same row twice, it may change in between.
> - **Repeatable Read** (MySQL default) – the same row always reads the same inside the transaction.
> - **Serializable** – acts like transactions run one by one. Safest but slowest.
>
> For stock and wallet updates, I usually keep the default level and lock the rows with `SELECT ... FOR UPDATE`, instead of raising the level for everything."

| Level | Prevents | Still allows |
|---|---|---|
| Read Uncommitted | – | Dirty reads |
| Read Committed (Postgres default) | Dirty reads | Row may change if read twice |
| Repeatable Read (MySQL default) | Dirty reads + row changes | New rows may appear (phantom) |
| Serializable | Everything | Slowest |

- **Dirty read** – reading data another transaction hasn't saved yet.
- **Non-repeatable read** – the same row read twice gives different values.
- **Phantom read** – the same query returns new rows the second time.

**Locks:**

> 🎤 **Interview answer:** "There are two ways to stop two people changing the same row at the same time:
> - **Pessimistic locking** – `SELECT ... FOR UPDATE` locks the row. Anyone else who wants to change it must wait until I'm done. Good when clashes are common, like stock or wallet.
> - **Optimistic locking** – no lock. I keep a `version` number on the row and update only if the version is still the same. If 0 rows were updated, someone else changed it first, so I retry. Good when clashes are rare.
>
> A **deadlock** is when two transactions wait for each other forever. The database kills one of them. To avoid it, I always lock rows in the same order and keep transactions short."

```sql
-- Optimistic locking
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- 0 rows updated → someone else changed it → retry
```

See also [11 Scenarios → Q7](11-scenario-questions.md#7-how-do-you-manage-database-transactions).

---

### 9. Query optimisation

> 🎤 **Interview answer:** "I always **measure first**, then fix.
> 1. **Find the slow queries** – slow query log, MongoDB profiler, or APM tools.
> 2. **Run EXPLAIN** – to see if the DB reads the whole table, how many rows it checks, and if it sorts in memory.
> 3. **Fix** – usually by:
>    - adding the right index on the columns used in WHERE, JOIN and ORDER BY,
>    - returning only the needed columns, not `SELECT *`,
>    - fixing the N+1 problem (one query per row in a loop),
>    - adding pagination.
> 4. **Cache** heavy, frequent reads in Redis.
>
> My real example: a user-listing API went from 4.5 seconds to 550 ms by adding compound indexes for the filters and sort, returning only the needed fields with `.lean()`, and doing the filtering in the DB instead of in Node."

**Step-by-step checklist:**
1. **Find slow queries** – slow query log (MySQL), `pg_stat_statements` (Postgres), MongoDB profiler, APM.
2. **Run EXPLAIN** – `EXPLAIN ANALYZE SELECT ...` (Postgres) / `.explain("executionStats")` (Mongo). Look for full table scans.
3. **Add the right index** – on columns in WHERE, JOIN, ORDER BY.
4. **Select only needed columns** – no `SELECT *`; in Mongo use projection and `.lean()`.
5. **Fix N+1 queries** – use JOIN / `include` / `$in` / populate.
6. **Paginate** – `LIMIT`; for very deep pages use cursor pagination.
7. **Don't wrap indexed columns in functions** – `WHERE YEAR(created_at) = 2026` can't use the index. Write `created_at >= '2026-01-01' AND created_at < '2027-01-01'` instead.
8. **Avoid `LIKE '%abc'`** (starting with %) – it can't use a normal index.
9. **Cache** heavy and frequent reads in Redis.
10. **Bigger fixes** – summary tables, read replicas, partitioning.

Details of the 4.5 s → 550 ms story in [13 Resume → API optimisation](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).

---

## Extra MongoDB questions

### Q. SQL vs NoSQL – when to use which?

> 🎤 **Interview answer:** "**SQL** (Postgres, MySQL) has fixed tables, strong relations with joins and foreign keys, and strong transactions. Best for data that is very connected and must be correct, like orders, payments, inventory or ERP data.
>
> **NoSQL** (MongoDB) stores flexible JSON-like documents. Easy to change the structure, easy to scale across servers. Good for profiles, catalogues and content.
>
> Both can do most jobs today, so I choose by the shape of the data. I used MongoDB for matrimony profiles and location search, and PostgreSQL with Prisma for the Air India SAP–WMS project, where the data is strictly relational."

| SQL (Postgres/MySQL) | NoSQL (MongoDB) |
|---|---|
| Fixed tables and relations | Flexible JSON-like documents |
| Strong joins and transactions | Limited joins; embed instead |
| Money, orders, inventory, ERP | Profiles, catalogues, content, logs |
| Scale mostly with a bigger server + read copies | Built-in scaling across servers (sharding) |

### Q. Replica set vs sharding

> 🎤 **Interview answer:**
> - "**Replica set** – the **same data copied** on 3 servers. One (primary) takes writes, the others keep copies. If the primary goes down, another one takes over automatically. It's for **safety and uptime**.
> - **Sharding** – the data is **split** across servers. Each server holds only part of the data, based on a 'shard key'. It's for **very large data** that doesn't fit on one server.
>
> The shard key must spread data evenly, otherwise one server gets all the load."

### Q. What is `.lean()` in Mongoose?

> 🎤 **Interview answer:** "By default, Mongoose returns heavy document objects with extra features like `.save()`. `.lean()` returns plain JavaScript objects instead, which is much faster and uses less memory. I use it on all read-only APIs, like listings. I don't use it when I need to change and save the document."

### Q. Mongoose middleware (hooks)

> 🎤 **Interview answer:** "Hooks are functions that run automatically before or after an operation. For example, `pre('save')` to hash the password before saving, or `pre('find')` to skip soft-deleted records. Be careful: `updateOne` and `findOneAndUpdate` do **not** run `save` hooks, so password hashing in `pre('save')` won't run on them."

### Q. Geo query (matrimony "profiles near me")

> 🎤 **Interview answer:** "Each profile stores its location as a point with `[longitude, latitude]`, and the collection has a `2dsphere` index. For 'matches near me', I use `$near` with the user's location and a max distance in metres, plus filters like gender and age. A common mistake is putting latitude first. For privacy, we never return the exact location, only a rounded distance."

```js
Profile.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [lng, lat] },
      $maxDistance: 50000, // metres
    },
  },
  gender: "female",
}).limit(20);
```
Needs a `2dsphere` index. Order is **[longitude, latitude]**.

### Q. What is the `_id` / ObjectId?

> 🎤 **Interview answer:** "ObjectId is MongoDB's default id. It is 12 bytes and contains the **creation time**, a random part, and a counter. So it's unique across servers, and roughly sorted by creation time. I can even get the creation date from it with `getTimestamp()`."

### Q. Capped collections

> 🎤 **Interview answer:** "A capped collection has a fixed size. When it is full, the oldest documents are automatically overwritten. Good for logs or 'recent activity' where only the latest data matters."

### Q. Write concern and read preference

> 🎤 **Interview answer:**
> - "**Write concern** – when MongoDB says 'write done'. `w: 'majority'` means most copies have saved it, which is safer. I use it for orders and payments.
> - **Read preference** – which server to read from. Default is the primary, which has the latest data. Reading from secondaries reduces load, but data may be a little old, so I use it only for reports."

### Q. Change streams

> 🎤 **Interview answer:** "Change streams let my app **listen** to inserts, updates and deletes in a collection in real time, without checking the DB again and again. Useful for sending notifications when data changes, or syncing data to a cache or search engine. Needs a replica set."

---

## Extra SQL questions

### Q. WHERE vs HAVING

> 🎤 **Interview answer:** "`WHERE` filters **rows before** grouping. `HAVING` filters **groups after** `GROUP BY`, so it can use `SUM` or `COUNT`. Example: customers who spent more than 10,000 on paid orders. `WHERE status = 'paid'` removes unpaid rows first, then `HAVING SUM(amount) > 10000` keeps only the big spenders."

```sql
SELECT user_id, SUM(amount) AS total FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(amount) > 10000;
```

### Q. DELETE vs TRUNCATE vs DROP

> 🎤 **Interview answer:**
> - "`DELETE` – removes chosen rows with WHERE. Can be undone inside a transaction. Slow on big tables.
> - `TRUNCATE` – removes **all** rows at once. Very fast. Resets the id counter.
> - `DROP` – removes the **whole table**: structure, indexes and data.
>
> In production I never run TRUNCATE or DROP without a backup."

### Q. UNION vs UNION ALL

> 🎤 **Interview answer:** "Both join the results of two queries. `UNION` removes duplicate rows, so it's slower. `UNION ALL` keeps all rows, so it's faster. I use `UNION ALL` when duplicates are not a problem."

### Q. Normalisation (simple)

> 🎤 **Interview answer:** "Normalisation means organising tables so the same data is **not stored in many places**. Then when something changes, we update it in only one place.
> - **1NF** – each cell has one value only (no lists inside a cell).
> - **2NF** – every column depends on the whole primary key.
> - **3NF** – columns depend only on the key, not on other columns. Example: don't store the customer's city in the orders table; it belongs in the customers table.
>
> **Denormalisation** is the opposite: copying some data on purpose to make reads faster, like saving the price inside the order item."

### Q. Clustered vs non-clustered index

> 🎤 **Interview answer:**
> - "**Clustered index** – the table rows are actually stored in this order. Only one per table. In MySQL, the primary key is the clustered index.
> - **Non-clustered index** – a separate list that points to the rows. A table can have many."

### Q. What are views, stored procedures, triggers?

> 🎤 **Interview answer:**
> - "**View** – a saved SELECT query that we can use like a table.
> - **Stored procedure** – a saved SQL program inside the database, called by name.
> - **Trigger** – SQL that runs automatically on INSERT, UPDATE or DELETE, like writing an audit log.
>
> I keep most business logic in the app code, because it's easier to test and track in Git."

### Q. Window functions

> 🎤 **Interview answer:** "Window functions do calculations across related rows, but **without merging them into one row** like `GROUP BY` does. Each row keeps its data and gets an extra calculated value. `PARTITION BY` makes groups and `ORDER BY` sets the order inside each group. Common uses: rank inside a group (top salary per department), running totals.
> - `ROW_NUMBER` → 1, 2, 3, 4
> - `RANK` → 1, 2, 2, 4 (skips after a tie)
> - `DENSE_RANK` → 1, 2, 2, 3 (no skip) – best for 'Nth highest salary'."

```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
```

### Q. Connection pooling

> 🎤 **Interview answer:** "Opening a new DB connection for every request is slow, and the DB allows only a limited number of connections. A **pool** keeps some connections open and reuses them. A request borrows a connection and gives it back after use. I set the pool size based on the DB limit divided by the number of app copies. In AWS Lambda, many functions can open too many connections, so we use RDS Proxy."

### Q. N+1 problem

> 🎤 **Interview answer:** "N+1 is when the code runs **one query to get a list**, and then **one more query for each item**. For example, get 100 users, then get orders for each user in a loop = 101 queries. Fine with test data, very slow with real data. The fix: get all related data in one go, with a JOIN, Prisma `include`, or one query with `WHERE user_id IN (...)`."

```js
// Bad – 1 + N queries
const users = await User.findAll();
for (const u of users) {
  u.orders = await Order.findAll({ where: { userId: u.id } });
}

// Good – 2 queries total
const users = await User.findAll();
const ids = users.map(u => u.id);
const orders = await Order.findAll({ where: { userId: ids } }); // WHERE user_id IN (...)
```

### Q. Postgres JSONB

> 🎤 **Interview answer:** "JSONB is a Postgres column type that stores JSON, and we can search inside it and index it. It's useful when a mostly relational table needs a few flexible fields, like product attributes or the raw data received from SAP for debugging. I don't use it for core data, because we lose the strict rules of normal columns."

### Must-practise SQL queries

> 🎤 **Interview answer:** "For these I first explain my approach, then write. For 'Nth highest' I use `DENSE_RANK`, which handles ties. For 'top per group' I use `RANK` with `PARTITION BY`. For duplicates, `GROUP BY` with `HAVING COUNT(*) > 1`. For 'never ordered', `LEFT JOIN` with `IS NULL`, or `NOT EXISTS`."

```sql
-- 1. Second highest salary
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- 2. Nth highest salary (N = 3)
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2;
-- or
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS r FROM employees
) t WHERE r = 3 LIMIT 1;

-- 3. Highest salary in each department
SELECT department, MAX(salary) FROM employees GROUP BY department;

-- 4. Employee(s) with highest salary per department (full row)
SELECT * FROM (
  SELECT e.*, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS r FROM employees e
) t WHERE r = 1;

-- 5. Find duplicate emails
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- 6. Delete duplicates, keep lowest id (Postgres)
DELETE FROM users a USING users b
WHERE a.email = b.email AND a.id > b.id;

-- 7. Employees earning more than their manager
SELECT e.name FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;

-- 8. Top 3 customers by total spend
SELECT user_id, SUM(amount) AS total FROM orders
GROUP BY user_id ORDER BY total DESC LIMIT 3;

-- 9. Monthly revenue
SELECT DATE_TRUNC('month', created_at) AS month, SUM(amount)
FROM orders GROUP BY 1 ORDER BY 1;

-- 10. Users who registered but never ordered
SELECT u.id, u.name FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 11. Running total
SELECT id, amount, SUM(amount) OVER (ORDER BY created_at) AS running_total FROM orders;

-- 12. Count of employees per department, including empty departments
SELECT d.name, COUNT(e.id) FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;
```

---

## ORMs – Prisma, TypeORM, Mongoose

**Q. What is an ORM? Pros and cons?**

> 🎤 **Interview answer:** "An ORM lets us work with the database using normal code and objects, instead of writing raw SQL everywhere. For example, `prisma.user.findMany()` instead of `SELECT * FROM users`.
>
> **Pros:** faster to write, type safety (especially Prisma), built-in migrations, and protection from SQL injection.
> **Cons:** it can hide slow queries like N+1, and complex queries can be hard.
>
> So I use the ORM for normal CRUD, and raw SQL for complex reports. I've used Prisma at Air India, TypeORM at KindSwap, and Mongoose in my MongoDB projects."

| | Prisma | TypeORM | Mongoose |
|---|---|---|---|
| DB | SQL (+ Mongo) | SQL | MongoDB |
| Schema | `schema.prisma` file | Decorators on classes (`@Entity`) | `new Schema({...})` |
| Types | Auto-generated, very strong | From classes | Need extra typing |
| Migrations | `prisma migrate` | Built-in | Not needed |
| Used by me at | Air India (Notus) | KindSwap (Revinfotech) | Appsinvo, matrimony, Byte Digital |

**Q. What is a migration?**

> 🎤 **Interview answer:** "A migration is a script that changes the database structure, like adding a table, a column or an index. Migrations are saved in Git and run in the same order on every environment, so dev, staging and production databases always match. With Prisma I use `prisma migrate dev` locally and `prisma migrate deploy` on the server. For big tables, I make changes in safe steps: first add the new column as optional, fill the data, then make it required."

---

[Home](README.md) | [← 03 Node.js & Express](03-nodejs-express.md) | Next: [05 React →](05-react.md)
