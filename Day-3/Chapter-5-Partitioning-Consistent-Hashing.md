# Chapter 5: Partitioning & Consistent Hashing

## Overview

This chapter covers **partitioning** (also called **sharding**) — splitting data across multiple nodes to scale beyond a single machine's capacity. We'll explore partitioning strategies, consistent hashing, rebalancing, and real-world implementations.

---

## 1. What is Partitioning?

### Definition

**Partitioning (Sharding)** is the process of splitting a large dataset into smaller chunks (**partitions** or **shards**) distributed across multiple nodes.

### Why Partition?

#### Problem: Single Node Limits

```md
Single Database Server:
  - Storage: 1TB disk (can't store 10TB dataset)
  - Memory: 64GB RAM (can't cache all data)
  - CPU: 16 cores (can't handle 1M QPS)
  
→ Vertical scaling has limits 💥
```

#### Solution: Horizontal Partitioning

```md
Partition data across 10 nodes:
  - Storage: 10 × 1TB = 10TB ✅
  - Memory: 10 × 64GB = 640GB ✅
  - CPU: 10 × 16 = 160 cores ✅
  - Throughput: 10 × 100K = 1M QPS ✅
```

### Real-World Analogy: Library 📚

```md
Small Library (No Partitioning):
  - All books on one shelf
  - Hard to find books
  - Limited capacity
  
Large Library (Partitioned):
  - Fiction section
  - Non-fiction section
  - Reference section
  - Each section in different room
  - Easy to find books
  - Unlimited capacity ✅
```

---

## 2. Types of Partitioning

### 1. Horizontal Partitioning (Sharding)

Split **rows** across nodes based on some key.

```md
Users Table (1M rows):

Partition 1 (Node A):
  id: 1-333,333
  
Partition 2 (Node B):
  id: 333,334-666,666
  
Partition 3 (Node C):
  id: 666,667-1,000,000
```

**Use When:**
- Dataset too large for one node
- Need to scale horizontally

---

### 2. Vertical Partitioning

Split **columns** across nodes.

```md
Users Table:

Partition 1 (Node A):
  id, name, email
  
Partition 2 (Node B):
  id, address, phone, bio
```

**Use When:**
- Different columns have different access patterns
- Some columns are rarely accessed

**Real-World Example: Facebook**
```md
User Profile:
  - Hot data (name, photo): Fast SSD
  - Cold data (old posts): Slower disk
```

---

### 3. Range Partitioning

Partition by **key range**.

```md
Sensor Data by Timestamp:

Partition 1: Jan 2024 - Mar 2024
Partition 2: Apr 2024 - Jun 2024
Partition 3: Jul 2024 - Sep 2024
```

**Pros:**
- Range queries are efficient
- Time-series data fits naturally

**Cons:**
- Can create **hotspots** if data not evenly distributed

**Example: Hotspot Problem**
```md
E-commerce Orders:
  - Black Friday: Millions of orders
  - Regular day: Thousands of orders
  
Range partition by date:
  - Partition for Black Friday: 🔥 Overloaded
  - Partition for regular day: 😴 Underutilized
```

---

### 4. Hash Partitioning

Use **hash function** to determine partition.

```md
Hash Function: hash(key) % num_partitions

Example:
  hash("user_123") % 3 = 1 → Partition 1
  hash("user_456") % 3 = 2 → Partition 2
  hash("user_789") % 3 = 0 → Partition 0
```

**Pros:**
- Even distribution (no hotspots)
- Simple to implement

**Cons:**
- Range queries inefficient (data scattered)
- Adding/removing nodes requires repartitioning

---

### 5. List Partitioning

Partition by **specific values**.

```md
Users by Region:

Partition 1 (US): region IN ('US', 'Canada')
Partition 2 (EU): region IN ('UK', 'Germany', 'France')
Partition 3 (Asia): region IN ('India', 'China', 'Japan')
```

**Use When:**
- Logical grouping by category
- Data sovereignty requirements (GDPR)

**Real-World Example: Amazon**
```md
Products by Category:
  - Electronics partition
  - Books partition
  - Clothing partition
```

---

## 3. Skew and Hotspots

### What is Skew?

**Skew** occurs when data is unevenly distributed across partitions.

```md
Ideal (No Skew):
  Partition 1: 10GB
  Partition 2: 10GB
  Partition 3: 10GB
  
Skewed:
  Partition 1: 25GB 🔥
  Partition 2: 3GB
  Partition 3: 2GB
```

