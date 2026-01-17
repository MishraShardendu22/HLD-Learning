# Chapter 4: Leaderless Replication & Quorum

## Overview

This chapter covers **leaderless replication** systems where there's no designated leader, and any node can accept reads and writes. We'll explore quorum-based systems, eventual consistency strategies, and real-world use cases from DynamoDB, Cassandra, and Riak.

---

## 1. What is Leaderless Replication?

### Definition

In **leaderless replication** (also called **Dynamo-style**), **all nodes are equal** — any node can handle reads and writes.

### Architecture Comparison

#### Leader-Based
```md
         Leader
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
Follower Follower Follower
(reads)  (reads)  (reads)

Writes → Leader only
```

#### Leaderless
```md
   Node A    Node B    Node C
     ◄──────►◄──────►
     
All nodes accept reads & writes!
Writes go to multiple nodes in parallel
```

### Real-World Analogy: Voting System 🗳️

**Leader-Based = Monarchy**
- One king makes all decisions
- Others just obey

**Leaderless = Democracy**
- Multiple voters decide together
- Majority (quorum) must agree
- No single point of authority

---

## 2. How Leaderless Replication Works

### Write Operation

Client writes to **multiple nodes** in parallel (not just one leader).

```md
Client wants to write: User {id: 1, name: "Alice"}

Step 1: Client sends write to 3 nodes
        ┌──────────┐
        │  Client  │
        └────┬─────┘
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
    Node A  Node B  Node C
    
Step 2: Each node writes data independently
    Node A: ✅ Success
    Node B: ✅ Success
    Node C: ❌ Failed (network timeout)
    
Step 3: Client receives 2/3 confirmations
    Result: Write considered successful ✅
```

### Read Operation

Client reads from **multiple nodes** and picks the latest version.

```md
Client wants to read: User {id: 1}

Step 1: Client sends read to 3 nodes
        ┌──────────┐
        │  Client  │
        └────┬─────┘
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
    Node A  Node B  Node C
    
Step 2: Nodes respond with versions
    Node A: {id: 1, name: "Alice", version: 5}
    Node B: {id: 1, name: "Alice", version: 5}
    Node C: {id: 1, name: "Bob", version: 3} ← Stale!
    
Step 3: Client picks latest version (version 5)
    Result: {id: 1, name: "Alice"} ✅
```

---

## 3. Quorum Reads and Writes

### What is a Quorum?

**Quorum** is the minimum number of nodes that must agree for an operation to succeed.

### Quorum Formula

```md
n = Total number of replicas
w = Write quorum (nodes that must confirm write)
r = Read quorum (nodes that must confirm read)

For strong consistency:
  w + r > n
  
Example: n=3
  w=2, r=2 → 2+2=4 > 3 ✅ Strong consistency
  w=1, r=3 → 1+3=4 > 3 ✅ Strong consistency
  w=1, r=1 → 1+1=2 ≤ 3 ❌ Eventual consistency
```

### Why Does w + r > n Guarantee Consistency?

**Key Insight:** At least one node in the read quorum has the latest write.

```md
Scenario: n=3, w=2, r=2

Write to 2 nodes:
  Nodes A, B: Have version 5 ✅
  Node C: Has version 4 (missed the write)
  
Read from 2 nodes:
  Possible combinations:
    {A, B}: Both have version 5 ✅
    {A, C}: A has version 5 ✅ (pick latest)
    {B, C}: B has version 5 ✅ (pick latest)
    
  In ALL cases, at least one node has latest version!
```

### Real-World Example: DynamoDB

Amazon DynamoDB allows tunable quorum:

```javascript
// Strong consistency: r + w > n
const params = {
  TableName: 'Users',
  Key: { id: '123' },
  ConsistentRead: true  // Forces quorum read
};

// Eventual consistency: r + w ≤ n (faster but stale reads possible)
const params = {
  TableName: 'Users',
  Key: { id: '123' },
  ConsistentRead: false  // May read from any node
};
```

---

## 4. Sloppy Quorum & Hinted Handoff

### What is Sloppy Quorum?

When a node is unavailable, writes go to a **temporary substitute node** outside the normal replica set.

### Regular Quorum (Strict)

```md
User data normally stored on nodes: {A, B, C}

Write when all nodes healthy:
  Write to A, B, C → Success ✅
  
Write when Node C is down:
  Write to A, B → Only 2/3 nodes
  ❌ Quorum not met (w=3), write fails
```

### Sloppy Quorum

```md
User data normally stored on nodes: {A, B, C}

Write when Node C is down:
  Write to A, B, D (D is temporary substitute)
  ✅ Quorum met (3 nodes confirmed)
  
When Node C comes back online:
  Node D sends data to C (hinted handoff)
  Node C catches up ✅
  Node D deletes temporary data
```

### Real-World Analogy: Mail Delivery 📬

```md
Regular Quorum:
  - Mail must be delivered to your home
  - If you're not home → mail not delivered ❌
  
Sloppy Quorum + Hinted Handoff:
  - Mail delivered to neighbor
  - Neighbor hands it to you when you return ✅
```

### DynamoDB Example

