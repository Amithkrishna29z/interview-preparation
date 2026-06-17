# MongoDB Interview Questions & Answers

## MongoDB Basics

### Q1: What is MongoDB and why is it used?

**Answer:** MongoDB is a document-oriented, NoSQL database that stores data in flexible, JSON-like documents. It's used because:

- **Flexible Schema**: No rigid schema, documents can vary in structure
- **Scalability**: Horizontal scaling via sharding
- **Performance**: Optimized for read/write operations
- **Developer Friendly**: Natural JSON format, easy to work with
- **High Availability**: Built-in replication and automatic failover
- **Rich Query Language**: Powerful aggregation and query capabilities
- **Geospatial Support**: Built-in geospatial indexing and queries
- **Open Source**: Community version with enterprise features available

**Real-world use cases:**
- Content management systems
- Real-time analytics
- Mobile applications
- Internet of Things (IoT) data storage
- Social media platforms
- E-commerce platforms with flexible product attributes

### Q2: What are the key components of MongoDB architecture?

**Answer:** Applications connect via language drivers to a `mongod` process (query engine + WiredTiger storage engine), which stores data hierarchically: Instance → Database → Collection → Document.

**Key Components:**

**Database:**
- Container for collections
- Physical storage on disk
- Multiple databases per MongoDB instance

**Collection:**
- Group of MongoDB documents
- Similar to tables in relational databases
- No enforced schema

**Document:**
- Basic unit of data in MongoDB
- JSON-like structure (BSON)
- Key-value pairs with various data types

**Example Structure:**

```javascript
// Database: e-commerce
// Collection: products
// Document:
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Laptop",
  "price": 999.99,
  "category": "Electronics",
  "specs": {
    "processor": "Intel i7",
    "ram": "16GB",
    "storage": "512GB SSD"
  },
  "reviews": [
    {
      "user": "user1",
      "rating": 5,
      "comment": "Great product!"
    }
  ],
  "tags": ["computer", "portable", "electronics"],
  "createdAt": ISODate("2024-01-15T10:30:00Z"),
  "inStock": true
}
```

### Q3: What is BSON and how does it differ from JSON?

**Answer:** BSON (Binary JSON) is a binary-encoded serialization format used to store documents and make remote procedure calls in MongoDB.

**BSON vs JSON:**

| Feature | JSON | BSON |
|---------|------|------|
| Format | Text-based | Binary |
| Size | Larger | Smaller (compressed) |
| Data Types | Limited (string, number, boolean, null, array, object) | Extended (ObjectId, Date, Binary, etc.) |
| Traversal | Slower (parsing required) | Faster (direct access) |
| Indexing | Not supported | Supported |
| Use Case | Data exchange | Storage and networking |

**Common BSON Data Types:**

```javascript
{
  "name": "John Doe",                        // String
  "age": 30,                                 // Number (int32/int64/double)
  "isActive": true,                          // Boolean
  "tags": ["tag1", "tag2"],                  // Array
  "address": {"city": "New York"},           // Object (embedded document)
  "_id": ObjectId("507f1f77bcf86cd799439011"), // ObjectId
  "createdAt": ISODate("2024-01-15T10:30:00Z") // Date
}
```

Other types include Binary, Regular Expression, Timestamp, and MinKey/MaxKey.

**ObjectId Structure:** A 12-byte value = 4-byte timestamp + 5-byte random value + 3-byte counter. This makes `_id` roughly time-ordered and globally unique. Use `objectId.getTimestamp()` to extract the creation time.

---

## Data Modeling

### Q4: How do you design data models in MongoDB?

**Answer:** MongoDB offers flexible data modeling with two main approaches: embedding and referencing.

**Embedding (Denormalization):**

