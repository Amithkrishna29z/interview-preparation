# MongoDB Interview Questions & Answers

## Table of Contents
1. [MongoDB Basics](#mongodb-basics)
2. [Data Modeling](#data-modeling)
3. [CRUD Operations](#crud-operations)
4. [Aggregation Framework](#aggregation-framework)
5. [Indexing](#indexing)
6. [Replication](#replication)
7. [Sharding](#sharding)
8. [Performance Optimization](#performance-optimization)
9. [Transactions](#transactions)
10. [MongoDB vs SQL Databases](#mongodb-vs-sql-databases)
11. [Advanced Features](#advanced-features)
12. [Common Mistakes](#common-mistakes-2)
13. [Short Revision Summary](#short-revision-summary-2)

---

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

**Answer:**

**MongoDB Architecture Components:**

```
┌─────────────────────────────────────────┐
│         Application Layer               │
│   (Drivers for various languages)       │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         MongoDB Instance                │
│  ┌──────────────────────────────────┐  │
│  │      mongod (Database Process)  │  │
│  │  ┌────────────────────────────┐  │  │
│  │  │    Query Engine           │  │  │
│  │  │    Storage Engine          │  │  │
│  │  │    (WiredTiger)            │  │  │
│  │  └────────────────────────────┘  │  │
│  └──────────────────────────────────┘  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Data Storage                    │
│  ┌──────────┐  ┌──────────┐            │
│  │ Database │  │ Database │            │
│  │  (DB)    │  │  (DB)    │            │
│  │ ┌──────┐ │  │ ┌──────┐ │            │
│  │ │ Coll │ │  │ │ Coll │ │            │
│  │ │ ect  │ │  │ │ ect  │ │            │
│  │ └──────┘ │  │ └──────┘ │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
```

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

**BSON Data Types:**

```javascript
// String
{"name": "John Doe"}

// Number (int32, int64, double)
{"age": 30, "price": 99.99, "count": NumberLong("9007199254740991")}

// Boolean
{"isActive": true}

// Null
{"middleName": null}

// Array
{"tags": ["tag1", "tag2", "tag3"]}

// Object (embedded document)
{"address": {"street": "123 Main St", "city": "New York"}}

// ObjectId (MongoDB specific)
{"_id": ObjectId("507f1f77bcf86cd799439011")}

// Date
{"createdAt": ISODate("2024-01-15T10:30:00Z")}

// Binary Data
{"fileData": BinData(0, "SGVsbG8gV29ybGQ=")}

// Regular Expression
{"pattern": /mongodb/i}

// JavaScript Code
{"code": Code("function() { return 'Hello'; }")}

// Timestamp
{"timestamp": Timestamp(1610754600, 1)}

// MinKey/MaxKey (for comparison)
{"minKey": MinKey(), "maxKey": MaxKey()}
```

**ObjectId Structure:**

```javascript
// ObjectId is 12 bytes:
// Bytes 0-3: Timestamp (seconds since epoch)
// Bytes 4-6: Machine identifier
// Bytes 7-8: Process ID
// Bytes 9-11: Counter (incrementing value)

// Example: 507f1f77bcf86cd799439011
// 507f1f77: Timestamp (2012-10-17T20:46:15Z)
// bcf86c: Machine identifier
// d799: Process ID
// 439011: Counter

// Generate ObjectId
var objectId = new ObjectId();
var objectIdFromString = ObjectId("507f1f77bcf86cd799439011");

// Extract timestamp from ObjectId
var timestamp = objectId.getTimestamp();  // Returns Date object
```

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

**3. Use Appropriate Data Types:**

```javascript
// ❌ BAD: Using strings for numbers and dates
{
  "price": "99.99",
  "quantity": "10",
  "createdAt": "2024-01-15"
}

// ✅ GOOD: Use appropriate BSON types
{
  "price": 99.99,                    // Number
  "quantity": 10,                    // Number
  "createdAt": ISODate("2024-01-15") // Date
}
```

**4. Design for Read vs Write Performance:**

```javascript
// Read-heavy: Denormalize (embed)
{
  "_id": "product1",
  "name": "Laptop",
  "category": {
    "_id": "cat1",
    "name": "Electronics",
    "description": "Electronic devices"
  }
}

// Write-heavy: Normalize (reference)
{
  "_id": "product1",
  "name": "Laptop",
  "categoryId": ObjectId("cat1")
}
```

**5. Use Arrays Efficiently:**

```javascript
// ❌ BAD: Unbounded arrays
{
  "_id": "user1",
  "notifications": []  // Can grow indefinitely
}

// ✅ GOOD: Limit array size or use separate collection
{
  "_id": "user1",
  "recentNotifications": []  // Limit to last N notifications
}
```

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

// $redact: Restrict content based on field values
db.documents.aggregate([
  {
    $redact: {
      $cond: {
        if: {$eq: ["$accessLevel", "public"]},
        then: "$$DESCEND",
        else: "$$PRUNE"
      }
    }
  }
]);
```

**Advanced Aggregation:**

```javascript
// Complex aggregation: Customer lifetime value analysis
db.orders.aggregate([
  // Join with users collection
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  {$unwind: "$user"},

  // Calculate customer metrics
  {
    $group: {
      _id: "$userId",
      customerName: {$first: "$user.name"},
      customerEmail: {$first: "$user.email"},
      firstOrderDate: {$min: "$orderDate"},
      lastOrderDate: {$max: "$orderDate"},
      totalOrders: {$sum: 1},
      totalSpent: {$sum: "$total"},
      avgOrderValue: {$avg: "$total"},
      categories: {$addToSet: "$category"}
    }
  },

  // Calculate customer lifetime value and days active
  {
    $project: {
      customerName: 1,
      customerEmail: 1,
      firstOrderDate: 1,
      lastOrderDate: 1,
      totalOrders: 1,
      totalSpent: {$round: ["$totalSpent", 2]},
      avgOrderValue: {$round: ["$avgOrderValue", 2]},
      daysActive: {
        $divide: [
          {$subtract: ["$lastOrderDate", "$firstOrderDate"]},
          1000 * 60 * 60 * 24  // Convert milliseconds to days
        ]
      },
      categoryCount: {$size: "$categories"},
      preferredCategories: {$slice: ["$categories", 3]}
    }
  },

  // Segment customers
  {
    $addFields: {
      segment: {
        $cond: {
          if: {$gte: ["$totalSpent", 1000]},
          then: "high-value",
          else: {
            $cond: {
              if: {$gte: ["$totalSpent", 500]},
              then: "medium-value",
              else: "low-value"
            }
          }
        }
      }
    }
  },

  // Sort by total spent
  {$sort: {totalSpent: -1}}
]);

// Faceted search (multiple aggregations in one query)
db.products.aggregate([
  {
    $facet: {
      "products": [
        {$match: {category: "electronics", price: {$lt: 1000}}},
        {$sort: {rating: -1}},
        {$skip: 0},
        {$limit: 10}
      ],
      "categories": [
        {$group: {_id: "$category", count: {$sum: 1}}},
        {$sort: {count: -1}}
      ],
      "priceRanges": [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, 5000],
            default: "5000+",
            output: {
              count: {$sum: 1},
              products: {$push: "$name"}
            }
          }
        }
      ]
    }
  }
]);
```

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

// Find locations within a polygon
db.locations.find({
  coordinates: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-73.97, 40.77],
          [-73.97, 40.78],
          [-73.96, 40.78],
          [-73.96, 40.77],
          [-73.97, 40.77]
        ]]
      }
    }
  }
});
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

**Answer:** MongoDB replication provides data redundancy and high availability through replica sets.

**Replica Set Components:**

```
┌─────────────────────────────────────────┐
│         Replica Set                     │
│  ┌──────────┐  ┌──────────┐            │
│  │ Primary  │  │ Secondary│            │
│  │ (Master) │  │  (Slave) │            │
│  │  Writes  │  │  Reads   │            │
│  └──────────┘  └──────────┘            │
│       │              │                 │
│       └──────┬───────┘                 │
│              │                         │
│       ┌──────▼──────┐                  │
│       │  Arbiter    │                  │
│       │  (Voting)   │                  │
│       └─────────────┘                  │
└─────────────────────────────────────────┘
```

**Replica Set Configuration:**

```javascript
// Initialize replica set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    {_id: 0, host: "mongodb1.example.com:27017"},
    {_id: 1, host: "mongodb2.example.com:27017"},
    {_id: 2, host: "mongodb3.example.com:27017", arbiterOnly: true}
  ]
});

// Check replica set status
rs.status();

// Check replica set configuration
rs.conf();

// Add member to replica set
rs.add("mongodb4.example.com:27017");

// Add arbiter
rs.addArb("mongodb5.example.com:27017");

// Remove member
rs.remove("mongodb2.example.com:27017");

// Step down primary (force election)
rs.stepDown();
```

**Read Preferences:**

```javascript
// Read from primary (default)
db.collection.find().readPref("primary");

// Read from primary if available, otherwise secondary
db.collection.find().readPref("primaryPreferred");

// Read from secondary
db.collection.find().readPref("secondary");

// Read from secondary if available, otherwise primary
db.collection.find().readPref("secondaryPreferred");

// Read from nearest member (based on network latency)
db.collection.find().readPref("nearest");

// Set read preference for connection
conn = new Mongo("mongodb://mongodb1,mongodb2,mongodb3/mydb?readPreference=secondary");
```

**Write Concern:**

```javascript
// Write concern levels
db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: 1}}  // Default: acknowledge write from primary
);

db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: 2}}  // Acknowledge from 2 members
);

db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: "majority"}}  // Acknowledge from majority of members
);