```md
Normal replica set for key "user:123": {Node1, Node2, Node3}

Scenario: Node3 is down
  1. Write goes to {Node1, Node2, Node4} ← Node4 is substitute
  2. Node4 stores data temporarily
  3. Node4 remembers: "This belongs to Node3"
  4. When Node3 recovers:
     - Node4 sends data to Node3 (hint: this is yours)
     - Node3 saves data
     - Node4 deletes it
```

### Benefits
- **High availability**: Writes don't fail when nodes are down
- **Fault tolerance**: System continues working

### Drawback
- **Temporary inconsistency**: During hinted handoff, latest data not on correct nodes

---

## 5. Handling Staleness

### Problem: Stale Reads

Even with quorum, nodes can have stale data if they missed writes.

```md
Scenario: n=3, w=2, r=1

Write:
  Nodes A, B: version 5 ✅
  Node C: version 4 (was down during write)
  
Read (r=1):
  Read from Node C
  ❌ Returns version 4 (stale!)
```

### Solution 1: Read Repair

When stale data is detected during a read, update the stale node.

```md
Client reads from 3 nodes:
  Node A: version 5
  Node B: version 5
  Node C: version 4 ← Stale!
  
Client detects staleness:
  1. Return version 5 to user
  2. Send version 5 to Node C (background repair)
  3. Node C updates: version 4 → 5 ✅
```

**Implementation (Cassandra):**
```java
// Cassandra automatically does read repair
SELECT * FROM users WHERE id = 123;

// Behind the scenes:
// 1. Query sent to multiple replicas
// 2. Compare versions
// 3. If mismatch, update stale replicas
```

### Solution 2: Anti-Entropy (Background Process)

Background process continuously syncs data between nodes.

```md
Anti-Entropy Daemon:
  while (true) {
    1. Pick two random nodes
    2. Compare their data (Merkle trees)
    3. Identify differences
    4. Sync missing/stale data
    5. Sleep for interval
  }
```

**Merkle Tree for Efficient Comparison:**

```md
Instead of comparing every key:
  - Build hash tree of data
  - Compare tree hashes
  - Only sync differing branches
  
Example:
  Node A tree hash: 0xABCD1234
  Node B tree hash: 0xABCD5678
  
  Only the differing branch needs sync (efficient!)
```

**Real-World Example: Cassandra**

Cassandra runs `nodetool repair` periodically:

```bash
# Manually trigger anti-entropy
nodetool repair users

# Cassandra compares data across replicas and fixes inconsistencies
```

---

## 6. Read-Write Conflicts in Leaderless Systems

### Scenario: Concurrent Writes

Two clients write to same key simultaneously.

```md
T=0:
  Client 1: Write {id: 1, name: "Alice"} to {Node A, Node B}
  Client 2: Write {id: 1, name: "Bob"} to {Node B, Node C}
  
T=1:
  Node A: {id: 1, name: "Alice"}
  Node B: {id: 1, name: ???} ← Received both writes!
  Node C: {id: 1, name: "Bob"}
```

### Conflict Resolution: Last Write Wins (LWW)

Use timestamp to determine winner.

```md
Client 1: {id: 1, name: "Alice", timestamp: 100}
Client 2: {id: 1, name: "Bob", timestamp: 101}

Node B receives both:
  - Compare timestamps: 101 > 100
  - Keep "Bob" (latest)
  
Result: {id: 1, name: "Bob"} ✅
```

**DynamoDB Example:**

```javascript
// DynamoDB uses LWW by default
await dynamodb.put({
  TableName: 'Users',
  Item: {
    id: '1',
    name: 'Alice',
    updatedAt: Date.now()  // Timestamp
  }
});
```

**Problem with LWW:**
- Clock skew can cause issues
- Data loss if clocks are wrong

```md
Server A clock: 10:00:00
Server B clock: 09:59:00 (1 minute behind)

Client writes "Alice" to A at 10:00:00
Client writes "Bob" to B at 09:59:00 (clock says)

LWW picks "Alice" (timestamp 10:00:00 > 09:59:00)
But "Bob" was actually newer! ❌ Data loss
```

---

## 7. Node Failures & Recovery

### Problem: Node Down During Write

```md
Write to {A, B, C} with w=2:
  Node A: ✅ Success
  Node B: ❌ Down
  Node C: ✅ Success
  
Result: 2/3 nodes confirmed → Write successful ✅
But Node B has stale data
```

### Solution: Hinted Handoff (Covered Earlier)

```md
1. Node B comes back online
2. Receives hint from substitute node
3. Catches up with missed writes ✅
```

### Solution: Anti-Entropy Repair

```md
Anti-entropy process:
  1. Compares Node B with A, C
  2. Detects B is behind
  3. Syncs missing data to B
```

---

## 8. Tombstones (Handling Deletes)

### Problem: Zombie Data

When you delete data in leaderless system, deleted data can "come back to life."

```md
Scenario:
  T=0: Write {id: 1, name: "Alice"} to {A, B, C}
  T=1: Delete {id: 1} to {A, B} (C is down)
  
  Node A: (deleted)
  Node B: (deleted)
  Node C: {id: 1, name: "Alice"} ← Stale
  
  T=2: Node C comes back online
  T=3: Anti-entropy runs
  
  Problem: C thinks it has data, spreads it to A, B
  Result: {id: 1, name: "Alice"} reappears! 🧟 Zombie data
```

