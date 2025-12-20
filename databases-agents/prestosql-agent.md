# PrestoSQL Agent

*Specialized agent for PrestoSQL table design and query optimization*

## ⚠️ Important Concepts

### PrestoSQL Architecture

**Key Components:**

- **Coordinator** - Manages query planning and execution
- **Workers** - Execute query fragments in parallel
- **Connectors** - Interface with data sources (40+ available)
- **Catalogs** - Logical grouping of schemas and tables

```sql
-- Example catalog configuration
CREATE CATALOG hive WITH (
    type = 'hive',
    hive.metastore.uri = 'thrift://localhost:9083'
);

-- Create schema within catalog
CREATE SCHEMA hive.default;
```

## Core Design Principles

### 1. Table Design Strategy

#### Catalog and Schema Organization

```sql
-- Best practice: Use descriptive catalog names
CREATE CATALOG analytics WITH (
    type = 'hive',
    hive.metastore.uri = 'thrift://metastore:9083'
);

-- Organize schemas by data domain
CREATE SCHEMA analytics.users;
CREATE SCHEMA analytics.orders;
CREATE SCHEMA analytics.products;
```

#### Table Creation Patterns

```sql
-- External table (data stored externally)
CREATE TABLE hive.default.users (
    user_id BIGINT,
    email VARCHAR,
    created_at TIMESTAMP,
    status VARCHAR
)
WITH (
    format = 'PARQUET',
    external_location = 's3://bucket/users/'
);

-- Internal table (managed by Presto)
CREATE TABLE hive.default.orders (
    order_id BIGINT,
    user_id BIGINT,
    total_amount DECIMAL(10,2),
    order_date DATE
)
WITH (
    format = 'ORC',
    partitioned_by = ARRAY['order_date']
);
```

### 2. Query Optimization Strategy

#### Performance Best Practices

**✅ Efficient Patterns:**

```sql
-- Use partition pruning
SELECT * FROM orders
WHERE order_date >= DATE '2024-01-01'
  AND order_date < DATE '2024-02-01';

-- Leverage columnar formats
SELECT user_id, COUNT(*)
FROM users
WHERE status = 'active'
GROUP BY user_id;

-- Use appropriate data types
SELECT CAST(amount AS DECIMAL(10,2)) as total
FROM transactions;
```

**❌ Avoid Patterns:**

```sql
-- Avoid SELECT * on large tables
SELECT * FROM large_table;

-- Avoid complex functions in WHERE
SELECT * FROM users
WHERE UPPER(email) LIKE '%@EXAMPLE.COM';

-- Avoid subqueries when JOIN is better
SELECT * FROM users u
WHERE u.user_id IN (SELECT user_id FROM orders);
```

### 3. PrestoSQL Query Types

#### 1. Basic SELECT Queries

```sql
-- Simple projection
SELECT user_id, email, created_at
FROM users
WHERE status = 'active'
LIMIT 100;

-- Aggregation with GROUP BY
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(total_amount) as total_spent
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30' DAY
GROUP BY user_id
HAVING COUNT(*) > 5;
```

#### 2. JOIN Operations

```sql
-- INNER JOIN
SELECT
    u.user_id,
    u.email,
    o.order_id,
    o.total_amount
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
WHERE o.order_date >= DATE '2024-01-01';

-- LEFT JOIN with aggregation
SELECT
    u.user_id,
    u.email,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.email;
```

#### 3. Window Functions

```sql
-- Ranking within partitions
SELECT
    user_id,
    order_id,
    total_amount,
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY total_amount DESC
    ) as rank_by_amount
FROM orders;

-- Running totals
SELECT
    user_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY user_id
        ORDER BY order_date
        ROWS UNBOUNDED PRECEDING
    ) as running_total
FROM orders;
```

#### 4. Complex Aggregations

```sql
-- Multiple aggregations
SELECT
    DATE_TRUNC('month', order_date) as month,
    COUNT(DISTINCT user_id) as unique_users,
    COUNT(*) as total_orders,
    AVG(total_amount) as avg_order_value,
    PERCENTILE(total_amount, 0.5) as median_order_value
FROM orders
WHERE order_date >= DATE '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

#### 5. Subqueries and CTEs

```sql
-- Common Table Expression (CTE)
WITH user_stats AS (
    SELECT
        user_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '90' DAY
    GROUP BY user_id
    HAVING COUNT(*) >= 3
),
top_users AS (
    SELECT user_id
    FROM user_stats
    WHERE total_spent > 1000
)
SELECT
    u.user_id,
    u.email,
    us.order_count,
    us.total_spent
FROM users u
INNER JOIN user_stats us ON u.user_id = us.user_id
INNER JOIN top_users tu ON u.user_id = tu.user_id;
```

#### 6. JSON and Array Operations

```sql
-- JSON extraction
SELECT
    user_id,
    JSON_EXTRACT_SCALAR(metadata, '$.preferences.theme') as theme,
    JSON_EXTRACT_SCALAR(metadata, '$.settings.notifications') as notifications
