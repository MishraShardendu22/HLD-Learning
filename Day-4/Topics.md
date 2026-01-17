# Database Transactions & Isolation Topics

## Overview

This file summarizes the key topics covered across Day-4 chapters. Topics focus on transactions, ACID properties, isolation levels, concurrency anomalies, and how databases handle concurrent access to data.

---

### Chapter 1 — Transactions & ACID Properties (see `Chapter-1-Transactions-ACID.md`) 💳

- What is a transaction? (with ATM & order examples)
- The need for Atomicity – all-or-nothing operations
- Ensuring Consistency via constraints and business rules
- Isolation levels: preventing dirty reads, lost updates, and race conditions
- Durability – how databases survive crashes (WAL, replication)
- Real-world anomalies: lost updates, phantom reads, etc.
- Concurrency control: Locks vs MVCC (pessimistic vs optimistic)
- Single-node vs Distributed Transactions
- Two-Phase Commit Protocol (2PC) explained
- Real-world use cases: ticket booking, inventory, banking systems
- When you should RELAX consistency for performance
- What is Atomicity? All-or-nothing operations explained
- Understanding Isolation – preventing interference between transactions
- Real-world examples: bank transfer, shopping cart, multi-document writes
- Why even single-record updates need atomicity and isolation
- Handling failures, retries, and error scenarios safely
- How databases use WAL, MVCC, and locking for safety
- Multi-object transactions – why they're hard in distributed systems
- Use cases: email updates, foreign keys, denormalized fields, and more
- Best-effort vs ACID-compliant databases – trade-offs and choices
- How to avoid race conditions, anomalies, and data loss

---

### Chapter 2 — Isolation Levels (see `Chapter-2-Isolation-Levels.md`) 🔒

