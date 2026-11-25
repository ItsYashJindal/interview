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

> 🎤 **Interview answer:** "In MongoDB, create is `insertOne` or `insertMany`. Read is `find` with a filter and a projection, usually with sort, skip or limit, or `findOne`. Update is `updateOne` or `updateMany` with operators like `$set`, `$inc`, `$push` and `$addToSet`, and `upsert: true` inserts the document if nothing matches. Delete is `deleteOne` or `deleteMany`. In real projects I almost always use update operators instead of replacing whole documents, because they're atomic and avoid overwriting concurrent changes. For example, I reduce stock with `$inc` and a condition `stock >= qty` in the filter, so it can never go negative."

```js
// Create
db.users.insertOne({ name: "Yash", age: 27 });
db.users.insertMany([{ name: "A" }, { name: "B" }]);

// Read
db.users.find({ age: { $gte: 18 } }, { name: 1, email: 1 })  // filter, projection
        .sort({ createdAt: -1 }).skip(0).limit(10);
db.users.findOne({ email: "a@x.com" });

// Update
db.users.updateOne({ _id: id }, { $set: { name: "New" }, $inc: { loginCount: 1 } });
db.users.updateMany({ active: false }, { $set: { archived: true } });
db.users.updateOne({ email }, { $set: { name } }, { upsert: true }); // insert if not found

// Delete
db.users.deleteOne({ _id: id });
db.users.deleteMany({ archived: true });
```

**Common operators:**
- Compare: `$eq $ne $gt $gte $lt $lte $in $nin`
- Logic: `$and $or $not $nor`
- Element: `$exists $type`
- Array: `$all $elemMatch $size`
- Update: `$set $unset $inc $push $pull $addToSet $rename`

**Q. updateOne vs findOneAndUpdate?**

> 🎤 **Interview answer:** "`updateOne` only returns metadata, like matchedCount and modifiedCount. `findOneAndUpdate` returns the document itself, before or after the update depending on the `new` or `returnDocument` option. Both are atomic on a single document. I use `updateOne` when I just need to change something, since it's lighter, and `findOneAndUpdate` when I need the updated document back, for example to atomically claim a job and get its details in one step."

`updateOne` returns only a result count. `findOneAndUpdate` returns the document (before or after update with `{ new: true }` in Mongoose / `returnDocument: "after"` in driver). Both are atomic for a single document.

**Q. Soft delete vs hard delete?**

> 🎤 **Interview answer:** "A hard delete permanently removes the record. A soft delete just marks it, with `isDeleted: true` or a `deletedAt` timestamp, so the data can be restored, audited, and doesn't break references like old orders pointing to a deleted product. The trade-off is that every query must filter out deleted records. I handle that in a base query or a Mongoose pre-find hook, and I index the field or use a partial index. For legal or privacy deletion requests, we still do a hard delete or anonymise the data."

Soft delete = set `isDeleted: true` / `deletedAt` so data can be recovered and audited. Remember to filter it out in all queries (and index it).

---

### 2. Indexing

**Q. What is an index?**

> 🎤 **Interview answer:** "An index is a separate sorted data structure, a B-tree in MongoDB, that stores the values of one or more fields with pointers to the documents. It's like the index at the back of a book: instead of reading every page, you go straight to the right one. Without an index MongoDB does a COLLSCAN and reads every document. With one, it does an IXSCAN and reads only the matching entries. The right indexes were the main reason I could bring a user-listing API down from 4.5 seconds to 550 milliseconds."

A separate sorted data structure (B-tree) that stores field values with pointers to documents. Like a book index – find a topic without reading every page.
Without an index MongoDB does a **COLLSCAN** (reads every document). With an index it does **IXSCAN**.

**Types of indexes:**
| Type | Example | Use |
|---|---|---|
| Single field | `{ email: 1 }` | Lookup by one field |
| Compound | `{ status: 1, createdAt: -1 }` | Filter + sort together |
| Unique | `{ email: 1 }, { unique: true }` | No duplicate emails |
| Multikey | index on an array field | Search inside arrays (tags) |
| Text | `{ title: "text" }` | Basic keyword search |
| TTL | `{ createdAt: 1 }, { expireAfterSeconds: 300 }` | Auto-delete OTPs/sessions |
| 2dsphere | `{ location: "2dsphere" }` | Geo queries ("near me") – used in matrimony project |
| Partial | index only where `isDeleted: false` | Smaller index |
| Sparse | skip documents missing the field | Optional fields |

**Q. ESR rule for compound indexes**