### What is a Hotspot?

**Hotspot** occurs when one partition receives disproportionate traffic.

```md
Celebrity on Twitter:
  - Taylor Swift posts tweet
  - Millions of users like/comment
  - All traffic hits partition containing her posts 🔥
  
Normal user:
  - Posts tweet
  - Few likes/comments
  - Low traffic on partition 😴
```

### Causes of Hotspots

#### 1. **Popular Keys (Celebrity Problem)**

```md
Hash partition by user_id:
  - Taylor Swift (100M followers): 🔥
  - Regular user (100 followers): 😴
```

#### 2. **Temporal Hotspots**

```md
News website:
  - Breaking news article posted
  - Everyone reads that article
  - Partition with that article: 🔥
```

#### 3. **Poor Hash Function**

```md
Bad hash: hash(key) = key[0].charCodeAt()
  - All keys starting with 'A' go to same partition
  - Skewed distribution
```

---

## 4. Consistent Hashing

### Problem with Simple Hashing

Traditional hashing: `partition = hash(key) % N`

```md
Setup: 3 nodes
  hash("user_123") % 3 = 1 → Node 1
  hash("user_456") % 3 = 2 → Node 2
  
Add 1 node (now 4 nodes):
  hash("user_123") % 4 = 3 → Node 3 ❌ Changed!
  hash("user_456") % 4 = 0 → Node 0 ❌ Changed!
  
Problem: Adding/removing nodes reshuffles ALL data 💥
```

### Solution: Consistent Hashing

**Key Idea:** Map both nodes and keys to a circle (0-360°), assign keys to nearest clockwise node.

```md
Hash Ring (Circle):

       0° / 360°
          │
    270°──┼──90°
          │
        180°

Nodes:
  Node A: 45°
  Node B: 135°
  Node C: 270°
  
Keys:
  Key "user_123": hash → 60° → Belongs to Node B (next clockwise)
  Key "user_456": hash → 200° → Belongs to Node C
  Key "user_789": hash → 300° → Belongs to Node A
```

### Adding a Node

```md
Add Node D at 180°:

Before:
  Keys 135°-270°: Node C
  
After:
  Keys 135°-180°: Node D ← Only this range moves!
  Keys 180°-270°: Node C ← Unchanged
  
Result: Only ~25% of data moves (not 100%) ✅
```

### Real-World Analogy: Pizza Slices 🍕

```md
Traditional Hashing:
  - 3 friends share pizza (3 slices each)
  - 4th friend joins
  - Must re-cut entire pizza (3 slices → 2.25 slices each)
  
Consistent Hashing:
  - Pizza is a circle
  - Each friend sits at position on circle
  - Take pizza from current position to next friend
  - 4th friend joins: Only pizza in their "slice" moves
```

---

## 5. Virtual Nodes

### Problem: Uneven Distribution

With few physical nodes, data might not distribute evenly.

```md
3 Physical Nodes on hash ring:
  Node A: 0°
  Node B: 120°
  Node C: 240°
  
If keys cluster around 50°-119°:
  Node B gets most data 🔥
```

### Solution: Virtual Nodes

Each physical node represents **multiple virtual nodes** on the ring.

```md
Physical Node A → Virtual Nodes: A1, A2, A3
Physical Node B → Virtual Nodes: B1, B2, B3
Physical Node C → Virtual Nodes: C1, C2, C3

Hash Ring:
  0°: A1
  40°: B1
  80°: C1
  120°: A2
  160°: B2
  200°: C2
  240°: A3
  280°: B3
  320°: C3
  
More even distribution! ✅
```

### Benefits

1. **Load balancing**: More virtual nodes = better distribution
2. **Flexibility**: Can weight powerful nodes with more virtual nodes

```md
Node A (powerful server): 100 virtual nodes
Node B (weak server): 50 virtual nodes
→ Node A handles 2× load ✅
```

### Real-World Example: Cassandra

Cassandra uses 256 virtual nodes per physical node by default.

```yaml
# cassandra.yaml
num_tokens: 256  # Virtual nodes per physical node
```

---

## 6. Java Implementation of Consistent Hashing