```javascript
// One-to-One: Embed related document
{
  "_id": ObjectId("..."),
  "name": "John Doe",
  "email": "john@example.com",
  "address": {                    // Embedded address
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zip": "10001"
  }
}

// One-to-Few: Embed array of related documents
{
  "_id": ObjectId("..."),
  "name": "John Doe",
  "email": "john@example.com",
  "phoneNumbers": [               // Embedded array (few items)
    {
      "type": "home",
      "number": "555-1234"
    },
    {
      "type": "work",
      "number": "555-5678"
    }
  ]
}

// When to use embedding:
// - One-to-One relationships
// - One-to-Few relationships (typically < 100 items)
// - Data that is always accessed together
// - Data that doesn't change frequently
```

**Referencing (Normalization):**

```javascript
// One-to-Many: Use references for many related documents
// User document
{
  "_id": ObjectId("user123"),
  "name": "John Doe",
  "email": "john@example.com"
}

// Order documents (reference user)
{
  "_id": ObjectId("order1"),
  "userId": ObjectId("user123"),  // Reference to user
  "orderDate": ISODate("2024-01-15"),
  "total": 99.99,
  "items": [
    {"productId": ObjectId("prod1"), "quantity": 2, "price": 49.99}
  ]
}

// Many-to-Many: Use array of references
// Product document
{
  "_id": ObjectId("prod1"),
  "name": "Laptop",
  "categoryIds": [ObjectId("cat1"), ObjectId("cat2")]  // References to categories
}

// Category document
{
  "_id": ObjectId("cat1"),
  "name": "Electronics",
  "productIds": [ObjectId("prod1"), ObjectId("prod2")]  // References to products
}

// When to use referencing:
// - One-to-Many relationships
// - Many-to-Many relationships
// - Large arrays of related items
// - Data that changes frequently
// - When you need to access related data independently
```

**Hybrid Approach:**

```javascript
// Combine embedding and referencing
{
  "_id": ObjectId("order1"),
  "userId": ObjectId("user123"),
  "orderDate": ISODate("2024-01-15"),
  "total": 99.99,
  // Embed product details (frequently accessed together)
  "items": [
    {
      "productId": ObjectId("prod1"),
      "name": "Laptop",              // Denormalized for quick access
      "price": 49.99,                // Price at time of order
      "quantity": 2
    }
  ],
  // Reference full product data (may change)
  "productDetails": [
    {
      "productId": ObjectId("prod1"),
      "currentPrice": 47.99,         // Current price (may differ)
      "stock": 50
    }
  ]
}
```

### Q5: What are the best practices for MongoDB schema design?

**Answer:**

**1. Design for Query Patterns:**

```javascript
// ❌ BAD: Design for data normalization only
// User document
{
  "_id": "user1",
  "name": "John",
  "addressIds": ["addr1", "addr2"]
}

// Address documents
{"_id": "addr1", "userId": "user1", "street": "123 Main St"}
{"_id": "addr2", "userId": "user1", "street": "456 Oak Ave"}

// ✅ GOOD: Design based on query patterns
// If you always need user's address with user info:
{
  "_id": "user1",
  "name": "John",
  "addresses": [
    {"street": "123 Main St", "type": "home"},
    {"street": "456 Oak Ave", "type": "work"}
  ]
}
```

**2. Avoid Large Documents (>16MB):**

```javascript
// ❌ BAD: Too much embedded data
{
  "_id": "user1",
  "name": "John",
  "allOrders": [...]  // Thousands of orders
}

// ✅ GOOD: Use references for large datasets
{
  "_id": "user1",
  "name": "John",
  "recentOrders": [...last 10 orders...]  // Embed recent data
}
// Orders stored separately with userId reference
```

**3. Use Appropriate Data Types:** Store numbers, dates, and booleans as their BSON types — not strings — so queries, sorting, and indexes work correctly (e.g. `price: 99.99` and `createdAt: ISODate(...)`, not `"99.99"` / `"2024-01-15"`).

**4. Design for Read vs Write Performance:** Read-heavy workloads favor denormalization (embed related data); write-heavy workloads favor normalization (reference by id) to avoid updating duplicated data in many places.

