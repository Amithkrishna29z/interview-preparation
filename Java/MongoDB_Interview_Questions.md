# MongoDB Interview Questions & Answers

## MongoDB Basics

### Q1: What is MongoDB and why is it used?

**Answer:** MongoDB is a document-oriented NoSQL database that stores data in flexible, JSON-like documents. Key advantages: flexible schema (documents can vary in structure), horizontal scaling via sharding, natural JSON format for developers, and built-in high availability via replication.

**Common use cases:** content management, real-time analytics, mobile apps, IoT, e-commerce with flexible product attributes.

### Q2: What are the key components of MongoDB architecture?

**Answer:** Data is organized hierarchically: **Instance → Database → Collection → Document**. Applications connect via language drivers to a `mongod` process backed by the WiredTiger storage engine.

- **Database**: container for collections, stored on disk
- **Collection**: group of documents (like a SQL table, but schema-free)
- **Document**: basic unit of data, stored as BSON (JSON-like)

```javascript
// Collection: products — Document example:
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Laptop",
  "price": 999.99,
  "specs": { "processor": "Intel i7", "ram": "16GB" },
  "tags": ["computer", "electronics"],
  "createdAt": ISODate("2024-01-15T10:30:00Z"),
  "inStock": true
}
```

### Q3: What is BSON and how does it differ from JSON?

**Answer:** BSON (Binary JSON) is the binary format MongoDB uses internally to store and transmit documents. It extends JSON with richer data types.

| Feature | JSON | BSON |
|---------|------|------|
| Format | Text | Binary |
| Data Types | Limited | Extended (ObjectId, Date, Binary, etc.) |
| Traversal | Slower (parse required) | Faster (direct access) |
| Use Case | Data exchange | Storage and networking |

**ObjectId:** A 12-byte, auto-generated `_id` = 4-byte timestamp + 5-byte random + 3-byte counter. Roughly time-ordered and globally unique. Use `objectId.getTimestamp()` to extract creation time.

---

## Data Modeling

### Q4: How do you design data models in MongoDB?

**Answer:** Two main approaches: **embedding** (denormalization) and **referencing** (normalization).

**Embedding** — store related data inside the parent document:
```javascript
// One-to-Few: embed small arrays
{
  "_id": ObjectId("..."),
  "name": "John Doe",
  "phoneNumbers": [
    { "type": "home", "number": "555-1234" },
    { "type": "work", "number": "555-5678" }
  ]
}
// Use when: data is always accessed together, array stays small (<100 items)
```

**Referencing** — store related data in separate documents:
```javascript
// One-to-Many: reference by ID
{ "_id": ObjectId("user123"), "name": "John Doe" }

{
  "_id": ObjectId("order1"),
  "userId": ObjectId("user123"),  // reference
  "total": 99.99
}
// Use when: large arrays, frequently updated data, data accessed independently
```

### Q5: What are the best practices for MongoDB schema design?

**Answer:**

1. **Design for query patterns** — embed data you always fetch together; reference data you access separately.
2. **Avoid large documents** — max document size is 16MB. Use references for unbounded arrays (e.g., thousands of orders).
3. **Use correct BSON types** — store numbers as numbers, dates as `ISODate()`, not strings. Indexes and sorting break with wrong types.
4. **Read-heavy → denormalize; write-heavy → normalize** — duplicating data speeds up reads but means more updates on writes.
5. **Cap unbounded arrays** — e.g., keep only last 10 notifications and store history in a separate collection.

---

## CRUD Operations

### Q6: How do you perform CRUD operations in MongoDB?

**Answer:**

**Create:**
```javascript
db.users.insertOne({ name: "John", email: "john@example.com", age: 30 });
db.users.insertMany([
  { name: "Jane", email: "jane@example.com" },
  { name: "Bob",  email: "bob@example.com"  }
]);
```

