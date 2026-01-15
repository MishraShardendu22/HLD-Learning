# Chapter 5: Replication & Distributed Systems

## Overview

This chapter covers **replication** — the practice of keeping copies of data on multiple nodes to improve scalability, availability, and performance. We'll explore leader-based replication, synchronous vs asynchronous replication, failover mechanisms, and how distributed systems handle node failures.

---

## 1. What is Replication?

### Definition

**Replication** means maintaining **multiple copies** of the same data on different nodes (servers/machines).

### Why Replicate Data?

#### 1. **Reduced Latency** (Performance)

- Place replicas geographically closer to users
- **Example**: Netflix has servers in multiple continents so users stream from nearby servers

#### 2. **High Availability** (Reliability)

- If one node fails, others can serve requests
- **Example**: If AWS US-East-1 goes down, switch to US-West-2

#### 3. **Scalability** (Throughput)

- Distribute read requests across multiple replicas
- **Example**: Wikipedia has many read replicas to handle millions of readers

---

### Real-World Analogy: Coffee Shop Franchise ☕

Imagine you own a popular coffee shop called "Code Brew":

- **Single shop (No replication)**:
  - One location, long lines, limited customers
  - If it closes, no one gets coffee
  
- **Multiple shops (Replication)**:
  - Open branches in different neighborhoods
  - Customers go to nearest location (low latency)
  - If one closes, others remain open (high availability)
  - More customers served overall (scalability)

---

## 2. Leader-Based Replication (Master-Slave)

### What is Leader-Based Replication?

In **leader-based replication**, one node is designated as the **leader (master)**, and other nodes are **followers (slaves/replicas)**.

### Architecture

```md
         ┌─────────┐
         │ Leader  │◄─── Writes only
         │ (Master)│
         └────┬────┘
              │
       ┌──────┼──────┐
       │      │      │
       ▼      ▼      ▼
   ┌──────┐┌──────┐┌──────┐
   │Follower││Follower││Follower│
   │(Replica)││(Replica)││(Replica)│◄─── Reads
   └──────┘└──────┘└──────┘
```

### How It Works

#### Step 1: **Writes go to Leader**

- Client sends INSERT/UPDATE/DELETE to leader
- Leader writes to its local storage

#### Step 2: **Leader propagates changes**

- Leader sends changes to all followers (via replication log)

#### Step 3: **Followers apply changes**

- Followers apply changes in the same order
- Eventually, all replicas have the same data

#### Step 4: **Reads can go to any node**

- Read requests can be served by leader or followers
- Distributes load across replicas

---

### Example: Database Replication (PostgreSQL, MySQL)

#### Setup

- **Leader**: `db-master.example.com`
- **Followers**: `db-replica1.example.com`, `db-replica2.example.com`

#### Write Operation

```sql
-- Client sends write to leader
INSERT INTO users (id, name) VALUES (1, 'Alice');

-- Leader writes to disk
-- Leader sends change to replicas
-- Replicas apply change
```

#### Read Operation

```sql
-- Client can read from any replica
SELECT * FROM users WHERE id = 1;
-- Query served by replica1 or replica2 (load balanced)
```

---

## 3. Writes vs Reads in Replication

### Writes (Always to Leader)

- **Why leader only?** To maintain consistency (single source of truth)
- Leader orders all writes sequentially
- Prevents conflicting updates

### Reads (Leader or Followers)

- **Read from followers**: Reduces load on leader
- **Read from leader**: For strong consistency (latest data)

### Read Scaling

- Add more followers to handle more read traffic
- **Example**: E-commerce site with 90% reads, 10% writes → add 5 read replicas

---

## 4. Synchronous Replication

### What is Synchronous Replication?

The leader **waits** for at least one follower to confirm the write before acknowledging to the client.

### How It Works

```md
Client           Leader          Follower 1
  │                │                 │
  │──Write────────▶│                 │
  │                │──Replicate─────▶│
  │                │◄──ACK───────────│
  │◄──Success──────│                 │
  │                │                 │
```

**Timeline**:

1. Client sends write to leader (t=0ms)
2. Leader writes locally (t=1ms)
3. Leader sends to follower (t=2ms)
4. Follower writes and sends ACK (t=10ms)
5. Leader responds to client (t=11ms) ← **Client waits**

---

### Real-World Analogy: Group Chat 💬

#### Synchronous (WhatsApp with "Read Receipts")

- You send a message
- You see "✓✓" only after your friend receives it
- You **know** they have the message

#### Characteristics

- **Guaranteed durability**: If leader crashes after ACK, data is safe on follower
- **Higher latency**: Client waits for follower confirmation
- **Consistency**: Follower is always up-to-date (within network delay)

---

### Pros and Cons

#### Pros

- **Strong durability**: Data on multiple nodes before acknowledging
- **No data loss**: If leader fails, follower has latest data
- **Consistency**: Follower is very close to leader state

#### Cons

- **Higher latency**: Every write waits for follower
- **Availability risk**: If follower is slow/down, writes block
- **Lower throughput**: Limited by slowest follower

---

## 5. Asynchronous Replication

### What is Asynchronous Replication?

The leader **doesn't wait** for follower confirmation — it acknowledges immediately after writing locally.

```md
Client           Leader          Follower 1
  │                │                 │
  │──Write────────▶│                 │
  │                │ (write locally) │
  │◄──Success──────│                 │ ← Fast!
  │                │──Replicate─────▶│
  │                │                 │
```

**Timeline**:

1. Client sends write to leader (t=0ms)
2. Leader writes locally (t=1ms)
3. Leader responds to client (t=2ms) ← **Fast!**
4. Leader replicates to follower (t=5ms) ← **In background**

---

### Real-World Analogy: Sending Email 📧

#### Asynchronous (Email)

- You hit "Send" → **Instant "Sent" confirmation**
- Email actually delivers seconds/minutes later
- You don't wait to see if recipient received it

#### Characteristics

- **Low latency**: Client gets instant response
- **Higher availability**: Writes succeed even if follower is down
- **Eventual consistency**: Followers catch up eventually

---

### Pros

- **Lower latency**: Client doesn't wait for followers
- **Higher availability**: Writes succeed even if followers are slow/down
- **Higher throughput**: Not limited by follower speed

### Cons

- **Potential data loss**: If leader crashes before replicating, recent writes are lost
- **Replication lag**: Followers may be seconds/minutes behind
- **Stale reads**: Reading from follower may return old data

---

## 6. Synchronous vs Asynchronous: Trade-offs

| Aspect | Synchronous | Asynchronous |

|--------|-------------|--------------|
| **Latency** | Higher (wait for follower) | Lower (immediate response) |
| **Durability** | Strong (data on multiple nodes) | Weaker (data might be only on leader) |
| **Availability** | Lower (blocked if follower down) | Higher (independent of follower) |
| **Consistency** | Strong (follower up-to-date) | Eventual (replication lag) |
| **Throughput** | Lower (limited by slowest follower) | Higher (not blocked) |
| **Data loss risk** | Minimal (if 1+ follower ACKs) | Possible (if leader crashes) |

---

## 7. Semi-Synchronous Replication (Best of Both Worlds)

### What is Semi-Synchronous?

Wait for **at least one follower** (but not all) to acknowledge the write.

- **Leader** waits for 1 follower to ACK
- Other followers replicate asynchronously
- **Balance**: Some durability guarantee without too much latency

### Configuration Example (MySQL)

```sql
-- Wait for at least 1 replica to acknowledge
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
```

### Benefits

- **Durability**: Data on 2+ nodes (leader + 1 follower)
- **Performance**: Only wait for fastest follower
- **Availability**: Other followers don't block writes

---

## 8. Handling Node Failure (Failover)

### What is Failover?

**Failover** is the process of automatically switching to a standby node when the active node fails.

---

### Scenario: Leader Failure

#### What Happens When Leader Fails?