**5. Use Arrays Efficiently:** Avoid unbounded arrays that grow forever (e.g. `notifications: []`). Cap them to the last N items or move the history to a separate collection.

---

## CRUD Operations

### Q6: How do you perform CRUD operations in MongoDB?

**Answer:**

**Create Operations:**

```javascript
// Insert single document
db.users.insertOne({
  name: "John Doe",
  email: "john@example.com",
  age: 30,
  createdAt: new Date()
});

// Insert multiple documents
db.users.insertMany([
  {name: "Jane Doe", email: "jane@example.com", age: 25},
  {name: "Bob Smith", email: "bob@example.com", age: 35}
]);

// Insert with custom _id
db.users.insertOne({
  _id: "custom_id_123",
  name: "Alice",
  email: "alice@example.com"
});
```

**Read Operations:**

```javascript
// Find all documents
db.users.find();

// Find with condition
db.users.find({age: {$gt: 30}});

// Find one document
db.users.findOne({email: "john@example.com"});

// Find with projection (return specific fields)
db.users.find(
  {age: {$gte: 18}},           // Query
  {name: 1, email: 1, _id: 0}  // Projection
);

// Find with sorting and limiting
db.users.find({})
  .sort({createdAt: -1})  // -1 for descending, 1 for ascending
  .limit(10)
  .skip(0);  // For pagination

// Complex queries
db.users.find({
  $or: [
    {age: {$lt: 25}},
    {age: {$gt: 60}}
  ],
  status: "active",
  createdAt: {
    $gte: ISODate("2024-01-01"),
    $lt: ISODate("2025-01-01")
  }
});

// Array queries
db.products.find({tags: "electronics"});  // Contains "electronics"
db.products.find({tags: {$all: ["electronics", "computers"]}});  // Contains all
db.products.find({tags: {$size: 3}});  // Array has exactly 3 elements

// Nested document queries
db.users.find({"address.city": "New York"});

// Element queries
db.users.find({
  email: {$exists: true},      // Field exists
  age: {$type: "int"},         // Field is integer
  phone: {$ne: null}           // Field is not null
});
```

**Update Operations:**

```javascript
// Update single document
db.users.updateOne(
  {email: "john@example.com"},
  {$set: {age: 31}}
);

// Update multiple documents
db.users.updateMany(
  {status: "inactive"},
  {$set: {status: "deleted"}}
);

// Upsert (update or insert)
db.users.updateOne(
  {email: "newuser@example.com"},
  {
    $set: {
      name: "New User",
      email: "newuser@example.com",
      createdAt: new Date()
    }
  },
  {upsert: true}
);

// Update operators
db.users.updateOne(
  {_id: ObjectId("...")},
  {
    $set: {age: 32},                    // Set field value
    $inc: {loginCount: 1},              // Increment field
    $mul: {score: 1.5},                 // Multiply field
    $rename: {oldName: "newName"},     // Rename field
    $unset: {temporaryField: ""},       // Remove field
    $min: {lastLogin: new Date()},      // Set to minimum of current and new
    $max: {maxScore: 100},              // Set to maximum of current and new
    $currentDate: {updatedAt: true}     // Set to current date
  }
);

// Array update operators
db.users.updateOne(
  {_id: ObjectId("...")},
  {
    $push: {tags: "new-tag"},           // Add to array
    $addToSet: {tags: "unique-tag"},    // Add if not exists
    $pull: {tags: "old-tag"},           // Remove from array
    $pop: {tags: 1},                    // Remove last element
    $pullAll: {tags: ["tag1", "tag2"]}, // Remove multiple elements
    $each: {tags: ["tag1", "tag2"]},    // Used with $push/$addToSet
    $position: {tags: 0},               // Insert at specific position
    $slice: {tags: -5}                  // Limit array size
  }
);

// Update nested array elements
db.orders.updateOne(
  {_id: ObjectId("..."), "items.productId": ObjectId("prod1")},
  {$inc: {"items.$.quantity": 1}}       // $ refers to matched array element
);

// Bulk write operations
db.users.bulkWrite([
  {insertOne: {document: {name: "User1", email: "user1@example.com"}}},
  {updateOne: {
    filter: {email: "john@example.com"},
    update: {$set: {age: 32}}
  }},
  {deleteOne: {filter: {email: "old@example.com"}}}
]);
```

