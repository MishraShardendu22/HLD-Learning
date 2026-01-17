# Chapter 1: Replication Methods in Databases

## Overview

This chapter covers the different **replication methods** used in distributed databases to keep data synchronized across multiple nodes. We'll explore statement-based replication, Write-Ahead Logging (WAL), logical replication, and trigger-based replication with real-world examples.

---

## 1. What is Replication?

### Definition

**Replication** is the process of copying data from one database server (leader/primary) to one or more other servers (followers/replicas) to ensure:

- **High Availability**: System remains operational even if nodes fail
- **Performance**: Read queries can be distributed across replicas
- **Disaster Recovery**: Data is backed up across multiple locations

---

## 2. Statement-Based Replication

### What is Statement-Based Replication?

The leader **logs every SQL statement** (INSERT, UPDATE, DELETE) and sends these statements to followers to execute.

### How It Works

```md
Leader:                    Followers:
1. Execute SQL            1. Receive SQL statement
   INSERT INTO users         INSERT INTO users
   VALUES (1, 'Alice')      VALUES (1, 'Alice')
   
2. Log statement          2. Execute same statement
3. Send to followers      3. Update local data
```

### Real-World Example: MySQL (Old Versions)

Early versions of MySQL used statement-based replication:

```sql
-- Leader executes:
INSERT INTO orders (id, user_id, created_at) 
VALUES (1, 42, NOW());

-- Followers receive and execute the same SQL
```

### Challenges with Statement-Based Replication

#### 1. **Non-Deterministic Functions**

Functions like `NOW()`, `RAND()`, `UUID()` can produce different results on different replicas.

**Problem Example:**
```sql
-- Leader executes at 2:00 PM:
INSERT INTO logs (id, timestamp) VALUES (1, NOW());
-- Result: timestamp = '2024-01-17 14:00:00'

-- Follower executes at 2:01 PM:
-- Result: timestamp = '2024-01-17 14:01:00' ❌ Different!
```

**Solution**: Replace non-deterministic functions with actual values before replication.

#### 2. **Auto-Increment Columns**

If statement uses auto-increment, followers might generate different IDs.

```sql
-- Leader:
INSERT INTO products (name) VALUES ('Laptop');
-- Auto-generated ID: 100

-- Follower:
-- Auto-generated ID might be: 101 ❌
```

#### 3. **Triggers and Stored Procedures**

Side effects from triggers might execute differently on replicas.

```sql
-- If trigger updates another table, timing matters
INSERT INTO users (name) VALUES ('Bob');
-- Trigger: INSERT INTO audit_log (action) VALUES ('User created');
-- Race conditions possible
```

### When to Use Statement-Based Replication

✅ **Good for:**
- Simple CRUD operations with deterministic SQL
- Small transaction sizes
- When bandwidth is limited (SQL statements are small)

❌ **Not good for:**
- Complex queries with non-deterministic functions
- Heavy use of triggers/stored procedures
- Systems requiring exact byte-level consistency

---

## 3. Write-Ahead Logging (WAL) Replication

### What is WAL?

**Write-Ahead Log** is a record of all changes made to the database at the **byte level**. Before any data is written to disk, changes are first written to the WAL.

### How It Works

```md
Leader:                        Followers:
1. Transaction starts          
2. Changes written to WAL      
   [Byte offset 1000: 
    Change page 42, 
    offset 100, 
    bytes: 0x4A 0x6F...]
    
3. Changes applied to DB       
4. WAL shipped to followers → 1. Receive WAL
                              2. Apply byte changes
                              3. Exact replica created
```

### Real-World Example: PostgreSQL

PostgreSQL uses WAL for replication:

```bash
# Leader writes WAL
# /var/lib/postgresql/data/pg_wal/000000010000000000000001

# WAL contains binary changes like:
# Offset 1000: UPDATE page 42, offset 200, 
# OLD: 0x41 0x6C 0x69 0x63 0x65 (Alice)
# NEW: 0x42 0x6F 0x62 0x00 0x00 (Bob)

# Followers stream and apply WAL
```

### Benefits of WAL Replication

#### 1. **Byte-Level Precision**

- Exact copy of data at binary level
- No risk of divergence due to non-deterministic functions

#### 2. **Crash Recovery**

- Database can recover to last consistent state after crash
- WAL acts as a "redo log"

**Example: PostgreSQL Crash Recovery**
```bash
# Database crashes at 2:05 PM
# On restart:
# 1. Load last checkpoint (data at 2:00 PM)
# 2. Replay WAL entries from 2:00 PM to 2:05 PM
# 3. Database fully recovered
```

#### 3. **Physical Replication**

- Very fast because it's copying raw bytes
- No need to parse or interpret SQL

### Challenges with WAL Replication

#### 1. **Storage Engine Dependency**

WAL is specific to storage engine internals. Leader and followers must use:
- Same database version
- Same storage engine
- Same configuration

**Problem Example:**
```md
Leader: PostgreSQL 13 with specific page layout
Follower: PostgreSQL 14 with different page layout
❌ WAL from v13 is incompatible with v14
```