```java
import java.util.*;
import java.security.MessageDigest;

public class ConsistentHash<T> {
    private final TreeMap<Long, T> ring = new TreeMap<>();
    private final int virtualNodes;
    
    public ConsistentHash(int virtualNodes) {
        this.virtualNodes = virtualNodes;
    }
    
    // Add a node
    public void addNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            String virtualKey = node.toString() + "#" + i;
            long hash = hash(virtualKey);
            ring.put(hash, node);
        }
    }
    
    // Remove a node
    public void removeNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            String virtualKey = node.toString() + "#" + i;
            long hash = hash(virtualKey);
            ring.remove(hash);
        }
    }
    
    // Get node for a key
    public T getNode(String key) {
        if (ring.isEmpty()) return null;
        
        long hash = hash(key);
        
        // Find next clockwise node
        Map.Entry<Long, T> entry = ring.ceilingEntry(hash);
        if (entry == null) {
            // Wrap around to first node
            entry = ring.firstEntry();
        }
        
        return entry.getValue();
    }
    
    // Hash function (MD5)
    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            long hash = 0;
            for (int i = 0; i < 8; i++) {
                hash = (hash << 8) | (digest[i] & 0xFF);
            }
            return hash;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

// Usage:
public class Main {
    public static void main(String[] args) {
        ConsistentHash<String> ch = new ConsistentHash<>(3); // 3 virtual nodes
        
        // Add servers
        ch.addNode("Server-A");
        ch.addNode("Server-B");
        ch.addNode("Server-C");
        
        // Get server for keys
        System.out.println("user_123 -> " + ch.getNode("user_123"));
        System.out.println("user_456 -> " + ch.getNode("user_456"));
        
        // Add new server
        ch.addNode("Server-D");
        
        // Only some keys move to Server-D
        System.out.println("user_123 -> " + ch.getNode("user_123"));
    }
}
```

---

## 7. Systems Using Consistent Hashing

### 1. **Cassandra**

```md
Token Ring:
  - Each node owns a range of tokens (hash values)
  - Data distributed by partition key hash
  - Virtual nodes for balance
  
Example:
  Node A: tokens 0-100
  Node B: tokens 101-200
  Node C: tokens 201-255
```

### 2. **Memcached (with Client Library)**

```python
from consistent_hash import ConsistentHash

ch = ConsistentHash()
ch.add_node("memcached-1:11211")
ch.add_node("memcached-2:11211")
ch.add_node("memcached-3:11211")

# Get server for key
server = ch.get_node("user_session_123")
# Connect to that server and store/retrieve data
```

### 3. **CDN (Akamai, Cloudflare)**

```md
Content Delivery:
  - Hash content URL
  - Route to nearest edge server using consistent hashing
  - Adding edge servers doesn't invalidate entire cache
```

### 4. **DynamoDB**

```md
Partition Key:
  - Hash partition key
  - Determine which partition stores data
  - Consistent hashing for scalability
```

---

## 8. Partitioning with Secondary Indexes

### Problem

Partitioning works great for queries by partition key, but what about queries by other fields?

```md
Users table partitioned by user_id:

Query 1: SELECT * FROM users WHERE user_id = 123
  → Easy! Hash user_id, go to partition ✅
  
Query 2: SELECT * FROM users WHERE email = 'alice@example.com'
  → Hard! Must scan ALL partitions ❌
```

### Solution 1: Document Partitioning (Local Index)

Each partition maintains its own index.

```md
Partition 1:
  Users: {id: 1, email: 'alice@example.com'}
  Index: {'alice@example.com' → id: 1}
  
Partition 2:
  Users: {id: 2, email: 'bob@example.com'}
  Index: {'bob@example.com' → id: 2}
  
Query by email:
  → Must query ALL partitions (scatter-gather)
  → Slow for queries ❌
```

**Used by:** MongoDB, Elasticsearch (document-partitioned indexes)

---

### Solution 2: Term Partitioning (Global Index)

Index itself is partitioned separately from data.

```md
Data Partitions:
  Partition 1: {id: 1, email: 'alice@example.com'}
  Partition 2: {id: 2, email: 'bob@example.com'}
  
Index Partitions:
  Index Partition A: {'alice@example.com' → id: 1}
  Index Partition B: {'bob@example.com' → id: 2}
  
Query by email:
  1. Hash('alice@example.com') → Index Partition A
  2. Look up user_id: 1
  3. Hash(1) → Data Partition 1
  4. Retrieve user ✅
```