**Delete Operations:**

```javascript
// Delete single document
db.users.deleteOne({email: "john@example.com"});

// Delete multiple documents
db.users.deleteMany({status: "deleted"});

// Delete all documents (use with caution!)
db.users.deleteMany({});

// Drop collection
db.users.drop();
```

---

## Aggregation Framework

### Q7: What is the MongoDB aggregation framework and how do you use it?

**Answer:** The MongoDB aggregation framework provides powerful data processing capabilities using a pipeline approach.

**Basic Aggregation Pipeline:**

```javascript
// Simple aggregation
db.orders.aggregate([
  {$match: {status: "completed"}},
  {$group: {_id: "$userId", total: {$sum: "$total"}}},
  {$sort: {total: -1}},
  {$limit: 10}
]);

// Real-world example: Monthly sales report
db.orders.aggregate([
  // Stage 1: Filter completed orders
  {
    $match: {
      status: "completed",
      orderDate: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2025-01-01")
      }
    }
  },

  // Stage 2: Group by month and calculate totals
  {
    $group: {
      _id: {
        year: {$year: "$orderDate"},
        month: {$month: "$orderDate"}
      },
      totalOrders: {$sum: 1},
      totalRevenue: {$sum: "$total"},
      averageOrderValue: {$avg: "$total"}
    }
  },

  // Stage 3: Sort by month
  {
    $sort: {"_id.year": 1, "_id.month": 1}
  },

  // Stage 4: Format output
  {
    $project: {
      _id: 0,
      month: {
        $dateToString: {
          format: "%Y-%m",
          date: {
            $dateFromParts: {
              year: "$_id.year",
              month: "$_id.month"
            }
          }
        }
      },
      totalOrders: 1,
      totalRevenue: {$round: ["$totalRevenue", 2]},
      averageOrderValue: {$round: ["$averageOrderValue", 2]}
    }
  }
]);
```

**Common Aggregation Stages:**

```javascript
// $match: Filter documents
db.users.aggregate([
  {$match: {age: {$gte: 18, $lte: 65}}}
]);

// $group: Group documents
db.orders.aggregate([
  {
    $group: {
      _id: "$userId",
      totalSpent: {$sum: "$total"},
      orderCount: {$sum: 1},
      avgOrderValue: {$avg: "$total"},
      minOrder: {$min: "$total"},
      maxOrder: {$max: "$total"}
    }
  }
]);

// $project: Reshape documents
db.users.aggregate([
  {
    $project: {
      name: 1,
      email: 1,
      fullName: {$concat: ["$name.firstName", " ", "$name.lastName"]},
      age: 1,
      isAdult: {$gte: ["$age", 18]},
      _id: 0
    }
  }
]);

// $lookup: Left outer join (like SQL JOIN)
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "userDetails"
    }
  },
  {
    $unwind: "$userDetails"  // Unwind array from $lookup
  }
]);

// $unwind: Deconstruct array
db.products.aggregate([
  {$unwind: "$tags"},
  {$group: {_id: "$tags", count: {$sum: 1}}}
]);

// $sort: Sort documents
db.users.aggregate([
  {$sort: {createdAt: -1, name: 1}}
]);

// $limit: Limit number of documents
db.users.aggregate([
  {$sort: {score: -1}},
  {$limit: 10}
]);

// $skip: Skip documents (for pagination)
db.users.aggregate([
  {$skip: 20},
  {$limit: 10}
]);

// $addFields: Add computed fields to documents
db.orders.aggregate([
  {$addFields: {isLarge: {$gte: ["$total", 1000]}}}
]);
```