> 🎤 **Interview answer:** "ESR stands for Equality, Sort, Range. It's the recommended order for fields in a compound index. First the fields you match exactly, like `status: 'active'`, then the fields you sort on, like `createdAt`, and last the fields with range conditions, like `age >= 25`. This way MongoDB can jump to the equality values, read entries already in sorted order without an in-memory sort, and then apply the range. If you put the range before the sort, MongoDB has to sort in memory, which is slow and has memory limits."

Order fields as **E**quality → **S**ort → **R**ange.
Query: `find({ status: "active", age: { $gte: 25 } }).sort({ createdAt: -1 })`
Best index: `{ status: 1, createdAt: -1, age: 1 }`

**Q. Prefix rule**

> 🎤 **Interview answer:** "A compound index can be used by any query on its leading fields, its prefixes. So an index on a, b, c supports queries on a, on a and b, and on a, b and c, but not queries on only b or only c, because the index is sorted by a first. That's why I design compound indexes around the most common query patterns, and why I don't create a separate index on `a` if the compound index already starts with it."

Index `{ a: 1, b: 1, c: 1 }` can serve queries on `a`, `a+b`, `a+b+c` – but **not** `b` alone or `c` alone.

**Q. How do you check if a query uses an index?**

> 🎤 **Interview answer:** "I run the query with `.explain('executionStats')`. I check three things: the stage, where IXSCAN is good and COLLSCAN on a large collection is bad; totalDocsExamined compared to nReturned, which should be close, because examining 100,000 documents to return 10 means the index is wrong or missing; and executionTimeMillis. If I see a SORT stage, the sort isn't covered by the index. In SQL the equivalent is `EXPLAIN ANALYZE`."

```js
db.users.find({ email: "a@x.com" }).explain("executionStats");
```
Look at: `stage` (IXSCAN good, COLLSCAN bad), `totalDocsExamined` vs `nReturned` (should be close), `executionTimeMillis`.

**Q. Downsides of indexes?**

> 🎤 **Interview answer:** "Indexes speed up reads but have costs. Every insert, update and delete must also update every index, so writes get slower. They use disk and, more importantly, RAM, because the working set of indexes should fit in memory. Too many or unused indexes waste resources. So I index only for real query patterns, prefer one good compound index over many single-field ones, and check `$indexStats` to remove unused indexes."

- Every insert/update must also update all indexes → slower writes.
- Uses RAM and disk.
- Too many unused indexes waste memory. Check usage with `$indexStats`.

**Covered query:** when all fields in the filter and projection are in the index, MongoDB answers from the index only, without reading documents – fastest.

---

### 3. Aggregation pipeline

**Q. What is it?**

> 🎤 **Interview answer:** "The aggregation pipeline is MongoDB's framework for data processing and analytics. Documents flow through a sequence of stages, and each stage transforms them, like Linux pipes. `$match` filters, `$group` groups and calculates sums or averages, `$project` shapes the fields, `$sort` and `$limit` order and paginate, `$lookup` joins other collections, `$unwind` flattens arrays, and `$facet` runs several pipelines in parallel. I've used it for sales reports, top-selling products, dashboard stats and geo-search with `$geoNear`. Performance-wise, I always put `$match` and `$sort` first so they can use indexes, and project early to reduce document size."

A list of stages; documents flow through each stage and get filtered, changed, grouped. Like Linux pipes.

**Common stages:**
| Stage | Does |
|---|---|
| `$match` | Filter (like WHERE) – put it **first** so indexes are used |
| `$project` / `$addFields` | Choose / compute fields |
| `$group` | Group and calculate (`$sum`, `$avg`, `$max`, `$push`) |
| `$sort`, `$skip`, `$limit` | Order and paginate |
| `$lookup` | Join with another collection |
| `$unwind` | Turn an array into one document per element |
| `$facet` | Run several pipelines at once (data + total count) |
| `$count` | Count documents |
| `$geoNear` | Geo distance (must be first stage) |

**Example – monthly sales report for delivered orders:**
```js
db.orders.aggregate([
  { $match: { status: "delivered", createdAt: { $gte: ISODate("2026-01-01") } } },
  { $group: {
      _id: { month: { $month: "$createdAt" } },
      totalSales: { $sum: "$amount" },
      orders: { $sum: 1 },
      avgOrder: { $avg: "$amount" },
  }},
  { $sort: { "_id.month": 1 } },
]);
```

