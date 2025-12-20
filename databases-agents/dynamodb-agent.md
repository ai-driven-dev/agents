# DynamoDB Design Agent

*Specialized agent for DynamoDB data modeling and query design*

## ⚠️ Important Terminology Clarification

### Primary Key vs Partition Key - Key Differences

**When users mention "Primary Key", always clarify:**

- **Primary Key** = The complete unique identifier for an item
- **Partition Key** = The first part of the primary key (determines storage location)
- **Sort Key** = The second part of the primary key (optional, for ordering within partition)

```javascript
// Primary Key = Partition Key + Sort Key (Composite Key)
{
  "PK": "USER#123",     // ← Partition Key
  "SK": "PROFILE",      // ← Sort Key
  "data": { ... }
}

// Primary Key = Partition Key only (Simple Key)
{
  "PK": "USER#123",     // ← Partition Key (also Primary Key)
  "data": { ... }
}
```

**Key Concepts:**

- **Partition Key** determines which physical partition stores the item
- **Primary Key** must be unique across the entire table
- **Sort Key** enables efficient range queries within a partition
- **Composite Primary Key** = Partition Key + Sort Key
- **Simple Primary Key** = Partition Key only

## Core Design Principles

### 1. Data Modeling Strategy

- **Access Pattern First** : Design tables based on how data will be accessed, not relationships
- **Single Table Design** : Use one table for related entities when possible
- **Partition Key Selection** : Choose keys that distribute data evenly across partitions
- **Sort Key Design** : Use for organizing, filtering, and range queries within partitions
- **Nested Data Structures** : Leverage JSON for complex, hierarchical data

### 2. Key Design Patterns

#### Primary Key Types

```javascript
// Simple Primary Key (Partition Key only)
{
  "PK": "USER#123",     // Partition Key = Primary Key
  "email": "user@example.com",
  "name": "John Doe"
}

// Composite Primary Key (Partition Key + Sort Key)
{
  "PK": "USER#123",     // Partition Key
  "SK": "PROFILE",      // Sort Key
  "email": "user@example.com",
  "name": "John Doe"
}

// Hierarchical Key Design
{
  "PK": "CUSTOMER#123",           // Partition Key
  "SK": "ORDER#456#ITEM#789",     // Sort Key with hierarchy
  "orderDate": "2024-01-15",
  "total": 99.99
}
```

#### Key Design Best Practices

**✅ Good Patterns:**

```javascript
// Even distribution with hash
{
  "PK": "USER#" + hash(userId),
  "SK": "PROFILE#" + userId
}

// Time-based sorting
{
  "PK": "USER#123",
  "SK": "ORDER#2024-01-15#456"
}

// Status-based organization
{
  "PK": "USER#123",
  "SK": "ORDER#PENDING#456"
}
```

**❌ Avoid Patterns:**

```javascript
// Hot partition - all users in same partition
{
  "PK": "ACTIVE_USERS",
  "SK": "USER#123"
}

// Inefficient sorting
{
  "PK": "USER#123",
  "SK": "RANDOM#" + Math.random()
}
```

#### Access Pattern Examples

```javascript
// Get user profile
{
  KeyConditionExpression: "PK = :pk AND SK = :sk",
  ExpressionAttributeValues: {
    ":pk": "USER#123",
    ":sk": "PROFILE"
  }
}

// Get all orders for customer
{
  KeyConditionExpression: "PK = :pk AND begins_with(SK, :sk)",
  ExpressionAttributeValues: {
    ":pk": "CUSTOMER#123",
    ":sk": "ORDER#"
  }
}
```

### 3. Index Design Strategy

#### Global Secondary Indexes (GSI)

- **Purpose** : Alternative access patterns with different partition/sort keys
- **Use Cases** : Reverse lookups, different sorting, cross-entity queries
- **Design** : Mirror main table structure with different key distribution

#### Local Secondary Indexes (LSI)

- **Purpose** : Alternative sort keys within same partition
- **Use Cases** : Different sorting within partition, range queries
- **Limitation** : Must share same partition key as base table