**Advanced stages (awareness-level):** Beyond the basics, MongoDB offers powerful operators you'll meet on larger datasets:

- `$facet` — run multiple sub-pipelines in one query (e.g. results + category counts + price buckets for a search page).
- `$bucket` / `$bucketAuto` — group documents into ranges (histograms).
- `$redact` — include/exclude content based on field values (access control).
- Date, math, and conditional operators (`$year`, `$round`, `$cond`, `$switch`) for in-pipeline computation.

---

## Indexing

### Q8: What are the different types of indexes in MongoDB?

**Answer:**

**Index Types:**

**1. Single Field Index:**

```javascript
// Create index on single field
db.users.createIndex({email: 1});  // 1 for ascending, -1 for descending

// Create unique index
db.users.createIndex({email: 1}, {unique: true});

// Create sparse index (only index documents that have the field)
db.users.createIndex({phone: 1}, {sparse: true});

// Create TTL index (auto-delete documents after expiry)
db.sessions.createIndex({createdAt: 1}, {expireAfterSeconds: 3600});  // 1 hour
```

**2. Compound Index:**

```javascript
// Create compound index
db.orders.createIndex({userId: 1, orderDate: -1, status: 1});

// Query can use index for:
// - userId
// - userId + orderDate
// - userId + orderDate + status

// Cannot efficiently use for:
// - orderDate only
// - status only
// - orderDate + status (missing userId)
```

**3. Multikey Index (Array Index):**

```javascript
// Index on array field (automatically becomes multikey)
db.products.createIndex({tags: 1});

// Queries that can use the index:
db.products.find({tags: "electronics"});
db.products.find({tags: {$all: ["electronics", "computers"]}});
```

**4. Text Index:**

```javascript
// Create text index
db.articles.createIndex({content: "text"});

// Text search
db.articles.find({$text: {$search: "mongodb database"}});

// Text search with language
db.articles.find({$text: {$search: "mongodb database", $language: "english"}});

// Text search with sorting by relevance
db.articles.find(
  {$text: {$search: "mongodb"}},
  {score: {$meta: "textScore"}}
).sort({score: {$meta: "textScore"}});

// Compound text index
db.articles.createIndex({title: "text", content: "text"});
```

**5. Geospatial Index:**

```javascript
// 2dsphere index (for GeoJSON)
db.locations.createIndex({coordinates: "2dsphere"});

// Find locations near a point
db.locations.find({
  coordinates: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-73.97, 40.77]  // Longitude, latitude
      },
      $maxDistance: 1000  // Meters
    }
  }
});
// $geoWithin can also find points inside a given Polygon.
```

**6. Hashed Index:**

```javascript
// Create hashed index (for sharding)
db.users.createIndex({_id: "hashed"});

// Good for equality queries only
db.users.find({_id: ObjectId("...")});
```

**7. Wildcard Index (MongoDB 4.2+):**

```javascript
// Index all fields in a document
db.products.createIndex({"$**": 1});

// Index specific fields with wildcard
db.products.createIndex({"productDetails.$**": 1});
```

**Index Management:**

```javascript
// List indexes
db.users.getIndexes();

// Drop index
db.users.dropIndex("email_1");

// Drop all indexes except _id
db.users.dropIndexes();

// Index usage statistics
db.users.aggregate([{$indexStats: {}}]);

// Create index in background (doesn't block operations)
db.users.createIndex({email: 1}, {background: true});

// Create index with custom name
db.users.createIndex({email: 1}, {name: "user_email_idx"});
```

---

## Replication

### Q9: How does replication work in MongoDB?

**Answer (awareness-level):** MongoDB replication provides data redundancy and high availability through **replica sets** — a group of `mongod` instances holding the same data.