db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: 1, j: true}}  // Acknowledge write + journaling
);

db.collection.insertOne(
  {name: "John"},
  {writeConcern: {w: 1, j: true, wtimeout: 5000}}  // Timeout after 5 seconds
);
```

---

## Sharding

### Q10: How does sharding work in MongoDB?

**Answer:** Sharding distributes data across multiple machines to support large datasets and high throughput operations.

**Sharding Components:**

```
┌─────────────────────────────────────────┐
│         Application                     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         mongos (Query Router)           │
│         (Multiple instances)             │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
┌───────▼────┐ ┌──▼──────┐ ┌▼──────────┐
│  Shard 1   │ │ Shard 2 │ │ Shard 3  │
│ (Primary)  │ │(Primary)│ │(Primary) │
│  ┌───────┐ ││ ┌──────┐ ││ ┌───────┐ │
│  │Sec 1  │ ││ │Sec 1 │ ││ │Sec 1  │ │
│  │Sec 2  │ ││ │Sec 2 │ ││ │Sec 2  │ │
│  └───────┘ ││ └──────┘ ││ └───────┘ │
└────────────┘ └─────────┘ └───────────┘
        │             │            │
        └─────────────┼────────────┘
                      │
        ┌─────────────▼─────────────┐
        │    Config Servers         │
        │  (3-member replica set)   │
        └───────────────────────────┘