**Read:**
```javascript
db.users.find({ age: { $gt: 30 } });
db.users.findOne({ email: "john@example.com" });

// Projection, sort, pagination
db.users.find({ age: { $gte: 18 } }, { name: 1, email: 1, _id: 0 })
  .sort({ createdAt: -1 })
  .limit(10)
  .skip(0);

// Array, nested, and existence queries
db.products.find({ tags: "electronics" });
db.users.find({ "address.city": "New York" });
db.users.find({ email: { $exists: true } });
```

**Update:**
```javascript
db.users.updateOne({ email: "john@example.com" }, { $set: { age: 31 } });
db.users.updateMany({ status: "inactive" }, { $set: { status: "deleted" } });

// Upsert
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "New User", email: "new@example.com" } },
  { upsert: true }
);

// Common update operators
db.users.updateOne({ _id: ObjectId("...") }, {
  $set:  { age: 32 },          // set field
  $inc:  { loginCount: 1 },    // increment
  $unset: { tempField: "" },   // remove field
  $currentDate: { updatedAt: true }
});

// Array update operators
db.users.updateOne({ _id: ObjectId("...") }, {
  $push:     { tags: "new-tag" },       // add to array
  $addToSet: { tags: "unique-tag" },    // add if not exists
  $pull:     { tags: "old-tag" }        // remove from array
});

// Update matched array element
db.orders.updateOne(
  { _id: ObjectId("..."), "items.productId": ObjectId("prod1") },
  { $inc: { "items.$.quantity": 1 } }  // $ = matched element
);
```

**Delete:**
```javascript
db.users.deleteOne({ email: "john@example.com" });
db.users.deleteMany({ status: "deleted" });
db.users.drop();  // drop entire collection
```

---

## Aggregation Framework

### Q7: What is the MongoDB aggregation framework and how do you use it?

**Answer:** The aggregation framework processes documents through a **pipeline** of stages, where each stage transforms the data. Think of it as SQL's `GROUP BY`, `JOIN`, and `WHERE` combined.

**Core stages:**
```javascript
// $match → $group → $sort → $limit (common pattern)
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$total" }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
]);

// $project: reshape output
db.users.aggregate([
  { $project: {
    name: 1,
    fullName: { $concat: ["$firstName", " ", "$lastName"] },
    isAdult: { $gte: ["$age", 18] },
    _id: 0
  }}
]);

// $lookup: left outer join
db.orders.aggregate([
  { $lookup: {
    from: "users",
    localField: "userId",
    foreignField: "_id",
    as: "userDetails"
  }},
  { $unwind: "$userDetails" }
]);

// $unwind: flatten array for per-element processing
db.products.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } }
]);

// $addFields: add computed fields
db.orders.aggregate([
  { $addFields: { isLarge: { $gte: ["$total", 1000] } } }
]);
```

**Other useful stages:** `$facet` (multiple sub-pipelines in one query), `$bucket`/`$bucketAuto` (histogram ranges), date/math/conditional operators (`$year`, `$round`, `$cond`).

---

## Indexing

### Q8: What are the different types of indexes in MongoDB?

**Answer:**

**1. Single Field:**
```javascript
db.users.createIndex({ email: 1 });                          // ascending
db.users.createIndex({ email: 1 }, { unique: true });        // unique
db.users.createIndex({ phone: 1 }, { sparse: true });        // skip docs missing field
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 }); // TTL auto-delete
```

**2. Compound:**
```javascript
db.orders.createIndex({ userId: 1, orderDate: -1, status: 1 });
// Can use: userId | userId+orderDate | userId+orderDate+status
// Cannot use: orderDate alone | status alone (prefix rule)
```

**3. Multikey (Array):** Created automatically when indexing an array field.
```javascript
db.products.createIndex({ tags: 1 });
db.products.find({ tags: "electronics" });  // uses the index
```

**4. Text Index:**
```javascript
db.articles.createIndex({ title: "text", content: "text" });
db.articles.find({ $text: { $search: "mongodb database" } },
                 { score: { $meta: "textScore" } })
           .sort({ score: { $meta: "textScore" } });
```