```md
Before Failure:
         ┌─────────┐
         │ Leader  │ ◄─── Writes
         └────┬────┘
              │
       ┌──────┼──────┐
       │      │      │
       ▼      ▼      ▼
   Follower1 Follower2 Follower3

After Failure (Leader crashes):
         ┌─────────┐
         │ Leader  │ ✖✖✖ CRASHED
         └─────────┘
       
       Follower1 promoted to leader:
         ┌─────────┐
         │ Leader  │ ◄─── New leader
         │(was F1) │
         └────┬────┘
              │
         ┌────┼────┐
         │    │    │
         ▼    ▼    ▼
      Follower2 Follower3
```

---

### Real-World Analogy: Head Chef and Cooks 👨‍🍳

#### Scenario: Restaurant Kitchen

- **Head chef (leader)**: Coordinates all cooking, decides menu changes
- **Cooks (followers)**: Follow head chef's instructions

#### What if Head Chef Leaves?

1. **Detect absence**: Cooks notice head chef is gone (5 minutes)
2. **Elect new head chef**: Most experienced cook takes over
3. **Continue service**: New head chef coordinates kitchen
4. **Catch up**: New chef reviews recent orders, continues where old chef left off

---

## 9. Failover Process

### Step 1: **Detect Failure (Timeout Mechanism)**

#### How Detection Works

- **Heartbeats**: Leader sends periodic "I'm alive" signals
- **Timeout**: If no heartbeat for N seconds, assume leader is dead

#### Example

```python
# Follower monitors leader
while True:
    if not received_heartbeat_in_last(30_seconds):
        trigger_failover()
    sleep(5)
```

#### Challenges

- **False positives**: Network hiccup → incorrectly assume leader is down
- **Split-brain**: Multiple nodes think they're the leader (dangerous!)

---

### Step 2: **Leader Election**

#### How to Choose a New Leader?

##### Option 1: **Predetermined Priority**

- Designate a specific follower as "next in line"
- Fast, but inflexible

##### Option 2: **Consensus Algorithm**

- Followers vote to elect a new leader
- **Raft, Paxos, ZooKeeper** handle this
- Most up-to-date replica usually wins

##### Option 3: **External Coordinator**

- An external system (e.g., ZooKeeper, etcd) manages leader election

---

### Step 3: **Reconfigure System**

#### Actions

1. **Promote follower** to leader
2. **Update clients**: Redirect writes to new leader
3. **Update followers**: Point to new leader for replication

#### Example (Automatic failover with DNS)

```md
Before: db-master.example.com → 192.168.1.10 (old leader)
After:  db-master.example.com → 192.168.1.11 (new leader)
```

Clients automatically connect to new leader.

---

### Step 4: **Catch-Up Recovery**

#### Problem: What if Old Leader Comes Back?

- Old leader has been down, so it's behind
- It should become a **follower** (not reclaim leadership)
- Needs to **catch up** on missed changes

#### Process

1. Old leader reconnects as follower
2. Requests replication log from new leader
3. Applies all missed changes
4. Catches up to current state

---

## 10. Challenges in Failover

### Challenge 1: **Unreplicated Writes (Data Loss)**

#### Problem

- Leader crashes **before** replicating recent writes
- Those writes are lost forever

#### Example

```md
t=0: Leader receives write: "UPDATE balance = 100"
t=1: Leader writes to disk
t=2: Leader CRASHES (before replicating)
     ❌ Write is lost
```

#### Solutions

- **Synchronous replication**: Wait for at least 1 follower ACK (slower but safer)
- **WAL (Write-Ahead Log)**: Persist to durable storage before ACK
- **Idempotent retries**: Client retries if no response (but risk of duplicates)

---

### Challenge 2: **Split-Brain (Multiple Leaders)**

#### Problem

Network partition causes both old and new leader to accept writes simultaneously.

#### Scenario

```md
Partition:
   ┌─────────┐          Network Partition          ┌─────────┐
   │ Leader A│  ✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖✖  │ Leader B│
   └─────────┘                                    └─────────┘
      │                                               │
   Follower1                                      Follower2
```

Both leaders accept writes → **data diverges** → **conflict!**

#### Solutions

- **Quorum**: Require majority consensus (e.g., Raft)
- **Fencing tokens**: Leader has a unique token; old token is rejected
- **STONITH**: "Shoot The Other Node In The Head" (forcefully shut down old leader)