```

**Shard Keys:**

```javascript
// Enable sharding on database
sh.enableSharding("mydb");

// Shard collection with range-based sharding
sh.shardCollection("mydb.users", {_id: 1});

// Shard collection with hash-based sharding (better distribution)
sh.shardCollection("mydb.users", {userId: "hashed"});

// Shard collection with compound shard key
sh.shardCollection("mydb.orders", {userId: 1, orderDate: -1});

// Check shard status
sh.status();

// List shards
sh.getShardMap();

// Move chunk manually (rarely needed)
sh.moveChunk("mydb.users", {_id: ObjectId("...")}, "shard2");
```

**Sharding Strategies:**

```javascript
// Range-based sharding
// Good for: Range queries on shard key
// Bad for: Uneven data distribution
sh.shardCollection("mydb.logs", {timestamp: 1});

// Hash-based sharding
// Good for: Even data distribution, random access
// Bad for: Range queries on shard key
sh.shardCollection("mydb.users", {userId: "hashed"});

// Ranged sharding with tags (data locality)
// Tag specific ranges to specific shards
sh.addShardTag("shard1", "US-East");
sh.addShardTag("shard2", "US-West");

sh.addTagRange("mydb.users", {region: "East"}, {region: "East"}, "US-East");
sh.addTagRange("mydb.users", {region: "West"}, {region: "West"}, "US-West");
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

**Schema Optimization:**

```javascript
// 1. Embed frequently accessed data
{
  "_id": "order1",
  "userId": "user1",
  "items": [
    {
      "productId": "prod1",
      "name": "Laptop",  // Embedded product name
      "price": 999.99    // Embedded price
    }
  ]
}

// 2. Use appropriate data types
{
  "price": 999.99,                    // Number, not string
  "createdAt": ISODate("2024-01-15"), // Date, not string
  "count": 100,                       // Number, not string
  "isActive": true                    // Boolean, not string
}

// 3. Limit document size (<16MB)
{
  "_id": "user1",
  "recentActivity": [],  // Limit to recent items
  // Store full history in separate collection
}

// 4. Use arrays efficiently
// ❌ BAD: Unbounded arrays
{
  "_id": "user1",
  "allNotifications": []  // Can grow indefinitely
}

// ✅ GOOD: Bounded arrays or separate collection
{
  "_id": "user1",
  "recentNotifications": []  // Limit to last N
}
```