**5. Geospatial:**
```javascript
db.locations.createIndex({ coordinates: "2dsphere" });
db.locations.find({ coordinates: {
  $near: { $geometry: { type: "Point", coordinates: [-73.97, 40.77] }, $maxDistance: 1000 }
}});
```

**6. Hashed (for sharding):**
```javascript
db.users.createIndex({ _id: "hashed" });  // equality queries only
```

**Index management:**
```javascript
db.users.getIndexes();
db.users.dropIndex("email_1");
db.users.find({ email: "john@example.com" }).explain("executionStats");
db.users.aggregate([{ $indexStats: {} }]);
```

---

## Replication

### Q9: How does replication work in MongoDB?

**Answer:** A **replica set** is a group of `mongod` instances holding the same data, providing redundancy and automatic failover.

- **Primary**: receives all writes
- **Secondaries**: replicate primary data; can serve reads
- **Arbiter**: votes in elections, holds no data
- On primary failure, remaining members elect a new primary automatically

**Read preference** controls which member serves reads: `primary` (default), `primaryPreferred`, `secondary`, `secondaryPreferred`, `nearest`.

**Write concern** controls durability: `w: 1` (primary only), `w: "majority"` (most members), `j: true` (journaled). Higher concern = more durability, more latency.

```javascript
// Initialize 3-member replica set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongodb1:27017" },
    { _id: 1, host: "mongodb2:27017" },
    { _id: 2, host: "mongodb3:27017" }
  ]
});

// Durable write
db.collection.insertOne({ name: "John" }, { writeConcern: { w: "majority", j: true } });
```

---

## Sharding

### Q10: How does sharding work in MongoDB?

**Answer:** Sharding distributes data across multiple machines to support large datasets and high throughput (horizontal scaling).

- **Shards**: each is a replica set holding a subset of data
- **mongos**: query router the application connects to
- **Config servers**: store cluster metadata
- **Shard key**: the field MongoDB uses to split data into chunks

**Strategies**: *range-based* (good for range queries, may distribute unevenly) vs *hashed* (even distribution, poor for range queries).

```javascript
sh.enableSharding("mydb");
sh.shardCollection("mydb.users", { userId: "hashed" });
sh.status();
```

---

## Performance Optimization

### Q11: How do you optimize MongoDB performance?

**Answer:**

**Indexing:**
```javascript
// Covering query — satisfied entirely from the index (no doc fetch)
db.orders.createIndex({ userId: 1, status: 1, total: 1 });
db.orders.find({ userId: ObjectId("..."), status: "completed" }, { total: 1, _id: 0 });

// Analyze a query
db.users.find({ email: "john@example.com" }).explain("executionStats");
```

**Query patterns:**
```javascript
// Prefer high-selectivity fields
db.users.find({ email: "john@example.com" });   // returns 1 doc — good
db.users.find({ gender: "male" });              // returns ~50% — avoid as index

// Project only needed fields
db.users.find({ userId: ObjectId("...") }, { name: 1, email: 1, _id: 0 });

// Range-based pagination (avoid large .skip())
db.users.find({ _id: { $gt: lastSeenId } }).limit(10);
```

**Schema:** embed frequently-accessed data, use correct BSON types, keep documents under 16MB, avoid unbounded arrays.

---

## Transactions

### Q12: How do transactions work in MongoDB?

**Answer:** MongoDB supports multi-document ACID transactions from v4.0 (replica sets) and v4.2 (sharded clusters). Use them when operations across multiple documents/collections must all succeed or all fail.

```javascript
const session = client.startSession();
try {
  session.startTransaction();

  await usersCollection.updateOne(
    { _id: userId }, { $inc: { balance: -amount } }, { session }
  );
  await ordersCollection.insertOne(
    { userId, amount, createdAt: new Date() }, { session }
  );

  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
} finally {
  session.endSession();
}
```