**Example – top 5 selling products:**
```js
db.orders.aggregate([
  { $unwind: "$items" },
  { $group: { _id: "$items.productId", qty: { $sum: "$items.qty" } } },
  { $sort: { qty: -1 } },
  { $limit: 5 },
  { $lookup: { from: "products", localField: "_id", foreignField: "_id", as: "product" } },
  { $unwind: "$product" },
  { $project: { _id: 0, name: "$product.name", qty: 1 } },
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

**Tips:** `$match` and `$sort` early (to use indexes), `$project` early to drop unused fields, aggregation has 100 MB memory limit per stage → `allowDiskUse: true` for big data.

---

### 4. Population vs lookup

> 🎤 **Interview answer:** "Both join data from another collection, but in different places. Mongoose `populate` runs in Node: Mongoose first fetches the main documents and then sends a separate query with `$in` for the referenced ids, and merges the results. It's very easy to use, with a `ref` in the schema, but it's extra round trips, and you can't filter or group on the joined fields in the database. `$lookup` is an aggregation stage that does the join inside MongoDB in one query, so I can filter, sort and group on the joined data. I use populate for simple cases like an order with the user's name, and `$lookup` for reports. If I need either one constantly, I reconsider the schema and maybe embed or duplicate a few fields."

| | Mongoose `populate()` | MongoDB `$lookup` |
|---|---|---|
| Where it runs | In Node (Mongoose sends **extra queries**) | Inside MongoDB (one aggregation) |
| Queries | 1 + 1 per populated path | 1 |
| Ease | Very easy, uses `ref` in schema | More code |
| Filter/sort/group on joined data | Limited | Full power of pipeline |
| Best for | Simple "get order with user name" | Reports, filtering on joined fields, large data |

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

**Gotcha:** populate in a list of 1000 orders is fine (Mongoose batches with `$in`), but nested populates on big lists can get slow. Both are slower than embedding – if you join very often, re-check your schema design.

---

### 5. MongoDB schema design

> 🎤 **Interview answer:** "The main rule I follow is: design the schema around how the data is read, not how it would look in tables. I embed when the data is read together, is one-to-few and doesn't grow without limit, like addresses inside a user. I reference when it's one-to-many with unbounded growth or used independently, like orders, messages or payments. I keep the 16 MB document limit in mind, and never let an array grow forever.
>
> I also use patterns like extended reference, which copies a few needed fields like the product name and price into the order as a snapshot, and the computed pattern, which stores counters like ratingCount instead of recalculating on every read. In my matrimony project, preferences and photos are embedded in the profile, while interests, messages and payments are separate collections with proper indexes, including a 2dsphere index for location."

**Main rule:** *Design for how the data is read*, not just how it looks.

**Embed vs reference:**
| Embed (store inside) | Reference (store id) |
|---|---|
| Data is read together (user + address) | Data is used separately |
| One-to-few (a user has 3 addresses) | One-to-many / many-to-many (user has 10,000 orders) |
| Child doesn't make sense alone | Child is large or grows without limit |
| One read, atomic update | Need join (populate/$lookup) |

**Limits to remember:** document max size **16 MB**; arrays that grow forever (all chat messages inside a conversation document) are a bad idea – store messages in a separate collection.

**Patterns (simple):**
- **Subset pattern** – keep last 5 reviews embedded in product, rest in a reviews collection.
- **Extended reference** – copy a few fields you always need (store `userName` inside order to avoid a join). Accept small duplication.
- **Computed pattern** – store `totalOrders` / `rating` counters instead of calculating on every read.
- **Bucket pattern** – group time-series data (e.g. one document per sensor per hour).

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
Messages, interests (sent/received) and payments are separate collections because they grow a lot.

---

## SQL

### 6. SQL joins

> 🎤 **Interview answer:** "Joins combine rows from tables based on a related column. INNER JOIN returns only rows that match in both tables, like users who have orders. LEFT JOIN returns all rows from the left table, with NULLs where there's no match, so I use it for things like 'all users with their order count, including zero', or with `WHERE o.id IS NULL` to find users who never ordered. RIGHT JOIN is the mirror image. FULL OUTER JOIN returns everything from both, and MySQL doesn't support it, so you UNION a left and a right join. CROSS JOIN gives every combination. A SELF JOIN joins a table to itself, like employees to their managers."

Tables: `users(id, name)`, `orders(id, user_id, amount)`

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows that match in both tables |
| `LEFT JOIN` | All rows from left + matching from right (NULL if no match) |
| `RIGHT JOIN` | All rows from right + matching from left |
| `FULL OUTER JOIN` | All rows from both (not in MySQL – use UNION of left and right) |
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

> 🎤 **Interview answer:** "A primary key uniquely identifies each row: it must be unique and not null, there's one per table, it can be composite, and it's indexed automatically. A foreign key is a column that references another table's primary key, and the database enforces referential integrity, so you can't create an order for a user that doesn't exist. With ON DELETE we decide what happens to the children: CASCADE, SET NULL or RESTRICT. One practical point: Postgres doesn't automatically index foreign key columns, so I add those indexes myself, because joins and deletes depend on them. In the Air India SAP–WMS work, POs, PO lines, GRNs and materials are all linked with foreign keys."

- **Primary key** – uniquely identifies each row. Unique + NOT NULL. One per table (can be multiple columns = composite key). Automatically indexed.
- **Foreign key** – a column that points to the primary key of another table. Keeps data consistent (can't create an order for a user that doesn't exist).
- **Unique key** – unique values but can allow NULL; a table can have many.
- **Candidate key** – any column that could be a primary key. **Composite key** – PK made of 2+ columns (e.g. `order_id + product_id`).

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  amount NUMERIC(10,2) NOT NULL CHECK (amount >= 0),
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_orders_user_id ON orders(user_id);  -- FK is NOT auto-indexed in Postgres
```