- **Primary**: receives all writes. **Secondaries**: replicate the primary's data and can serve reads. **Arbiter**: votes in elections but holds no data.
- **Automatic failover**: if the primary goes down, the remaining members elect a new primary (election internals are a senior/DBA topic).
- **Read preference** controls which member serves reads: `primary` (default), `primaryPreferred`, `secondary`, `secondaryPreferred`, `nearest`.
- **Write concern** (`w`) controls write acknowledgement: `w: 1` (primary only), `w: "majority"` (durable across most members), optional `j: true` for journaling. Higher concern = more durability, more latency.

```javascript
// Initialize a 3-member replica set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    {_id: 0, host: "mongodb1:27017"},
    {_id: 1, host: "mongodb2:27017"},
    {_id: 2, host: "mongodb3:27017"}
  ]
});

// Durable write
db.collection.insertOne({name: "John"}, {writeConcern: {w: "majority", j: true}});
```

---

## Sharding

### Q10: How does sharding work in MongoDB?

**Answer (awareness-level):** Sharding distributes data across multiple machines (**shards**) to support large datasets and high throughput. It is the mechanism behind MongoDB's horizontal scaling.

- **Components**: **Shards** (each a replica set holding a subset of data), **mongos** (query router the app connects to), and **config servers** (store cluster metadata).
- **Shard key**: the field(s) MongoDB uses to split data into chunks across shards. Choosing it well is critical and a senior/DBA concern.
- **Strategies**: *range-based* (good for range queries, can distribute unevenly) vs *hashed* (even distribution, poor for range queries).

```javascript
sh.enableSharding("mydb");
sh.shardCollection("mydb.users", {userId: "hashed"});  // hashed sharding
sh.status();
```

---

## Performance Optimization

### Q11: How do you optimize MongoDB performance?

**Answer:**

**Indexing Optimization:**

```javascript
// 1. Create appropriate indexes
db.users.createIndex({email: 1});  // For email lookups
db.orders.createIndex({userId: 1, createdAt: -1});  // Compound index

// 2. Use covering queries (query can be satisfied from index)
db.orders.createIndex({userId: 1, status: 1, total: 1});
// This query uses only the index:
db.orders.find(
  {userId: ObjectId("..."), status: "completed"},
  {total: 1, _id: 0}
);

// 3. Monitor index usage
db.users.aggregate([{$indexStats: {}}]);

// 4. Remove unused indexes
db.users.dropIndex("unused_field_1");

// 5. Use explain to analyze queries
db.users.find({email: "john@example.com"}).explain("executionStats");
```

**Query Optimization:**

```javascript
// 1. Use selective queries
// ❌ BAD: Low selectivity
db.users.find({gender: "male"});  // Returns ~50% of documents

// ✅ GOOD: High selectivity
db.users.find({email: "john@example.com"});  // Returns 1 document

// 2. Limit returned fields
// ❌ BAD: Returns entire document
db.users.find({userId: ObjectId("...")});

// ✅ GOOD: Returns only needed fields
db.users.find(
  {userId: ObjectId("...")},
  {name: 1, email: 1, _id: 0}
);

// 3. Use pagination
db.users.find().skip(20).limit(10);

// 4. Avoid large skip values
// ❌ BAD: Inefficient for large offsets
db.users.find().skip(10000).limit(10);

// ✅ GOOD: Use range-based pagination
var lastId = ...;  // ID of last document from previous page
db.users.find({_id: {$gt: lastId}}).limit(10);

// 5. Use projection to reduce document size
db.products.find(
  {category: "electronics"},
  {name: 1, price: 1, rating: 1, _id: 0}
);
```

**Schema Optimization (recap):** Embed frequently-accessed data, use correct BSON types (numbers/dates, not strings), keep documents under 16MB, and avoid unbounded arrays. See the Data Modeling section for details.