#### 2. **Zero-Downtime Upgrades Are Hard**

Can't upgrade database version without downtime because WAL format changes.

**Real-World Impact:**
- Amazon RDS requires maintenance window for version upgrades
- Can't perform rolling upgrades across replicas

#### 3. **Large WAL Files**

Bulk operations generate huge WAL files.

```sql
-- Insert 1 million rows
INSERT INTO logs SELECT generate_series(1, 1000000);
-- Generates GB of WAL data
```

### When to Use WAL Replication

✅ **Good for:**
- Maximum consistency and precision
- Same database version across all nodes
- High-performance replication needs
- Disaster recovery and point-in-time recovery

❌ **Not good for:**
- Heterogeneous database versions
- Cross-database replication (PostgreSQL → MySQL)
- When you need rolling version upgrades

### Real-World Example: Amazon RDS

Amazon RDS uses WAL for PostgreSQL and MySQL replication:

```md
Primary Database:
  ├─ Writes data
  ├─ Generates WAL
  └─ Ships WAL to standby

Standby Database:
  ├─ Receives WAL
  ├─ Applies changes
  └─ Stays in sync (typically < 1 second lag)
```

---

## 4. Logical Replication

### What is Logical Replication?

Instead of copying byte-level changes, **logical replication** sends **logical changes** (row-level changes) in a more abstract format.

### How It Works

```md
Leader:                          Followers:
1. Execute SQL                   
   UPDATE users 
   SET name = 'Bob' 
   WHERE id = 1

2. Capture logical change        1. Receive change event
   Change Event:                    {
   {                                  table: 'users',
     table: 'users',                  operation: 'UPDATE',
     operation: 'UPDATE',             old: {id: 1, name: 'Alice'},
     old: {id: 1, name: 'Alice'},     new: {id: 1, name: 'Bob'}
     new: {id: 1, name: 'Bob'}      }
   }
                                 2. Apply change to local storage
3. Send to followers             3. Data synchronized
```

### Real-World Example: PostgreSQL Logical Replication

PostgreSQL 10+ supports logical replication:

```sql
-- On Leader (Publisher):
CREATE PUBLICATION my_publication FOR TABLE users, orders;

-- On Follower (Subscriber):
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=leader.example.com dbname=mydb'
PUBLICATION my_publication;

-- Now, changes to users and orders are replicated
```

### Benefits of Logical Replication

#### 1. **Version Independence**

Can replicate between different database versions.

```md
Leader: PostgreSQL 13
Follower: PostgreSQL 15
✅ Works fine because changes are logical, not byte-level
```

#### 2. **Selective Replication**

Can choose specific tables or even specific rows to replicate.

**Example: Replicate Only Active Users**
```sql
-- Replicate only active users
CREATE PUBLICATION active_users_pub 
FOR TABLE users WHERE (status = 'active');
```

#### 3. **Cross-Database Replication**

Can replicate from one database type to another using change data capture (CDC) tools.

**Example: PostgreSQL → Elasticsearch**
```md
PostgreSQL (Source)
  ↓ (CDC via Debezium)
Kafka (Event Stream)
  ↓
Elasticsearch (Destination)
```

#### 4. **Zero-Downtime Upgrades**

Can upgrade database versions with minimal downtime by using logical replication.

**Upgrade Strategy:**
```md
1. Setup new version (v15) as logical replica of old version (v13)
2. Let them sync
3. Switch application traffic to v15
4. Decommission v13
```

### Challenges with Logical Replication

#### 1. **More Overhead**

Need to decode WAL into logical changes, which requires CPU and memory.

```md
WAL:              Logical Change:
0x4A 0x6F 0x68 → {table: 'users', 
0x6E...          operation: 'UPDATE',
(binary)         new: {name: 'John'}}
                 (parsed, structured)
```

#### 2. **Replication Lag**

Can be slower than physical (WAL) replication due to decoding overhead.

**Benchmark Example:**
```md
Physical Replication Lag: ~10ms
Logical Replication Lag: ~50-100ms
```

#### 3. **Complex Conflict Resolution**

If using multi-master with logical replication, conflicts require custom resolution logic.

### When to Use Logical Replication

✅ **Good for:**
- Heterogeneous database versions
- Selective table replication
- Zero-downtime upgrades
- Change Data Capture (CDC) pipelines
- Data integration scenarios

❌ **Not good for:**
- Ultra-low latency requirements
- Very high write throughput (overhead is costly)
- When physical replication is sufficient

### Real-World Example: Netflix

Netflix uses logical replication (via Debezium and Kafka) to stream changes from MySQL to various downstream systems:

```md
MySQL Database
  ↓ (Debezium CDC)
Kafka Topic
  ├→ Elasticsearch (for search)
  ├→ Redis (for caching)
  └→ Data Warehouse (for analytics)
```

---

## 5. Trigger-Based Replication

### What is Trigger-Based Replication?

Uses **database triggers** to capture changes and custom application code to replicate data.

### How It Works