**ON DELETE options:** `CASCADE` (delete children), `SET NULL`, `RESTRICT` / `NO ACTION` (block delete).

**Q. Natural key vs surrogate key?** Natural = real data (email). Surrogate = generated id (SERIAL/UUID). Prefer surrogate as PK because real data can change.

> 🎤 **Interview answer:** "A natural key is real business data used as the key, like an email or a PO number. A surrogate key is a generated id with no business meaning, like an auto-increment or a UUID. I prefer surrogate keys as primary keys, because business data can change, for example a user changing their email, and changing a primary key breaks all references. I still put a unique constraint on the natural key, like a unique email."

**Q. UUID vs auto-increment?** Auto-increment: small, fast, ordered, but guessable and hard to merge across DBs. UUID: unique everywhere, not guessable, bigger and random order (slower inserts in B-tree; UUIDv7 is time-ordered and fixes this).

> 🎤 **Interview answer:** "Auto-increment ids are small, fast to index and naturally ordered, but they're guessable, which is a security issue if exposed in URLs, and they're hard to merge across databases or generate before inserting. UUIDs are globally unique and not guessable, and they can be generated in the app, which is good for distributed systems, but they're larger and random UUIDv4 values cause scattered B-tree inserts. UUIDv7 solves that because it's time-ordered. A common approach is an internal bigint id plus a public UUID."

---

### 8. Transactions

**Q. What is a transaction?**

> 🎤 **Interview answer:** "A transaction groups multiple database operations into one unit, so either all of them succeed and are committed, or if anything fails, all of them are rolled back. The classic example is a money transfer: debiting A and crediting B must both happen or neither. Transactions follow ACID: atomicity, consistency, isolation and durability. In the Air India GRN flow, I used a Prisma `$transaction` to create the GRN and update the received quantity on the PO together, so we never have a GRN without the PO being updated."

A group of queries that run as **one unit** – all succeed or all are rolled back. Example: transfer money – debit A and credit B must both happen or neither.

**ACID:**

> 🎤 **Interview answer:** "Atomicity means all or nothing: if any step fails, everything is rolled back. Consistency means the database moves from one valid state to another, and all constraints, foreign keys and checks still hold. Isolation means concurrent transactions don't see each other's half-finished work, and how strictly depends on the isolation level. Durability means once a transaction is committed, it survives crashes, because it's written to the write-ahead log."