**Pros:**
- Efficient queries (single partition lookup)

**Cons:**
- Writes slower (must update both data and index partitions)
- Eventual consistency (index might lag)

**Used by:** DynamoDB (GSI - Global Secondary Index), Cassandra (global indexes)

---

## 9. Rebalancing Partitions

### When to Rebalance?

- Add new nodes (scale out)
- Remove nodes (scale down or failure)
- Data growth (one partition too large)

### Strategy 1: Fixed Number of Partitions

Create many partitions upfront, assign to nodes.

```md
Setup: 1000 partitions, 10 nodes
  - Each node gets 100 partitions
  
Add 10 more nodes (now 20 total):
  - Reassign partitions
  - Each node gets 50 partitions
  - Move 500 partitions (50% of data)
```

**Pros:**
- Simple
- No rehashing

**Cons:**
- Partition size grows with data (eventually too large)

**Used by:** Riak, Elasticsearch, Couchbase

---

### Strategy 2: Dynamic Partitioning

Split partitions when they grow too large.

```md
Partition A grows to 10GB:
  → Split into A1 (5GB) and A2 (5GB)
  → Move A2 to different node
  
Partition B shrinks to 1GB:
  → Merge with neighbor partition
```

**Pros:**
- Adapts to data growth
- Efficient use of resources

**Cons:**
- Complex to implement

**Used by:** HBase, MongoDB

---

### Strategy 3: Consistent Hashing (Covered Earlier)

Add/remove virtual nodes.

**Used by:** Cassandra, DynamoDB

---

## 10. Real-World Use Cases

### Use Case 1: MongoDB Sharding

**Scenario:** Social media posts database.

```md
Collection: posts
Shard Key: user_id

Setup:
  Shard 1: user_id 0-1000000
  Shard 2: user_id 1000001-2000000
  Shard 3: user_id 2000001-3000000
  
Query:
  Find posts by user_id 1500000:
    → Hash 1500000 → Shard 2
    → Query only Shard 2 ✅
```

---

### Use Case 2: Cassandra Time-Series Data

**Scenario:** IoT sensor readings.

```md
Table: sensor_data
Partition Key: (sensor_id, date)

Setup:
  - Data partitioned by sensor_id and date
  - Each day's data for each sensor in separate partition
  
Query:
  Get sensor_123 data for 2024-01-17:
    → Hash (sensor_123, 2024-01-17) → Partition
    → Single partition read ✅
```

---

### Use Case 3: DynamoDB E-commerce Orders

**Scenario:** Order history.

```md
Table: orders
Partition Key: user_id
Sort Key: order_date

Setup:
  - Orders partitioned by user_id
  - Within partition, sorted by order_date
  
Query 1: Get user's recent orders
  → Hash user_id → Partition
  → Range query on order_date ✅
  
Query 2: Get order by order_id
  → Create GSI with order_id as partition key
  → Hash order_id → Index partition → Retrieve ✅
```

---

## 11. Pros and Cons of Partitioning

### Pros ✅

1. **Scalability**: Handle datasets larger than single node
2. **Performance**: Parallel queries across partitions
3. **Fault isolation**: One partition failure doesn't affect others

### Cons ❌

1. **Complexity**: More moving parts
2. **Cross-partition queries**: Slow (scatter-gather)
3. **Rebalancing overhead**: Data movement during scaling
4. **Transaction complexity**: Cross-partition transactions are hard

---

## 12. Key Takeaways

1. **Partitioning = Splitting data across nodes** for scale
2. **Types**: Horizontal (rows), vertical (columns), range, hash, list
3. **Hotspots**: Uneven load due to skew or popular keys
4. **Consistent hashing**: Minimize data movement when adding/removing nodes
5. **Virtual nodes**: Better load distribution
6. **Secondary indexes**: Document (local) vs term (global) partitioning
7. **Rebalancing**: Fixed partitions, dynamic splitting, or consistent hashing
8. **Use cases**: MongoDB, Cassandra, DynamoDB for large-scale systems

### Choosing a Partitioning Strategy

| Requirement | Strategy |
|-------------|----------|
| Even distribution | Hash partitioning |
| Range queries | Range partitioning |
| Logical grouping | List partitioning |
| Easy scaling | Consistent hashing |
| Time-series data | Range by timestamp |
| Global secondary indexes | Term partitioning |