**Configuration Optimization:**

```javascript
// MongoDB configuration (mongod.conf)
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # 50-60% of available RAM
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0

operationProfiling:
  mode: slowOp  # or all, none
  slowOpThresholdMs: 100

security:
  authorization: enabled
```

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

**Transaction with Retry Logic:**

```javascript
async function runTransactionWithRetry(txnFunc, session) {
  while (true) {
    try {
      await txnFunc(session);  // Perform transaction
      break;  // Success, exit retry loop
    } catch (error) {
      if (error.errorLabels && error.errorLabels.includes('TransientTransactionError')) {
        console.log("Transient transaction error, retrying...");
        continue;  // Retry
      } else {
        throw error;  // Non-retryable error
      }
    }
  }
}

async function commitTransactionWithRetry(session) {
  while (true) {
    try {
      await session.commitTransaction();
      break;  // Success
    } catch (error) {
      if (error.errorLabels && error.errorLabels.includes('UnknownTransactionCommitResult')) {
        console.log("Unknown commit result, retrying commit...");
        continue;  // Retry commit
      } else {
        throw error;  // Non-retryable error
      }
    }
  }
}

// Usage
const session = client.startSession();
try {
  await runTransactionWithRetry(async (session) => {
    session.startTransaction();
    // Transaction operations...
    await commitTransactionWithRetry(session);
  }, session);
} finally {
  session.endSession();
}
```

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

**Answer:** Change streams allow applications to access real-time data changes without the complexity and risk of tailing the oplog.

**Basic Change Stream:**

```javascript
// Watch for changes in a collection
const changeStream = db.collection('users').watch();

changeStream.on('change', (change) => {
  console.log('Change detected:', change);

  if (change.operationType === 'insert') {
    console.log('New document:', change.fullDocument);
  } else if (change.operationType === 'update') {
    console.log('Updated fields:', change.updateDescription.updatedFields);
  } else if (change.operationType === 'delete') {
    console.log('Deleted document ID:', change.documentKey._id);
  }
});

// Watch with filter
const changeStream = db.collection('orders').watch([
  {$match: {'fullDocument.status': 'completed'}}
]);

// Watch for database changes
const changeStream = db.watch();

// Resume after disconnect
const changeStream = db.collection('users').watch([], {resumeAfter: resumeToken});
changeStream.on('change', (change) => {
  resumeToken = change._id;
  // Process change...
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

### Key MongoDB Concepts

**Data Model:**
- Document-oriented (BSON/JSON)
- Flexible schema
- Embedding vs referencing
- Hybrid approach

**CRUD Operations:**
- Create: insertOne(), insertMany()
- Read: find(), findOne()
- Update: updateOne(), updateMany(), replaceOne()
- Delete: deleteOne(), deleteMany()

**Aggregation Framework:**
- Pipeline-based data processing
- Stages: $match, $group, $project, $lookup, $unwind, $sort, $limit, $skip
- Powerful data transformations

**Indexing:**
- Single field, compound, multikey
- Text, geospatial, hashed indexes
- Index management and monitoring

**Replication:**
- Replica sets for high availability
- Primary-secondary architecture
- Automatic failover

**Sharding:**
- Horizontal scaling
- Shard keys (range, hash)
- Config servers and mongos

**Transactions:**
- Multi-document ACID transactions
- Session-based
- Retry logic for transient errors

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

### Critical Points for Interviews:

1. **BSON vs JSON**: BSON is binary, supports more data types, faster for storage and querying
2. **Embedding vs Referencing**: Embed for one-to-few, reference for one-to-many/many-to-many
3. **Index Types**: Choose based on query patterns (single, compound, multikey, text, geospatial)
4. **Aggregation**: Pipeline approach for data transformation and analytics
5. **Replica Sets**: Provide high availability with automatic failover
6. **Sharding**: Horizontal scaling with shard keys and mongos routers
7. **Transactions**: Multi-document ACID transactions with retry logic
8. **Performance**: Indexing, query optimization, appropriate data modeling
9. **Change Streams**: Real-time data change notifications
10. **Schema Design**: Based on query patterns, not normalization

---

**Next Topics to Study:**
- General Database Concepts (ACID, Normalization, SQL Fundamentals)
- Database Design Patterns
- NoSQL vs SQL Decision Making
- Cloud Database Services (AWS MongoDB, Atlas, etc.)
- Database Security and Compliance