**Key considerations:**
- Transactions require a replica set (not standalone)
- They have performance overhead — prefer single-document operations when possible
- Limited to 16MB total data; avoid long-running transactions
- Use `withTransaction()` helper in official drivers for automatic retry on transient errors
- Prefer embedded documents for atomic single-document operations when ACID across collections isn't needed

---

## MongoDB vs SQL Databases

### Q13: What are the key differences between MongoDB and SQL databases?

| Aspect | MongoDB | SQL |
|--------|---------|-----|
| Data Model | Document (JSON-like) | Table (relational) |
| Schema | Flexible, dynamic | Fixed, predefined |
| Scalability | Horizontal (sharding) | Vertical |
| Joins | `$lookup` (limited) | JOIN (powerful) |
| Transactions | Multi-doc (v4.0+) | Full ACID |
| Schema Changes | No migration needed | ALTER TABLE |
| Consistency | Eventual (default) | Strong |

**Choose MongoDB:** flexible/evolving schema, document-oriented data, horizontal scaling, rapid prototyping.

**Choose SQL:** complex joins, strong data integrity, regulatory compliance, structured consistent data.

---

## Advanced Features

### Q14: What are change streams in MongoDB?

**Answer:** Change streams let applications subscribe to real-time data changes (inserts, updates, deletes) without polling. Useful for event-driven features, cache invalidation, and notifications. Support filtering via aggregation pipeline and resuming after disconnects via a resume token.

```javascript
const changeStream = db.collection('users').watch();
changeStream.on('change', (change) => {
  console.log(change.operationType, change.fullDocument);
});
```

### Q15: What are MongoDB views?

**Answer:** Views are read-only, queryable objects defined by an aggregation pipeline over a source collection. Queried like a normal collection.

```javascript
db.createView(
  "activeUsers",   // view name
  "users",         // source collection
  [
    { $match: { status: "active" } },
    { $project: { name: 1, email: 1, lastLogin: 1, _id: 0 } }
  ]
);

db.activeUsers.find({ lastLogin: { $gte: ISODate("2024-01-01") } });
```

---

## Common Mistakes

```javascript
// 1. Missing index → full collection scan
db.users.createIndex({ email: 1 });  // always index query fields

// 2. Embedding unbounded arrays (16MB doc limit)
// BAD:  { allOrders: [...thousands...] }
// GOOD: { recentOrders: [...last 10...] } + separate orders collection

// 3. Wrong data types
// BAD:  { price: "99.99", createdAt: "2024-01-15" }
// GOOD: { price: 99.99,   createdAt: ISODate("2024-01-15") }

// 4. No write concern (data loss risk on crash)
db.collection.insertOne({ name: "John" }, { writeConcern: { w: "majority", j: true } });

// 5. Not monitoring slow queries
db.setProfilingLevel(1, { slowms: 100 });
db.system.profile.find().sort({ millis: -1 }).limit(10);
```

---

## Quick Reference Cheat Sheet

**Create index:**
```javascript
db.collection.createIndex({ field: 1 });
db.collection.createIndex({ field1: 1, field2: -1 }, { unique: true });
```

**Aggregation pipeline:**
```javascript
db.collection.aggregate([
  { $match: { condition } },
  { $group: { _id: "$field", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]);
```

**Transaction skeleton:**
```javascript
const session = client.startSession();
try {
  session.startTransaction();
  // operations with { session } ...
  await session.commitTransaction();
} catch (e) {
  await session.abortTransaction();
} finally {
  session.endSession();
}
```

**Change stream:**
```javascript
const changeStream = db.collection('name').watch();
changeStream.on('change', (change) => console.log(change));
```

**Key update operators:** `$set`, `$inc`, `$unset`, `$push`, `$pull`, `$addToSet`, `$currentDate`

**Key query operators:** `$gt`, `$gte`, `$lt`, `$lte`, `$eq`, `$ne`, `$in`, `$nin`, `$exists`, `$or`, `$and`, `$all`, `$size`, `$text`