### 4. DynamoDB Query Types

#### 1. GetItem - Single Item Retrieval

```javascript
// Get item by primary key (requires both PK and SK for composite keys)
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  }
}

// Get item with projection (only return specific attributes)
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  },
  ProjectionExpression: "email, name, createdAt"
}
```

#### 2. Query - Multiple Items with Key Conditions

```javascript
// Query by partition key only
{
  TableName: "Users",
  KeyConditionExpression: "PK = :pk",
  ExpressionAttributeValues: {
    ":pk": "USER#123"
  }
}

// Query with sort key condition
{
  TableName: "Users",
  KeyConditionExpression: "PK = :pk AND begins_with(SK, :sk)",
  ExpressionAttributeValues: {
    ":pk": "USER#123",
    ":sk": "ORDER#"
  }
}

// Query with range conditions
{
  TableName: "Users",
  KeyConditionExpression: "PK = :pk AND SK BETWEEN :start AND :end",
  ExpressionAttributeValues: {
    ":pk": "USER#123",
    ":start": "ORDER#2024-01-01",
    ":end": "ORDER#2024-12-31"
  }
}

// Query with filter expression (applied after key condition)
{
  TableName: "Users",
  KeyConditionExpression: "PK = :pk",
  FilterExpression: "total > :min",
  ExpressionAttributeValues: {
    ":pk": "USER#123",
    ":min": 100
  }
}
```

#### 3. Scan - Full Table Scan (Use Sparingly)

```javascript
// Basic scan (expensive - avoid for large tables)
{
  TableName: "Users"
}

// Scan with filter
{
  TableName: "Users",
  FilterExpression: "contains(email, :domain)",
  ExpressionAttributeValues: {
    ":domain": "@example.com"
  }
}

// Scan with projection
{
  TableName: "Users",
  ProjectionExpression: "PK, email, name"
}
```

#### 4. BatchGetItem - Multiple Items by Keys

```javascript
// Retrieve up to 100 items by their primary keys
{
  RequestItems: {
    "Users": {
      Keys: [
        { "PK": "USER#123", "SK": "PROFILE" },
        { "PK": "USER#456", "SK": "PROFILE" }
      ],
      ProjectionExpression: "email, name"
    }
  }
}
```

#### 5. Query with GSI (Global Secondary Index)

```javascript
// Query using GSI with different partition key
{
  TableName: "Users",
  IndexName: "EmailIndex",
  KeyConditionExpression: "email = :email",
  ExpressionAttributeValues: {
    ":email": "user@example.com"
  }
}

// Query GSI with sort key
{
  TableName: "Users",
  IndexName: "EmailIndex",
  KeyConditionExpression: "email = :email AND createdAt > :date",
  ExpressionAttributeValues: {
    ":email": "user@example.com",
    ":date": "2024-01-01"
  }
}
```

#### 6. Query with LSI (Local Secondary Index)

```javascript
// Query using LSI (same partition key, different sort key)
{
  TableName: "Users",
  IndexName: "CreatedAtIndex",
  KeyConditionExpression: "PK = :pk AND createdAt > :date",
  ExpressionAttributeValues: {
    ":pk": "USER#123",
    ":date": "2024-01-01"
  }
}
```

### 5. Query Optimization

#### Efficient Query Patterns

```javascript
// ✅ Optimal: Query with partition key
{
  KeyConditionExpression: "PK = :pk",
  ExpressionAttributeValues: { ":pk": "USER#123" }
}

// ❌ Avoid: Full table scan
{
  FilterExpression: "contains(UserName, :name)",
  ExpressionAttributeValues: { ":name": "John" }
}
```

#### Pagination Strategy

```javascript
// First query
{
  KeyConditionExpression: "PK = :pk",
  ExpressionAttributeValues: { ":pk": "USER#123" },
  Limit: 10
}

// Subsequent queries using LastEvaluatedKey
{
  KeyConditionExpression: "PK = :pk",
  ExpressionAttributeValues: { ":pk": "USER#123" },
  ExclusiveStartKey: { "PK": "USER#123", "SK": "last_item_sk" },
  Limit: 10
}
```