- **Atomicity** – all or nothing.
- **Consistency** – data moves from one valid state to another (constraints hold).
- **Isolation** – parallel transactions don't see each other's half-done work.
- **Durability** – once committed, data survives a crash.

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
  await tx.purchaseOrder.update({ where: { id: poId }, data: { receivedQty: { increment: qty } } });
});
```

**MongoDB (multi-document, needs replica set):**
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
Note: a single-document update in MongoDB is already atomic. Use transactions only when several documents must change together.

**Isolation levels (low → high):**

> 🎤 **Interview answer:** "There are four standard levels. Read Uncommitted allows dirty reads, meaning you can see uncommitted data. Read Committed, the Postgres default, only shows committed data, but the same row read twice can change, which is a non-repeatable read. Repeatable Read, the MySQL InnoDB default, guarantees the same row reads the same within the transaction, though phantom rows are possible in theory. Serializable behaves as if transactions ran one by one, which is safest but slowest, and may need retries on serialization failures. For things like stock and wallet updates, I usually stay on Read Committed and use row locks with `SELECT FOR UPDATE` or atomic conditional updates, instead of raising the level for everything."

| Level | Prevents | Allows |
|---|---|---|
| Read Uncommitted | – | Dirty reads |
| Read Committed (Postgres default) | Dirty reads | Non-repeatable reads |
| Repeatable Read (MySQL InnoDB default) | Dirty + non-repeatable reads | Phantom reads (in theory) |
| Serializable | Everything | Slowest, may need retries |

- **Dirty read** – reading data another transaction hasn't committed.
- **Non-repeatable read** – same row read twice gives different values.
- **Phantom read** – same query returns new rows the second time.

**Locks:** `SELECT ... FOR UPDATE` locks the selected rows until commit – used for stock or wallet updates.

> 🎤 **Interview answer:** "`SELECT ... FOR UPDATE` locks the selected rows until the transaction commits, so another transaction trying to update them has to wait. That's pessimistic locking, good when conflicts are likely, like stock or wallet balance. The alternative is optimistic locking with a version column, where we update only if the version is unchanged and retry if zero rows were updated, which is good when conflicts are rare. For deadlocks, the database detects them and kills one transaction, and we prevent them by always locking rows in the same order and keeping transactions short."

**Optimistic locking:** keep a `version` column; `UPDATE ... WHERE id = 1 AND version = 5`; if 0 rows updated, someone else changed it → retry.
**Deadlock:** two transactions wait for each other's locks. DB kills one. Avoid by always locking rows in the same order and keeping transactions short.

See also [11 Scenarios → Q7](11-scenario-questions.md#7-how-do-you-manage-database-transactions).

---

### 9. Query optimisation

> 🎤 **Interview answer:** "I follow a measure-first approach. First, find the slow queries using the slow query log, `pg_stat_statements`, the MongoDB profiler or APM. Then run EXPLAIN to see whether it's doing a full scan, how many rows it examines, and whether it's sorting in memory. The usual fixes are the right composite index on the WHERE, JOIN and ORDER BY columns, selecting only the needed columns, fixing N+1 queries with joins or `$in`, paginating, preferably with cursors for deep pages, and avoiding functions on indexed columns or leading wildcards that block index use. After that comes caching in Redis, and for bigger problems, summary tables, read replicas or partitioning.
>
> My real example is the user-listing API at Appsinvo: from 4.5 seconds to 550 milliseconds, by adding compound indexes matching the filters and sort, using projection and lean to return only the needed fields, and moving filtering from Node into the query."

**Step-by-step approach:**
1. **Find slow queries** – slow query log (MySQL), `pg_stat_statements` (Postgres), MongoDB profiler / Atlas, APM.
2. **Run EXPLAIN** – `EXPLAIN ANALYZE SELECT ...` (Postgres) / `.explain("executionStats")` (Mongo). Look for full table scan (`Seq Scan`), high rows examined, sorts on disk.
3. **Add the right index** – on columns in WHERE, JOIN, ORDER BY. Composite index in the right order.
4. **Select only needed columns** – no `SELECT *`; in Mongo use projection and `.lean()`.
5. **Fix N+1 queries** – one query for a list, then one query per row → use JOIN / `include` / `$in` / populate.
6. **Paginate** – `LIMIT`; for deep pages use keyset (cursor) pagination.
7. **Avoid functions on indexed columns** – `WHERE YEAR(created_at) = 2026` can't use the index; use `created_at >= '2026-01-01' AND created_at < '2027-01-01'`.
8. **Avoid leading wildcard** – `LIKE '%abc'` can't use a normal index (use full-text search).
9. **Cache** heavy and frequent reads in Redis.
10. **Bigger fixes** – denormalise / summary tables, read replicas, partitioning.

**My real example:** User-listing API was 4.5 s → 550 ms by adding compound indexes for the filters used, returning only needed fields (smaller payload), and moving filtering into the DB query instead of filtering in Node. Details in [13 Resume → API optimisation](13-resume-questions.md#cut-a-user-listing-api-from-45-s-to-550-ms).

---

## Extra MongoDB questions

### Q. SQL vs NoSQL – when to use which?

> 🎤 **Interview answer:** "SQL databases like Postgres or MySQL have a fixed schema, strong relationships with joins and foreign keys, and mature ACID transactions. They're best for highly relational, consistency-critical data like orders, payments, inventory or ERP data. NoSQL databases like MongoDB store flexible documents, make it easy to evolve the schema, scale horizontally with sharding, and fit data that's read as a whole, like profiles, catalogues and content. Honestly, both can handle most use cases today, so I choose based on the data shape. That's why I used MongoDB for the matrimony profiles and geo-search, and PostgreSQL with Prisma for the Air India SAP–WMS integration, where the data is strictly relational."

| SQL (Postgres/MySQL) | NoSQL (MongoDB) |
|---|---|
| Fixed schema, tables, relations | Flexible JSON-like documents |
| Strong joins and transactions | Joins are limited; embed instead |
| Money, orders, inventory, ERP (like SAP–WMS) | Profiles, catalogs, content, logs, fast-changing fields |
| Scale mostly vertically + read replicas | Built-in horizontal scaling (sharding) |

Honest answer: both can do most jobs. Choose by data shape and team knowledge.

### Q. Replica set vs sharding

> 🎤 **Interview answer:** "A replica set is several copies of the same data, usually three nodes: one primary takes writes, and secondaries replicate it. If the primary fails, an election promotes a secondary automatically. So it's about high availability and redundancy, and optionally read scaling. Sharding splits the data across multiple servers based on a shard key, so each shard holds part of the data. It's about horizontal scaling when data or write load is too big for one server. In production, each shard is itself a replica set. Choosing the shard key is critical: it needs high cardinality and even distribution, otherwise one shard becomes a hotspot."

- **Replica set** – copies of the same data on 3+ nodes (1 primary for writes, secondaries for reads/backup). Gives **high availability** – if primary fails, a secondary becomes primary.
- **Sharding** – data is **split** across servers by a shard key. Gives **horizontal scale** for huge data. Choose a shard key with high cardinality and even distribution.

### Q. What is `.lean()` in Mongoose?

> 🎤 **Interview answer:** "By default Mongoose returns full document objects with change tracking, getters, setters, virtuals and methods like `save`, which is heavy. `.lean()` returns plain JavaScript objects instead, which is much faster and uses less memory, often several times faster for large lists. I use lean on all read-only APIs, like listings. I don't use it when I need to modify and save the document, or when I rely on virtuals or getters."

Returns plain JS objects instead of full Mongoose documents. Much faster and lighter for read-only APIs. You lose getters/setters, virtuals, `.save()`.

### Q. Mongoose middleware (hooks)

> 🎤 **Interview answer:** "Mongoose hooks are functions that run before or after certain operations. The common ones are `pre('save')`, for example to hash a password if it was modified, and `post('save')` to trigger side effects. There are also query hooks like `pre('find')`, which I've used to automatically exclude soft-deleted records. The big gotcha is that `updateOne` and `findOneAndUpdate` don't trigger save hooks, so password hashing in pre-save won't run on those. You need query middleware or to go through `save`."

`pre("save")` to hash password, `post("save")` to send event. Note: `updateOne`/`findOneAndUpdate` do **not** trigger `save` hooks.

### Q. Geo query (matrimony "profiles near me")

> 🎤 **Interview answer:** "In the matrimony project, every profile stores its location as a GeoJSON Point with coordinates in longitude, latitude order, and the collection has a 2dsphere index. For 'matches near me' I use `$near`, or `$geoNear` as the first aggregation stage, with the user's coordinates and a `$maxDistance` in metres, combined with filters like gender, age range and religion, sorted by distance. The common mistake is swapping latitude and longitude. For privacy, we never return exact coordinates, only a rounded distance."

```js
Profile.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [lng, lat] },
      $maxDistance: 50000, // meters
    },
  },
  gender: "female",
}).limit(20);
```
Requires a `2dsphere` index. Coordinates order is **[longitude, latitude]**.

### Q. What is the `_id` / ObjectId?

> 🎤 **Interview answer:** "ObjectId is MongoDB's default 12-byte primary key. It contains a 4-byte timestamp, a 5-byte random value and a 3-byte counter. So it can be generated on the client without a central counter, it's unique across machines, and it's roughly sortable by creation time. You can even get the creation date from it with `getTimestamp()`. Sorting by `_id` is a common way to do cursor pagination."

12 bytes: timestamp (4) + random (5) + counter (3). So it's roughly sortable by creation time, and you can get creation time from it.

### Q. Capped collections

> 🎤 **Interview answer:** "A capped collection has a fixed size or document count. Once it's full, it automatically overwrites the oldest documents, like a circular buffer, and it keeps insertion order. It's good for logs or recent activity feeds where only the latest data matters. Today, TTL indexes are often more flexible for expiring data by time."

Fixed-size collections that overwrite the oldest documents – good for logs.

### Q. Write concern and read preference

> 🎤 **Interview answer:** "Write concern controls when a write is acknowledged. `w: 1` means the primary has it, and `w: 'majority'` means most replica set members have it, which is safer because it survives a primary failover. I use majority for critical data like orders and payments. Read preference controls where reads go. The default primary always gives the latest data, while `secondary` or `secondaryPreferred` spreads load to replicas, but the data may be slightly stale, so I use it only for reports or analytics."

- Write concern `w: "majority"` – write is confirmed only after most replicas have it (safer).
- Read preference `secondary` – read from replicas to reduce load on primary (data may be slightly old).

### Q. Change streams

> 🎤 **Interview answer:** "Change streams let an application subscribe to real-time inserts, updates and deletes on a collection, database or cluster, using the oplog, so they need a replica set. They're resumable with a resume token, so you don't miss events after a restart. Use cases are triggering notifications or events when data changes, syncing data to a search index or cache, and audit trails, without polling the database."

Listen to inserts/updates in a collection in real time (needs a replica set). Useful to trigger events or sync data.

---

## Extra SQL questions

### Q. WHERE vs HAVING

> 🎤 **Interview answer:** "`WHERE` filters individual rows before grouping happens, so it can't use aggregate functions. `HAVING` filters groups after `GROUP BY`, so it works with aggregates like `SUM` or `COUNT`. For example, to find customers who spent more than 10,000 on paid orders, `WHERE status = 'paid'` filters the rows first and `HAVING SUM(amount) > 10000` filters the grouped totals. For performance, I filter as much as possible in WHERE, so fewer rows are grouped."

`WHERE` filters rows **before** grouping. `HAVING` filters **groups after** `GROUP BY`.
```sql
SELECT user_id, SUM(amount) total FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(amount) > 10000;
```

### Q. DELETE vs TRUNCATE vs DROP

> 🎤 **Interview answer:** "`DELETE` removes specific rows with a WHERE clause, row by row. It's logged, can be rolled back and fires triggers, but it's slow on big tables. `TRUNCATE` removes all rows at once, very fast with minimal logging, resets identity counters and doesn't fire row triggers. `DROP` removes the whole table, structure, indexes and data. In production I'm very careful with TRUNCATE and DROP, and I never run them without a backup and a second check."

- `DELETE` – removes chosen rows, can use WHERE, can be rolled back, triggers fire.
- `TRUNCATE` – removes all rows fast, resets identity; minimal logging.
- `DROP` – removes the whole table (structure + data).

### Q. UNION vs UNION ALL

> 🎤 **Interview answer:** "Both combine the results of two queries with the same columns. `UNION` removes duplicate rows, which requires a sort or hash step, so it's slower. `UNION ALL` just appends all rows, including duplicates, so it's faster. I use UNION ALL by default when I know there are no duplicates, or when duplicates are fine."

`UNION` removes duplicates (slower). `UNION ALL` keeps all rows (faster).

### Q. Normalisation (simple)

> 🎤 **Interview answer:** "Normalisation is organising tables to remove duplicate data and avoid update anomalies. 1NF means each column holds a single atomic value, with no repeating groups. 2NF means every non-key column depends on the whole primary key, not part of a composite key. 3NF means non-key columns depend only on the key, not on other non-key columns. For example, the customer's city shouldn't be stored in the orders table. In practice I normalise transactional data to 3NF, and denormalise on purpose for read-heavy reports or snapshots, like copying the price into order items."

Organising tables to avoid duplicate data.
- **1NF** – each cell has a single value, no repeating groups.
- **2NF** – 1NF + every non-key column depends on the whole primary key.
- **3NF** – 2NF + non-key columns don't depend on other non-key columns.
**Denormalisation** – adding some duplicate data on purpose to make reads faster (reports).

### Q. Clustered vs non-clustered index

> 🎤 **Interview answer:** "A clustered index defines the physical order in which rows are stored, so there can be only one per table. In MySQL InnoDB the primary key is the clustered index. A non-clustered, or secondary, index is a separate structure that stores the indexed values with a pointer to the row, or the primary key in InnoDB, and a table can have many. Lookups via a secondary index need an extra step to fetch the row, unless the index covers all the needed columns."

- Clustered – table rows are physically stored in index order (one per table; InnoDB primary key).
- Non-clustered – separate structure pointing to rows (many per table).

### Q. What are views, stored procedures, triggers?

> 🎤 **Interview answer:** "A view is a saved SELECT query that you can query like a table. It's good for simplifying complex joins or limiting which columns users see. A stored procedure is a precompiled program stored in the database that can contain logic, loops and transactions, and is called by name. A trigger is code that runs automatically on INSERT, UPDATE or DELETE, like writing an audit log. I mostly keep business logic in the application layer for testability and version control, and use views or triggers for things like reporting or auditing."

- **View** – a saved query used like a table.
- **Stored procedure** – saved SQL program run on the DB server.
- **Trigger** – SQL that runs automatically on INSERT/UPDATE/DELETE.

### Q. Window functions

> 🎤 **Interview answer:** "Window functions calculate across a set of related rows without collapsing them like GROUP BY does, so every row keeps its own data plus the calculated value. The `OVER` clause defines the window, with `PARTITION BY` for groups and `ORDER BY` for order. Common uses are ranking within a group, like top earners per department, running totals, and comparing with previous rows using LAG and LEAD. For ranking, ROW_NUMBER gives unique numbers, RANK leaves gaps after ties, and DENSE_RANK doesn't leave gaps. That's why DENSE_RANK is the right choice for 'Nth highest salary'."

Calculate across rows without collapsing them like GROUP BY.
```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
```
`ROW_NUMBER` (1,2,3,4), `RANK` (1,2,2,4), `DENSE_RANK` (1,2,2,3).

### Q. Connection pooling

> 🎤 **Interview answer:** "Opening a new database connection for every request is slow, because of the TCP and auth handshake, and the database has a connection limit. A pool keeps a set of open connections and lends them to requests, returning them after use. I size the pool based on the database's `max_connections` divided by the number of app instances. With PM2 cluster or many containers, each process has its own pool, so it adds up fast. In serverless environments like Lambda, many concurrent functions can exhaust connections, so we use RDS Proxy."

Opening a DB connection is slow. A pool keeps N connections open and reuses them. Set pool size based on DB limits (e.g. Postgres `max_connections`). In serverless (Lambda), use RDS Proxy to avoid too many connections.

### Q. N+1 problem

> 🎤 **Interview answer:** "The N+1 problem is when the code runs one query to fetch a list and then one more query for each item. For example, fetch 100 users, then query orders for each user in a loop, which is 101 queries. It works in development but becomes very slow with real data. The fix is fetching the related data in one go: a JOIN or Prisma `include`, or one query with `WHERE user_id IN (...)` and grouping the results in code. In Mongoose, `populate` does the `$in` batching for us. I spot it by logging the queries per request."

```js
const users = await User.findAll();             // 1 query
for (const u of users) u.orders = await Order.findAll({ where: { userId: u.id } }); // N queries
```
Fix: one JOIN or `include`, or one query with `WHERE user_id IN (...)`.

### Q. Postgres JSONB

> 🎤 **Interview answer:** "JSONB is Postgres's binary JSON column type. It stores JSON in a parsed format that supports operators for querying inside the document and can be indexed with GIN indexes. It's useful when a mostly relational schema needs some flexible fields, like product attributes, metadata or raw payloads from external systems, for example storing the raw SAP payload for debugging alongside the structured columns. I don't overuse it for core relational data, since you lose constraints and clear types."

Postgres can store JSON (`JSONB`) and index it (GIN index). Good for flexible fields inside a relational DB.

### Must-practise SQL queries

> 🎤 **Interview answer:** "For these I explain my approach before I write. For 'Nth highest', I use DENSE_RANK in a subquery, which handles ties correctly. For 'top per group', I use RANK with PARTITION BY. For duplicates, GROUP BY with HAVING COUNT greater than 1. For 'never ordered', LEFT JOIN with IS NULL, or NOT EXISTS. I also mention edge cases like ties, NULLs and empty groups, and which columns should be indexed."

```sql
-- 1. Second highest salary
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);