```sql
-- Create a trigger on source database
CREATE TRIGGER replicate_users_insert
AFTER INSERT ON users
FOR EACH ROW
BEGIN
  -- Insert change into replication queue
  INSERT INTO replication_queue (
    table_name, 
    operation, 
    data
  ) VALUES (
    'users', 
    'INSERT', 
    JSON_OBJECT('id', NEW.id, 'name', NEW.name)
  );
END;

-- Application reads replication_queue and sends to followers
```

### Real-World Example: Custom Replication System

```md
Source Database:
  ├─ Trigger captures INSERT/UPDATE/DELETE
  └─ Writes to replication_queue table

Application Worker:
  ├─ Polls replication_queue
  ├─ Reads change events
  └─ Sends to target databases

Target Databases:
  ├─ Receive changes via API
  └─ Apply changes
```

### Benefits of Trigger-Based Replication

#### 1. **Full Control**

Can implement custom business logic during replication.

**Example: Data Transformation**
```sql
-- Transform data before replicating
CREATE TRIGGER replicate_orders
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  -- Mask sensitive data before replication
  INSERT INTO replication_queue (data)
  VALUES (
    JSON_OBJECT(
      'order_id', NEW.id,
      'amount', NEW.amount,
      'customer_id', CONCAT('CUST_', MD5(NEW.customer_id))
    )
  );
END;
```

#### 2. **Cross-Database Replication**

Can replicate between completely different database systems.

```md
Oracle → MySQL
SQL Server → PostgreSQL
```

#### 3. **Flexible Routing**

Can route different data to different destinations.

**Example: Route by Region**
```sql
IF NEW.region = 'US' THEN
  -- Send to US database
ELSEIF NEW.region = 'EU' THEN
  -- Send to EU database
END IF;
```

### Challenges with Trigger-Based Replication

#### 1. **Performance Overhead**

Triggers execute on every write operation, adding latency.

```md
Without Trigger:  10ms per INSERT
With Trigger:     25ms per INSERT (150% overhead)
```

#### 2. **Complexity**

Need to maintain custom replication logic and handle errors.

#### 3. **Limited by Application Logic**

If trigger fails or application crashes, replication stops.

**Example Problem:**
```md
1. Trigger writes to replication_queue ✅
2. Application crashes ❌
3. Changes stuck in queue
4. Manual intervention needed
```

### When to Use Trigger-Based Replication

✅ **Good for:**
- Custom data transformations
- Cross-database replication with different schemas
- Hybrid systems with complex routing logic
- When built-in replication doesn't meet requirements

❌ **Not good for:**
- High-performance, high-throughput systems
- When built-in replication is available and sufficient
- Systems requiring automatic failover

### Real-World Example: Oracle GoldenGate

Oracle GoldenGate uses a trigger-like mechanism for heterogeneous replication:

```md
Source: Oracle Database
  ↓ (GoldenGate Capture)
Trail Files (log of changes)
  ↓ (GoldenGate Replicat)
Target: SQL Server, PostgreSQL, Hadoop, etc.
```

---

## 6. Comparison of Replication Methods

| Method | Precision | Flexibility | Performance | Version Independence | Use Case |
|--------|-----------|-------------|-------------|----------------------|----------|
| **Statement-Based** | Low | Low | High | Yes | Simple CRUD, bandwidth-limited |
| **WAL (Physical)** | Very High | Low | Very High | No | Same version, max consistency |
| **Logical** | High | High | Medium | Yes | Upgrades, CDC, selective replication |
| **Trigger-Based** | Medium | Very High | Low | Yes | Custom logic, cross-DB replication |

---

## 7. Real-World Use Cases

### Use Case 1: E-commerce (Amazon-like)

```md
Requirement: Replicate product catalog globally

Solution: Logical Replication
- Replicate product data to regional databases
- Selective replication (only relevant products per region)
- Transform currency during replication

Products Table (US):
  ↓ (Logical Replication with transformation)
Products Table (EU):
  - Prices converted to EUR
  - Compliance fields added
```

### Use Case 2: Financial System (Banking)

```md
Requirement: Exact consistency, disaster recovery

Solution: WAL (Physical) Replication
- Byte-perfect replicas
- Point-in-time recovery
- Synchronous replication for critical data

Primary Bank DB → WAL → Standby DB (< 1ms lag)
```

### Use Case 3: Analytics Pipeline (Netflix-like)

```md
Requirement: Stream changes to data warehouse

Solution: Logical Replication (CDC)
- Capture changes from OLTP database
- Stream to Kafka
- Load into data warehouse for analytics

MySQL → Debezium → Kafka → Snowflake
```

---

## Key Takeaways

1. **Statement-Based**: Simple but has limitations with non-deterministic functions
2. **WAL (Physical)**: Byte-perfect, very fast, but version-dependent
3. **Logical**: Flexible, version-independent, enables CDC and upgrades
4. **Trigger-Based**: Maximum control, but complex and slower

Choose based on:
- Consistency requirements
- Version upgrade needs
- Performance constraints
- Cross-database requirements
- Customization needs