- Introduction to Isolation
- What is Database Isolation?
- Kitchen Analogy for Isolation
- Isolation Levels & Trade-offs
- Read Committed: A Common Default
- Example Scenario: Shared Grocery List
- Concurrency Anomalies
- Dirty Reads: Seeing Uncommitted Data
- Dirty Read Example (Alice & Bob with Chocolates)
- Impact of Dirty Reads (Bank Example)
- Preventing Dirty Reads
- Dirty Writes: Overwriting Uncommitted Data
- Dirty Write Example (Alice & Bob with A=5 vs A=15)
- Dangers of Dirty Writes & Lost Updates
- Preventing Dirty Writes
- Read Committed Limitations
- Non-Repeatable Reads Explained
- Phantom Reads Explained
- Non-Repeatable vs. Phantom Reads: The Difference
- Why Read Committed Allows Them (Trade-off)
- Real-World Use Case: Product Catalog
- How Databases Implement Isolation: Pessimistic Locking
- Multi-Version Concurrency Control (MVCC)
- MVCC for Read Committed
- Problem with Read Committed Isolation
- Why Consistent Snapshots Matter
- Snapshot Isolation's Role & How it Works
- Why Read Committed Can Be Misleading (Alice's Bank Account Example)
- What is Read Skew (Non-Repeatable Read)
- How Read Committed Allows Inconsistencies
- Impact of Temporary Inconsistencies (Database Backups, Analytics)
- The Need to "Freeze" the Database
- Why Snapshot Matters & What is Snapshot Isolation (Freezing Time)
- Snapshot Isolation: Like Taking a Photograph of the Database
- Benefits of Snapshot Isolation (Long-Running & Read-Only Operations)
- Databases Supporting Snapshot Isolation (PostgreSQL, Oracle, MySQL, SQL Server)
- How MVCC Powers Snapshot Isolation (Multi-Version Concurrency Control)
- MVCC: Keeping Multiple Versions of Each Row
- How MVCC Works (Created By, Deleted By IDs)
- Visibility Rules in MVCC
- Indexing in MVCC-Based Databases
- CouchDB & Datomic: Append-Only B-Trees
- PostgreSQL's Approach to Indexing in MVCC
- Repeatable Read vs. Snapshot Isolation: Naming Confusion
- SQL Standard & Isolation Levels
- Importance of Reading Database Documentation
- Practical Applications (Backups, Analytics, Long Queries)

---

### Chapter 3 — Lost Updates, Write Skew & Serializability (see `Chapter-3-Lost-Updates-Write-Skew-Serializability.md`) ⚠️

- Understanding the Lost Update Problem
- Lost Update Example (Alice & Bob Bank Account)
- When Do Lost Updates Happen?
- Real-World Lost Update Scenarios
- Key Methods to Prevent Lost Updates
- Atomic Write Operations Explained
- Atomic Write Example (SQL UPDATE increment)
- How Atomic Writes Prevent Lost Updates
- Pessimistic Locking Explained
- SELECT FOR UPDATE in SQL
- Pessimistic Locking Example
- How Pessimistic Locking Works
- Analogy: Locking a Shared Document
- Drawbacks of Pessimistic Locking
- Automatic Detection: Snapshot Isolation
- Snapshot Isolation & "First Committer Wins"
- PostgreSQL Repeatable Read Example
- Oracle's Default Behavior & Version Columns
- How Version Columns Prevent Lost Updates
- Optimistic Concurrency: Compare and Set
- Compare and Set with Version Numbers
- Compare and Set SQL Example
- Web Application Example
- Comparing Data Fields (Without Version Numbers)
- Replicated Databases & Conflict Resolution
- Strategies: CRDTs vs. Last Write Wins (LWW)
- Convergent Replicated Data Types (CRDTs)
- CRDT Counter Example
- CRDTs for Independent Updates
- Last Write Wins (LWW) Explained
- LWW in Riak & Cassandra
- CRDTs vs. LWW: Safety & Risk
- Introduction to Write Skew
- What is Write Skew? (Not Dirty Write)
- Write Skew vs. Lost Update
- Example: The Pizza Problem
- Write Skew: Anomaly of Snapshot Isolation
- Example: Doctors on Call
- Impact of Write Skew
- Why Snapshot Isolation Allows Write Skew
- What are Phantom Reads?
- Phantom Reads & Write Skew Relationship
- Materializing Conflicts (Workaround)
- Why Materializing Conflicts is Complex
- Serializable Isolation: The Robust Solution
- Serializable Isolation in Doctor's Example
- Introduction to Serializable Isolation
- What is Serializable Isolation?
- Example: Preventing Lost Updates
- Concurrent Schedules & Isolation Levels
- Why Serializable Isolation Guarantees Correctness
- Actual Serial Execution: One Transaction at a Time
- Analogy: Vending Machine & Cash Register
- Modern Systems & Serial Execution
- Challenges of Serial Execution: Waiting Time, Single Core Limit, Memory
- Stored Procedures vs. Interactive Transactions
- Example: Flight Booking with Stored Procedures
- Why Databases Require Stored Procedures for Serial Execution
- Scaling with Partitioning
- Cross-Partition Transactions & Coordination
- Pros and Cons of Serializable Isolation via Serial Execution
- Two-Phase Locking (2PL)
- What is Two-Phase Locking (2PL)?
- 2PL Timeline: Growing and Shrinking Phases
- 2PL vs. Two-Phase Commit (2PC)
- How Locks Prevent Dirty Writes
- Why 2PL is Stricter Than Simple Locking
- Shared vs. Exclusive Locks & Lock Upgrades
- Deadlocks: Understanding and Resolution
- Performance Impact of 2PL
- Predicate Locks and Phantom Reads
- Index Range Locks as an Approximation
- Serializable Snapshot Isolation (SSI)
- Understanding Concurrency Anomalies
- MVCC and Consistent Snapshots Explained
- Pessimistic vs. Optimistic Concurrency Control
- Detecting Stale Reads and Write/Read Conflicts in SSI
- How SSI Works Step-by-Step
- SSI in the Real World: PostgreSQL & FoundationDB Examples
- When and Why to Use SSI: Performance Trade-offs & Best Practices

---

## Key Concepts Summary

### ACID Properties

- **Atomicity**: All-or-nothing execution
- **Consistency**: Data integrity and constraints
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data survives crashes

### Isolation Levels (Weakest to Strongest)

1. **Read Uncommitted**: Allows dirty reads
2. **Read Committed**: Prevents dirty reads/writes
3. **Repeatable Read**: Prevents non-repeatable reads
4. **Serializable**: Prevents all anomalies

### Concurrency Anomalies

- **Dirty Read**: Reading uncommitted data
- **Dirty Write**: Overwriting uncommitted data
- **Non-Repeatable Read**: Same query yields different results
- **Phantom Read**: New rows appear between queries
- **Lost Update**: Concurrent modifications lose data
- **Write Skew**: Different rows violate constraint

### Concurrency Control

- **Pessimistic (2PL)**: Lock before access
- **Optimistic (MVCC)**: Check at commit time
- **Serial Execution**: One transaction at a time

### Serializability Implementations

1. **Actual Serial Execution**: VoltDB, Redis
2. **Two-Phase Locking (2PL)**: Traditional databases
3. **Serializable Snapshot Isolation (SSI)**: PostgreSQL, FoundationDB

---

## Real-World Examples

| Use Case | Isolation Level | Strategy |
|----------|----------------|----------|
| **Banking Transfer** | Serializable | 2PC, Strong consistency |
| **E-commerce Checkout** | Repeatable Read | Pessimistic locking |
| **Ticket Booking** | Serializable | SELECT FOR UPDATE |
| **Shopping Cart** | Read Committed | Optimistic (compare-and-set) |
| **Social Media Likes** | Read Uncommitted | Eventually consistent |
| **Database Backup** | Snapshot Isolation | Consistent point-in-time |
| **Analytics Query** | Snapshot Isolation | Long-running reads |
| **Inventory System** | Serializable | Prevent overselling |
| **Meeting Room Booking** | Serializable | Prevent double-booking |

---

## Prevention Techniques

### Lost Updates

1. Atomic write operations
2. SELECT FOR UPDATE (pessimistic)
3. Snapshot isolation (optimistic)
4. Compare-and-set (optimistic)
5. CRDTs (conflict-free)
6. Last Write Wins (accept data loss)

### Write Skew

1. Serializable isolation
2. Explicit locking (materialize conflicts)
3. Database constraints
4. Application-level checking

---

## Database Comparison

| Database | Default Isolation | MVCC | Serializable | Notes |
|----------|------------------|------|--------------|-------|
| **PostgreSQL** | Read Committed | Yes | SSI | Excellent MVCC |
| **MySQL InnoDB** | Repeatable Read | Yes | 2PL | Gap locking |
| **Oracle** | Read Committed | Yes | SSI | Commercial |
| **SQL Server** | Read Committed | Yes | 2PL | Windows-focused |
| **MongoDB** | Read Committed | Yes | Snapshot | Document DB |
| **Cassandra** | Eventual | No | No | AP system |
| **DynamoDB** | Eventual | No | No | AP system |
| **VoltDB** | Serializable | No | Serial | In-memory |

---

## Interview Focus Areas

1. **Explain ACID** properties with real-world examples
2. **Isolation levels** and what anomalies each prevents
3. **Lost update problem** and prevention methods
4. **Write skew** scenario and why snapshot isolation doesn't prevent it
5. **MVCC implementation** and visibility rules
6. **2PL vs SSI** trade-offs
7. **When to use serializable** isolation (cost vs benefit)
8. **Distributed transactions** and 2PC protocol
9. **Compare-and-set** pattern for optimistic concurrency
10. **CRDTs** and how they enable conflict-free updates

---

## Additional Topics (For Reference)

### Distributed Systems Challenges

- Partial failures and non-deterministic behavior
- Network partitions and the CAP Theorem
- Unreliable clocks and timing knowledge
- Time of Day Clocks vs. Monotonic Clocks
- Google Spanner's approach to time
- Consensus and majority voting
- Fencing tokens for preventing split-brain
- Leader election in partitioned networks
- Quorum protocols (Raft, Paxos)
- Chaos engineering and resilience testing
- HPC vs. Cloud fault tolerance strategies