-- 2. Nth highest salary (N = 3)
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2;
-- or
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) r FROM employees
) t WHERE r = 3 LIMIT 1;

-- 3. Highest salary in each department
SELECT department, MAX(salary) FROM employees GROUP BY department;

-- 4. Employee(s) with highest salary per department (full row)
SELECT * FROM (
  SELECT e.*, RANK() OVER (PARTITION BY department ORDER BY salary DESC) r FROM employees e
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

> 🎤 **Interview answer:** "An ORM maps database tables or collections to classes and objects, so we work with code instead of writing raw SQL everywhere. The benefits are faster development, type safety, especially with Prisma, built-in migrations, and protection from SQL injection through parameterised queries. The downsides are that it can hide inefficient queries like N+1, complex queries can be awkward or slow, and it adds abstraction. So I use the ORM for regular CRUD and switch to raw SQL or the query builder for complex reports. I've used Prisma at Air India, TypeORM at KindSwap and Mongoose in my MongoDB projects."

Maps tables/collections to classes/objects so you write code instead of raw SQL.
Pros: faster development, type safety (Prisma), migrations, protection from SQL injection.
Cons: can create slow queries (N+1), harder for complex queries → use raw SQL for those.

| | Prisma | TypeORM | Mongoose |
|---|---|---|---|
| DB | SQL (+ Mongo) | SQL | MongoDB |
| Schema | `schema.prisma` file | Decorators on classes (`@Entity`) | `new Schema({...})` |
| Types | Auto-generated, very strong | From classes | Need extra typing |
| Migrations | `prisma migrate` | Built-in | Not needed (schemaless) |
| Used by me at | Air India (Notus) | KindSwap (Revinfotech) | Appsinvo, matrimony, Byte Digital |

**Q. What is a migration?**

> 🎤 **Interview answer:** "A migration is a versioned script that changes the database schema, like creating a table, adding a column or adding an index. Migrations are committed to Git and run in the same order on every environment, so dev, staging and production schemas always match, and there's a history of changes. With Prisma I use `prisma migrate dev` locally and `prisma migrate deploy` in CI. For big tables I keep migrations backward-compatible, for example add a nullable column, backfill, then make it required, so deploys don't break running code."

A versioned script that changes the DB structure (add column, add index). Stored in git, run in the same order on every environment so all DBs match.

---

[Home](README.md) | [← 03 Node.js & Express](03-nodejs-express.md) | Next: [05 React →](05-react.md)