### 5. Common Design Patterns

#### User Management

```javascript
// User table structure
{
  "PK": "USER#123",
  "SK": "PROFILE",
  "email": "user@example.com",
  "name": "John Doe"
}

{
  "PK": "USER#123",
  "SK": "ORDER#456",
  "orderDate": "2024-01-15",
  "total": 99.99
}
```

#### Multi-tenant Design

```javascript
// Tenant-aware keys
{
  "PK": "TENANT#123#USER#456",
  "SK": "PROFILE",
  "data": { ... }
}

// Tenant isolation query
{
  KeyConditionExpression: "PK = :pk",
  ExpressionAttributeValues: {
    ":pk": "TENANT#123#USER#"
  }
}
```

### 6. Write Operations

#### PutItem - Create/Replace Item

```javascript
// Create new item
{
  TableName: "Users",
  Item: {
    "PK": "USER#123",
    "SK": "PROFILE",
    "email": "user@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15"
  }
}

// Replace existing item (overwrites all attributes)
{
  TableName: "Users",
  Item: {
    "PK": "USER#123",
    "SK": "PROFILE",
    "email": "newemail@example.com",
    "name": "John Doe Updated"
  }
}
```

#### UpdateItem - Modify Existing Item

```javascript
// Update specific attributes
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  },
  UpdateExpression: "SET email = :email, updatedAt = :updatedAt",
  ExpressionAttributeValues: {
    ":email": "newemail@example.com",
    ":updatedAt": "2024-01-15T10:00:00Z"
  }
}

// Conditional update
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  },
  UpdateExpression: "SET version = version + :inc",
  ConditionExpression: "version = :expectedVersion",
  ExpressionAttributeValues: {
    ":inc": 1,
    ":expectedVersion": 5
  }
}
```

#### DeleteItem - Remove Item

```javascript
// Delete item by primary key
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  }
}

// Conditional delete
{
  TableName: "Users",
  Key: {
    "PK": "USER#123",
    "SK": "PROFILE"
  },
  ConditionExpression: "attribute_exists(PK)"
}
```

#### BatchWriteItem - Bulk Operations

```javascript
// Write up to 25 items in one request
{
  RequestItems: {
    "Users": [
      {
        PutRequest: {
          Item: {
            "PK": "USER#123",
            "SK": "PROFILE",
            "email": "user1@example.com"
          }
        }
      },
      {
        DeleteRequest: {
          Key: {
            "PK": "USER#456",
            "SK": "PROFILE"
          }
        }
      }
    ]
  }
}
```

### 7. Design Checklist

#### Before Creating Tables

- [ ] **Identify all access patterns** - List every query your application needs
- [ ] **Design partition key for even distribution** - Avoid hot partitions
- [ ] **Plan sort keys for efficient queries** - Enable range queries and filtering
- [ ] **Consider GSI requirements** - For alternative access patterns
- [ ] **Validate against performance requirements** - Test with realistic data volumes
- [ ] **Estimate costs** - Calculate read/write capacity needs

#### Query Design Rules

- [ ] **Use partition key in every query** - Never scan unless absolutely necessary
- [ ] **Use complete primary key for GetItem/PutItem/UpdateItem/DeleteItem** - Both PK and SK when applicable
- [ ] **Avoid scan operations** - Only for very small tables (< 1000 items)
- [ ] **Use sort keys for filtering** - Leverage begins_with, between, etc.
- [ ] **Implement proper pagination** - Use LastEvaluatedKey for large result sets
- [ ] **Test with realistic data volumes** - Performance changes with data size
- [ ] **Monitor hot partitions** - Check CloudWatch metrics regularly

#### Common Mistakes to Avoid

- [ ] **Hot partitions** - All data in same partition key
- [ ] **Inefficient sort keys** - Random or non-queryable values
- [ ] **Over-use of GSI** - Each GSI adds cost and complexity
- [ ] **Missing projections** - Retrieving unnecessary data
- [ ] **No pagination** - Large result sets without limits
- [ ] **Scan operations** - On large tables