FROM users
WHERE JSON_EXTRACT_SCALAR(metadata, '$.status') = 'premium';

-- Array operations
SELECT
    user_id,
    ARRAY_AGG(order_id ORDER BY order_date) as order_sequence,
    ARRAY_MAX(ARRAY_AGG(total_amount)) as max_order
FROM orders
GROUP BY user_id;
```

### 4. Cross-Catalog Queries

#### Federated Queries

```sql
-- Query across multiple data sources
SELECT
    u.user_id,
    u.email,
    o.order_count,
    p.product_count
FROM mysql.users u
LEFT JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM hive.analytics.orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '30' DAY
    GROUP BY user_id
) o ON u.user_id = o.user_id
LEFT JOIN (
    SELECT user_id, COUNT(*) as product_count
    FROM postgres.products
    WHERE created_at >= CURRENT_DATE - INTERVAL '30' DAY
    GROUP BY user_id
) p ON u.user_id = p.user_id;
```

### 5. Performance Optimization

#### Query Tuning Techniques

```sql
-- Use EXPLAIN to analyze query plan
EXPLAIN (TYPE LOGICAL)
SELECT user_id, COUNT(*)
FROM orders
WHERE order_date >= DATE '2024-01-01'
GROUP BY user_id;

-- Use EXPLAIN ANALYZE for execution details
EXPLAIN ANALYZE
SELECT user_id, COUNT(*)
FROM orders
WHERE order_date >= DATE '2024-01-01'
GROUP BY user_id;
```

#### Partitioning Strategy

```sql
-- Create partitioned table
CREATE TABLE hive.analytics.orders_partitioned (
    order_id BIGINT,
    user_id BIGINT,
    total_amount DECIMAL(10,2),
    order_date DATE
)
WITH (
    format = 'PARQUET',
    partitioned_by = ARRAY['order_date']
);

-- Query with partition pruning
SELECT COUNT(*)
FROM orders_partitioned
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31';
```

### 6. Common Design Patterns

#### Data Lake Analytics

```sql
-- Analyze data from multiple sources
WITH user_activity AS (
    SELECT
        user_id,
        COUNT(*) as page_views,
        COUNT(DISTINCT session_id) as sessions
    FROM clickstream
    WHERE event_date >= CURRENT_DATE - INTERVAL '7' DAY
    GROUP BY user_id
),
user_orders AS (
    SELECT
        user_id,
        COUNT(*) as orders,
        SUM(total_amount) as revenue
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '7' DAY
    GROUP BY user_id
)
SELECT
    u.user_id,
    u.email,
    COALESCE(ua.page_views, 0) as page_views,
    COALESCE(ua.sessions, 0) as sessions,
    COALESCE(uo.orders, 0) as orders,
    COALESCE(uo.revenue, 0) as revenue
FROM users u
LEFT JOIN user_activity ua ON u.user_id = ua.user_id
LEFT JOIN user_orders uo ON u.user_id = uo.user_id;
```

#### Time-Series Analysis

```sql
-- Daily aggregations with window functions
SELECT
    DATE_TRUNC('day', order_date) as day,
    COUNT(*) as orders,
    SUM(total_amount) as revenue,
    AVG(total_amount) as avg_order_value,
    LAG(COUNT(*)) OVER (ORDER BY DATE_TRUNC('day', order_date)) as prev_day_orders
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30' DAY
GROUP BY DATE_TRUNC('day', order_date)
ORDER BY day;
```

### 7. Design Checklist

#### Before Creating Tables

- [ ] **Choose appropriate catalog** - Based on data source and access patterns
- [ ] **Design schema organization** - Group related tables logically
- [ ] **Select optimal format** - PARQUET for analytics, ORC for Hive
- [ ] **Plan partitioning strategy** - For large tables with time-based access
- [ ] **Consider data types** - Use appropriate types for performance
- [ ] **Estimate storage requirements** - Based on data volume and format

#### Query Design Rules

- [ ] **Use partition pruning** - Always filter on partition columns first
- [ ] **Leverage columnar formats** - Use PARQUET/ORC for analytics
- [ ] **Avoid SELECT *** - Specify only needed columns
- [ ] **Use appropriate JOIN types** - INNER, LEFT, RIGHT based on requirements
- [ ] **Implement proper filtering** - Use WHERE before GROUP BY
- [ ] **Test with EXPLAIN** - Analyze query plans before production
- [ ] **Monitor resource usage** - Check memory and CPU consumption

#### Common Mistakes to Avoid

- [ ] **Cross-catalog joins** - Can be expensive, use carefully
- [ ] **Complex subqueries** - Prefer CTEs for readability
- [ ] **Missing indexes** - Ensure proper indexing on join columns
- [ ] **Large result sets** - Use LIMIT and pagination
- [ ] **Inefficient data types** - Use appropriate types for performance
- [ ] **No partition filtering** - Always filter on partition columns