### Solution: Tombstone Markers

Instead of deleting data, mark it as deleted with a **tombstone**.

```md
T=1: Delete {id: 1}
  Node A: {id: 1, name: "Alice", deleted: true, deleted_at: 100}
  Node B: {id: 1, name: "Alice", deleted: true, deleted_at: 100}
  Node C: {id: 1, name: "Alice", deleted: false}
  
T=2: Node C comes back
T=3: Anti-entropy compares:
  Node A: deleted=true, deleted_at=100
  Node C: deleted=false
  
  Resolution: A's tombstone wins (delete is newer)
  Node C: {id: 1, deleted: true} ✅
```

**Cassandra Example:**

```sql
-- Delete row
DELETE FROM users WHERE id = 1;

-- Behind the scenes:
-- Cassandra writes tombstone:
-- {id: 1, tombstone: true, deleted_at: 1674000000}

-- Tombstones are eventually garbage collected (configurable TTL)
```

---

## 9. Monitoring Staleness in Leaderless Systems

### Challenge

In leaderless systems, there's no single "replication lag" metric because there's no leader.

### Metrics to Monitor

#### 1. **Read Repair Frequency**

How often do reads trigger repairs?

```md
High read repair rate → Many stale replicas
```

#### 2. **Anti-Entropy Lag**

How long does anti-entropy take to converge?

```md
Fast convergence: Minutes
Slow convergence: Hours
```

#### 3. **Hinted Handoff Queue Size**

How many hints are waiting to be delivered?

```md
Large queue → Nodes frequently down
```

### Real-World Monitoring: Cassandra

```bash
# Check repair status
nodetool compactionstats

# Check hinted handoff status
nodetool statushandoff

# Monitor metrics
nodetool proxyhistograms
```

---

## 10. Real-World Use Cases

### Use Case 1: Amazon DynamoDB

**Scenario:** E-commerce shopping cart.

```md
Requirements:
  - High availability (can't lose customer's cart)
  - Fast writes (low latency)
  - Tolerate temporary inconsistency
  
Solution: DynamoDB with sloppy quorum
  - n=3, w=2, r=2
  - Sloppy quorum for availability
  - Eventually consistent reads for speed
  
Workflow:
  1. User adds item → Write to 2 nodes (w=2)
  2. User views cart → Read from 2 nodes (r=2)
  3. If nodes down, use sloppy quorum
  4. Anti-entropy ensures consistency
```

### Use Case 2: Cassandra (Messaging App)

**Scenario:** Store chat messages.

```md
Requirements:
  - Scale to billions of messages
  - Always available (global users)
  - Messages eventually consistent
  
Solution: Cassandra leaderless replication
  - Multi-datacenter deployment
  - Tunable consistency per query
  - Anti-entropy keeps nodes in sync
  
Workflow:
  1. User sends message → Write to multiple nodes
  2. Recipient reads → Read repair ensures latest
  3. Background anti-entropy syncs missed messages
```

### Use Case 3: Riak (IoT Sensor Data)

**Scenario:** Store sensor readings from millions of devices.

```md
Requirements:
  - Handle device failures
  - High write throughput
  - Tolerate data loss
  
Solution: Riak with LWW
  - n=3, w=1, r=1 (optimize for speed)
  - LWW for conflict resolution
  - Accept eventual consistency
  
Workflow:
  1. Sensor writes reading → 1 node confirms (fast)
  2. Dashboard reads → 1 node responds (fast)
  3. Anti-entropy eventually syncs all nodes
```

---

## 11. Pros and Cons

### Pros ✅

1. **No single point of failure** (no leader bottleneck)
2. **High availability** (works even when many nodes down)
3. **Scalable writes** (any node can handle writes)
4. **Tunable consistency** (choose speed vs consistency)

### Cons ❌

1. **Complex conflict resolution** (concurrent writes need handling)
2. **Eventual consistency** (stale reads possible)
3. **Read/write amplification** (need multiple round trips)
4. **Monitoring complexity** (no single lag metric)

---

## 12. Key Takeaways

1. **Leaderless = All nodes are equal**, any can handle reads/writes
2. **Quorum (w + r > n)** ensures strong consistency
3. **Sloppy quorum** improves availability but adds temporary inconsistency
4. **Read repair & anti-entropy** fix stale data
5. **LWW** resolves conflicts but can lose data
6. **Tombstones** prevent zombie data from deletions
7. **Use cases**: High availability systems (DynamoDB, Cassandra, Riak)

### When to Use Leaderless Replication

| Use Case | Leader-Based | Leaderless |
|----------|--------------|------------|
| Strong consistency required | ✅ | ❌ |
| High availability critical | ❌ | ✅ |
| Tolerate temporary inconsistency | ❌ | ✅ |
| Simple operations | ✅ | ✅ |
| Global scale | ❌ | ✅ |
| Low operational complexity | ✅ | ❌ |