**Configuration (awareness-level):** Production tuning lives in `mongod.conf` — notably the WiredTiger cache size (set to ~50% of RAM), journaling, and `operationProfiling` to log slow queries. This is largely a DBA/ops concern.

---

## Transactions

### Q12: How do transactions work in MongoDB?

**Answer:** MongoDB supports multi-document ACID transactions starting from version 4.0 (for replica sets) and 4.2 (for sharded clusters).

**Basic Transaction:**

```javascript
// Start session
const session = client.startSession();

try {
  // Start transaction
  session.startTransaction();

  // Perform operations within transaction
  const db = client.db("mydb");
  const usersCollection = db.collection("users");
  const ordersCollection = db.collection("orders");

  // Update user balance
  await usersCollection.updateOne(
    {_id: userId},
    {$inc: {balance: -amount}},
    {session}
  );

  // Create order
  await ordersCollection.insertOne(
    {
      userId: userId,
      amount: amount,
      createdAt: new Date()
    },
    {session}
  );

  // Commit transaction
  await session.commitTransaction();
  console.log("Transaction committed");
} catch (error) {
  // Abort transaction on error
  await session.abortTransaction();
  console.error("Transaction aborted:", error);
} finally {
  session.endSession();
}
```

**Retry Logic (awareness-level):** Production transactions should retry on transient errors. The driver may flag errors with `TransientTransactionError` (retry the whole transaction) or `UnknownTransactionCommitResult` (retry just the commit). Most official drivers provide a `withTransaction()` helper that handles this retry loop automatically.

**Transaction Considerations:**

```javascript
// 1. Transactions require replica sets (not standalone)
// 2. Transactions have performance overhead
// 3. Transactions are limited to 16MB of total data
// 4. Long-running transactions can cause performance issues
// 5. Some operations are not supported in transactions (e.g., create indexes)

// Use transactions when:
// - You need ACID guarantees across multiple documents/collections
// - Data integrity is critical
// - Operations must all succeed or all fail

// Consider alternatives when:
// - You can use embedded documents (atomic operations on single document)
// - You can use denormalization
// - Performance is more important than strong consistency
```

---

## MongoDB vs SQL Databases

### Q13: What are the key differences between MongoDB and SQL databases?

**Answer:**

| Aspect | MongoDB (NoSQL) | SQL Databases |
|--------|-----------------|---------------|
| **Data Model** | Document-based (JSON-like) | Table-based (relational) |
| **Schema** | Flexible, dynamic | Fixed, predefined |
| **Query Language** | MongoDB Query Language | SQL |
| **Scalability** | Horizontal (sharding) | Vertical (scaling up) |
| **Joins** | $lookup (limited) | JOIN (powerful) |
| **Transactions** | Multi-doc (4.0+) | Full ACID support |
| **Relationships** | Embedded documents, references | Foreign keys, normalization |
| **Schema Changes** | No migration needed | ALTER TABLE (complex) |
| **Consistency** | Eventual consistency (default) | Strong consistency |
| **Indexing** | B-tree, text, geospatial, hashed | B-tree, hash, full-text |
| **Aggregation** | Aggregation pipeline | GROUP BY, HAVING |
| **Data Types** | Rich (JSON, arrays, ObjectId) | Standard SQL types |
| **Use Cases** | Flexible data, rapid iteration | Structured data, complex relationships |

**When to Choose MongoDB:**
- Rapidly evolving schema
- Document-oriented data
- Need for horizontal scaling
- Real-time analytics
- Big data and unstructured data
- Rapid prototyping and development
- Geographic data processing

**When to Choose SQL Databases:**
- Complex relationships and joins
- Strong data integrity requirements
- Complex transactions
- Structured and consistent data
- Reporting and analytics
- Regulatory compliance requirements
- Existing SQL expertise

---

## Advanced Features

### Q14: What are change streams in MongoDB?