---

### Challenge 3: **Timeout Tuning**

#### Problem: Setting the Right Timeout

- **Too short**: False positives → unnecessary failovers
- **Too long**: Users wait longer during actual failures

**Example**:

- Timeout = 5 seconds: Fast failover, but might trigger on network hiccup
- Timeout = 60 seconds: Safer, but 1 minute downtime

#### Best Practice

- Use **adaptive timeouts** (adjust based on network conditions)
- Monitor failure patterns and tune accordingly

---

### Challenge 4: **Replication Lag**

#### Problem: Followers Are Behind Leader

**Example**:

- Leader writes at t=0
- Follower applies write at t=10 (10-second lag)
- Client reads from follower at t=5 → **stale data!**

#### Solutions
- **Read from leader**: For critical reads requiring fresh data
- **Monotonic reads**: Ensure subsequent reads don't go backward in time
- **Read-your-writes consistency**: Route user's reads to leader temporarily after write

---

## 11. Handling Follower Failure

### Catch-Up Recovery

#### Scenario: Follower Crashes and Restarts

**Process**:

1. Follower comes back online
2. Requests all changes since its last known position (using log sequence number)
3. Leader sends missing changes
4. Follower applies them and catches up

#### Example (PostgreSQL WAL)

```sql
-- Follower tracks its position
Last applied: WAL position 5000

-- Requests changes from leader
Request: "Give me WAL from 5000 onwards"

-- Leader sends WAL segments 5000-7000
-- Follower applies and catches up
```

---

## 12. Real-World Replication Architectures

### Single-Leader Replication (Most Common)

**Examples**:

- PostgreSQL (primary-standby)
- MySQL (master-slave)
- MongoDB (primary-secondary in replica sets)

**Best for**: Most applications (simple, consistent)

---

### Multi-Leader Replication (Complex)

**Setup**: Multiple leaders, each accepts writes
**Conflict resolution**: Required when same data updated on multiple leaders
**Examples**: CouchDB, Cassandra (multi-datacenter)

**Best for**: Multi-datacenter deployments, offline-first apps

---

### Leaderless Replication (Distributed)

**Setup**: No leader, clients write to multiple nodes
**Quorum**: Write to N nodes, read from N nodes (N > total/2)
**Examples**: Cassandra, Riak, DynamoDB

**Best for**: High availability, eventual consistency acceptable

---

## 13. Summary

### Key Takeaways

1. **Replication** improves latency, availability, and scalability by maintaining multiple copies of data
2. **Leader-based replication**: Single leader handles writes, followers handle reads
3. **Synchronous replication**: Wait for follower ACK (strong durability, higher latency)
4. **Asynchronous replication**: Don't wait for followers (low latency, risk of data loss)
5. **Failover**: Automatically promote follower to leader when leader fails
6. **Failure detection**: Use heartbeats and timeouts to detect node failures
7. **Split-brain**: Dangerous scenario where multiple nodes think they're leader (prevent with quorum, fencing)
8. **Replication lag**: Followers may be behind leader (use read-from-leader for strong consistency)
9. **Catch-up recovery**: Failed nodes replay missed changes when they come back

---

### Design Considerations

#### Choosing Replication Strategy

| Requirement | Strategy |
|-------------|----------|

| **Strong consistency** | Synchronous replication (wait for ACK) |
| **Low latency writes** | Asynchronous replication |
| **High availability** | Asynchronous with multiple replicas |
| **Zero data loss** | Synchronous with quorum (e.g., 2 out of 3 nodes) |
| **Read scaling** | Leader-based with many followers |
| **Multi-datacenter** | Multi-leader or leaderless |

---

### Next Steps

This completes the core topics for Day-2! You now understand:

- OLTP vs OLAP and data warehousing
- Data encoding and schema evolution
- Binary serialization (Protobuf, Thrift, Avro)
- Data flow patterns (RPC, message queues, pub-sub)
- Replication and distributed systems

---

**Related Topics**: Consensus algorithms (Raft, Paxos), Partitioning/Sharding, Distributed transactions, CAP theorem