**Answer (awareness-level):** Change streams let applications subscribe to real-time data changes (inserts, updates, deletes) without tailing the oplog directly. Useful for event-driven features, cache invalidation, and notifications. They support filtering with an aggregation pipeline and resuming after disconnects via a resume token (internals are an advanced topic).

```javascript
// Watch a collection for changes
const changeStream = db.collection('users').watch();

changeStream.on('change', (change) => {
  console.log(change.operationType, change.fullDocument);
});
```

### Q15: What are MongoDB views and how do you use them?

**Answer:** Views are read-only queryable objects computed on demand.

**Create View:**

```javascript
// Create view for active users
db.createView(
  "activeUsers",  // View name
  "users",        // Source collection
  [
    {$match: {status: "active"}},
    {$project: {name: 1, email: 1, lastLogin: 1, _id: 0}}
  ]
);

// Query view like a collection
db.activeUsers.find({lastLogin: {$gte: ISODate("2024-01-01")}});

// Create view with aggregation
db.createView(
  "monthlySales",
  "orders",
  [
    {$match: {status: "completed"}},
    {
      $group: {
        _id: {
          year: {$year: "$orderDate"},
          month: {$month: "$orderDate"}
        },
        totalRevenue: {$sum: "$total"},
        orderCount: {$sum: 1}
      }
    },
    {$sort: {"_id.year": 1, "_id.month": 1}}
  ]
);

// Query the view
db.monthlySales.find({"_id.year": 2024});
```

---

## Common Mistakes

### Mistake 1: Not creating indexes

```javascript
// ❌ BAD: No index, full collection scan
db.users.find({email: "john@example.com"});

// ✅ GOOD: Create index
db.users.createIndex({email: 1});
db.users.find({email: "john@example.com"});
```

### Mistake 2: Embedding too much data

```javascript
// ❌ BAD: Document too large (>16MB limit)
{
  "_id": "user1",
  "name": "John",
  "allOrders": [...]  // Thousands of orders
}

// ✅ GOOD: Use references
{
  "_id": "user1",
  "name": "John",
  "recentOrders": [...]  // Last 10 orders
}
// Store full order history in separate collection
```

### Mistake 3: Not using appropriate data types

```javascript
// ❌ BAD: Using strings for numbers and dates
{
  "price": "99.99",
  "quantity": "10",
  "createdAt": "2024-01-15"
}

// ✅ GOOD: Use appropriate BSON types
{
  "price": 99.99,
  "quantity": 10,
  "createdAt": ISODate("2024-01-15")
}
```

### Mistake 4: Ignoring write concerns

```javascript
// ❌ BAD: No write concern (data loss risk)
db.collection.insertOne({name: "John"});

// ✅ GOOD: Appropriate write concern
db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: "majority", j: true}}
);
```

### Mistake 5: Not monitoring performance

```javascript
// ✅ GOOD: Enable profiling
db.setProfilingLevel(1, {slowms: 100});

// ✅ GOOD: Monitor slow queries
db.system.profile.find().sort({millis: -1}).limit(10);

// ✅ GOOD: Check index usage
db.users.aggregate([{$indexStats: {}}]);
```

---

## Short Revision Summary

### Quick Reference

**Create Index:**
```javascript
db.collection.createIndex({field: 1});
db.collection.createIndex({field1: 1, field2: -1}, {unique: true});
```

**Aggregation Pipeline:**
```javascript
db.collection.aggregate([
  {$match: {condition}},
  {$group: {_id: "$field", count: {$sum: 1}}},
  {$sort: {count: -1}}
]);
```

**Transaction:**
```javascript
const session = client.startSession();
try {
  session.startTransaction();
  // Operations...
  session.commitTransaction();
} catch (error) {
  session.abortTransaction();
} finally {
  session.endSession();
}
```

**Change Stream:**
```javascript
const changeStream = db.collection('name').watch();
changeStream.on('change', (change) => {
  console.log(change);
});
```