# Chapter 1: Transactions & ACID Properties

## Overview

This chapter covers **transactions** — the fundamental unit of work in databases that ensures data reliability. We'll explore the ACID properties (Atomicity, Consistency, Isolation, Durability) with real-world examples and understand how databases handle failures and concurrency.

---

## 1. What is a Transaction?

### Definition

A **transaction** is a **sequence of operations** treated as a **single unit of work** — either all operations succeed or all fail.

### Real-World Analogy: ATM Withdrawal 🏧

```md
Steps:
  1. Check balance
  2. Deduct amount from account
  3. Dispense cash
  4. Print receipt
  
Problem without transaction:
  Step 2: Deduct $100 ✅
  Step 3: Machine jams, no cash dispensed ❌
  Result: Money deducted, no cash received! 💸 Lost money
  
Solution with transaction:
  BEGIN TRANSACTION
    1. Check balance
    2. Deduct amount
    3. Dispense cash
    4. Print receipt
  COMMIT TRANSACTION
  
  If ANY step fails → ROLLBACK (undo all changes)
```

---

## 2. ACID Properties

**ACID** is the set of properties that guarantee reliable database transactions.

### A = Atomicity

**All-or-nothing execution** — either all operations in transaction succeed, or none do.

#### Example: Bank Transfer

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
COMMIT;

-- If either UPDATE fails → Both are rolled back ✅
```

#### Without Atomicity:
```md
Step 1: Deduct $100 from Alice ✅
Step 2: Database crashes ❌
Step 3: Add $100 to Bob (never happens)

Result: Alice lost $100, Bob received nothing 💥
```

#### With Atomicity:
```md
Step 1: Deduct $100 from Alice ✅
Step 2: Database crashes ❌
Step 3: On restart, system rolls back transaction
Result: Alice still has $100, Bob has $0 ✅ (consistent state)
```

---

### C = Consistency

**Ensures data remains in valid state** according to defined rules and constraints.

#### Example: Foreign Key Constraint

```sql
-- Orders table has foreign key to users
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Try to insert order for non-existent user
BEGIN TRANSACTION;
  INSERT INTO orders (id, user_id) VALUES (1, 999);
COMMIT;

-- Result: ❌ Transaction fails (constraint violation)
-- Database remains consistent ✅
```

#### Example: Business Rule Consistency

```sql
-- Rule: Account balance must never be negative
CREATE TABLE accounts (
  id VARCHAR(50) PRIMARY KEY,
  balance DECIMAL(10,2) CHECK (balance >= 0)
);

BEGIN TRANSACTION;
  -- Alice has $50
  UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
  -- Would result in balance = -50
COMMIT;

-- Result: ❌ Transaction fails (check constraint)
-- Balance remains $50 ✅
```

---

### I = Isolation

**Concurrent transactions don't interfere with each other** — appears as if they run sequentially.

#### Example: Two Users Booking Last Seat

```md
Scenario: Flight has 1 seat left

User A:                       User B:
  1. Check seats: 1 left       1. Check seats: 1 left
  2. Book seat                 2. Book seat
  3. Deduct from inventory     3. Deduct from inventory
  
Without Isolation:
  Both see 1 seat, both book → Overbooking! ❌
  
With Isolation:
  User A's transaction locks seat
  User B waits
  User A completes → seats = 0
  User B sees 0 seats → Booking fails ✅
```

#### SQL Example:

```sql
-- User A's transaction
BEGIN TRANSACTION;
  SELECT seats FROM flights WHERE id = 'AA100' FOR UPDATE; -- Lock!
  -- Result: seats = 1
  UPDATE flights SET seats = seats - 1 WHERE id = 'AA100';
  -- seats = 0
COMMIT;

-- User B's transaction (waits for A's lock)
BEGIN TRANSACTION;
  SELECT seats FROM flights WHERE id = 'AA100' FOR UPDATE;
  -- Result: seats = 0 (A's change is visible)
  -- Booking logic rejects (no seats available)
ROLLBACK;
```

---

### D = Durability

**Committed transactions survive crashes** — data is permanently saved even if system fails immediately after commit.

#### How Durability is Achieved

##### 1. **Write-Ahead Log (WAL)**

Before committing, write transaction log to disk.

```md
Transaction: Transfer $100 from Alice to Bob

Step 1: Write to WAL (durable storage):
  LOG: [TX-123] Alice: $500 → $400
  LOG: [TX-123] Bob: $200 → $300
  LOG: [TX-123] COMMIT
  
Step 2: Acknowledge commit to client ✅
Step 3: Update actual data (can happen later)

If crash happens after step 2:
  On restart: Replay WAL
  Result: Data is recovered ✅
```

##### 2. **Replication**

Replicate committed data to multiple nodes.

```md
Leader writes data → Followers replicate
If leader crashes: Followers have data ✅
```

#### Real-World Example: PostgreSQL

```md
PostgreSQL uses WAL:
  1. Transaction writes to WAL
  2. WAL flushed to disk (fsync)
  3. COMMIT returns to client
  4. Background process writes to data files
  
Even if crash happens after COMMIT:
  - WAL is on disk
  - On restart: Replay WAL
  - Data recovered ✅
```

---

## 3. Real-World Transaction Examples

### Example 1: E-commerce Order

```sql
BEGIN TRANSACTION;
  -- 1. Deduct inventory
  UPDATE products SET stock = stock - 1 WHERE id = 'laptop-123';
  
  -- 2. Create order
  INSERT INTO orders (id, user_id, product_id, amount)
  VALUES (1, 'user_456', 'laptop-123', 999.99);
  
  -- 3. Charge payment
  INSERT INTO payments (order_id, amount, status)
  VALUES (1, 999.99, 'charged');
  
  -- 4. Update loyalty points
  UPDATE users SET points = points + 100 WHERE id = 'user_456';
  
COMMIT;
```

**What happens if:**
- Step 2 fails? → Rollback (inventory restored, no order created)
- Step 3 fails? → Rollback (inventory restored, order deleted)
- Crash after commit? → Durability ensures all changes persisted

---

### Example 2: Multi-Document Update in MongoDB

```javascript
const session = client.startSession();

session.startTransaction();

try {
  // Update user profile
  await db.users.updateOne(
    { _id: userId },
    { $set: { email: 'newemail@example.com' } },
    { session }
  );
  
  // Update all orders with new email
  await db.orders.updateMany(
    { userId: userId },
    { $set: { userEmail: 'newemail@example.com' } },
    { session }
  );
  
  // Update audit log
  await db.auditLog.insertOne(
    { action: 'email_changed', userId, timestamp: new Date() },
    { session }
  );
  
  await session.commitTransaction();
  console.log('Transaction committed ✅');
} catch (error) {
  await session.abortTransaction();
  console.log('Transaction rolled back ❌');
} finally {
  session.endSession();
}
```

---

### Example 3: Shopping Cart Checkout

```sql
BEGIN TRANSACTION;
  -- 1. Validate cart items are still available
  SELECT stock FROM products WHERE id IN (
    SELECT product_id FROM cart_items WHERE cart_id = 'cart_789'
  ) FOR UPDATE;
  
  -- 2. Create order from cart
  INSERT INTO orders (user_id, total)
  SELECT user_id, SUM(price * quantity)
  FROM cart_items
  WHERE cart_id = 'cart_789';
  
  -- 3. Deduct stock for each item
  UPDATE products p
  SET stock = stock - c.quantity
  FROM cart_items c
  WHERE p.id = c.product_id AND c.cart_id = 'cart_789';
  
  -- 4. Clear cart
  DELETE FROM cart_items WHERE cart_id = 'cart_789';
  
COMMIT;
```

---

## 4. Single-Node vs Distributed Transactions

### Single-Node Transactions

All data on one database server.

```md
┌─────────────────┐
│  Database Node  │
│  ┌───────────┐  │
│  │  Users    │  │
│  │  Orders   │  │
│  │  Products │  │
│  └───────────┘  │
└─────────────────┘

Transaction touches only this node
→ Easy to implement ACID ✅
```

---

### Distributed Transactions

Data spread across multiple nodes/services.

```md
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  User DB    │    │  Order DB   │    │ Payment API │
│  Node 1     │    │  Node 2     │    │  Node 3     │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                    Transaction spans 3 systems
                    → Much harder to implement ACID ❌
```

**Challenges:**
1. Network failures between nodes
2. Partial failures (some nodes succeed, others fail)
3. Higher latency (coordination overhead)

---

## 5. Two-Phase Commit (2PC) Protocol

### What is 2PC?

**Two-Phase Commit** is a protocol for coordinating distributed transactions across multiple nodes.

### How It Works

#### Phase 1: Prepare (Voting Phase)

Coordinator asks all participants: "Can you commit?"

```md
Coordinator:
  → Participant A: "Prepare to commit"
  → Participant B: "Prepare to commit"
  → Participant C: "Prepare to commit"
  
Participants respond:
  Participant A: "Yes, ready to commit" ✅
  Participant B: "Yes, ready to commit" ✅
  Participant C: "No, error occurred" ❌
```

#### Phase 2: Commit/Abort (Execution Phase)

If all vote "Yes" → Commit, else → Abort

```md
Scenario 1: All voted Yes
  Coordinator:
    → Participant A: "COMMIT"
    → Participant B: "COMMIT"
    → Participant C: "COMMIT"
  
  All participants commit ✅

Scenario 2: Any voted No
  Coordinator:
    → Participant A: "ABORT"
    → Participant B: "ABORT"
    → Participant C: "ABORT"
  
  All participants rollback ✅
```

### Real-World Example: Multi-Database Order

```md
System:
  - MySQL (User DB)
  - PostgreSQL (Inventory DB)
  - MongoDB (Order DB)
  
Transaction: Place order
  
Step 1 (Prepare):
  Coordinator → MySQL: "Can you deduct user balance?"
  MySQL: "Yes" ✅
  
  Coordinator → PostgreSQL: "Can you reserve inventory?"
  PostgreSQL: "Yes" ✅
  
  Coordinator → MongoDB: "Can you create order?"
  MongoDB: "Yes" ✅
  
Step 2 (Commit):
  All voted Yes → Coordinator sends COMMIT to all
  Result: Order placed successfully ✅
```

### Problems with 2PC

#### 1. **Blocking**

If coordinator crashes after prepare, participants are stuck waiting.

```md
Phase 1: Participants prepared (locks held)
Phase 2: Coordinator crashes ❌
Participants: Waiting forever? Locks held → Other transactions blocked 🚫
```

#### 2. **Single Point of Failure**

Coordinator failure blocks entire transaction.

#### 3. **Performance**

Multiple round trips increase latency.

**Alternatives:**
- Eventual consistency (no distributed transactions)
- Saga pattern (compensating transactions)
- Three-Phase Commit (3PC) — less blocking

---

## 6. Concurrency Control

### Pessimistic Concurrency Control (Locking)

**Lock resources before accessing** to prevent conflicts.

```sql
-- Pessimistic approach
BEGIN TRANSACTION;
  SELECT * FROM accounts WHERE id = 'Alice' FOR UPDATE; -- Acquire lock
  -- Other transactions wait here
  UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
COMMIT; -- Release lock
```

**Pros:**
- Prevents conflicts
- Guarantees consistency

**Cons:**
- Lower throughput (transactions wait for locks)
- Risk of deadlocks

---

### Optimistic Concurrency Control (MVCC)

**Assume no conflicts, check at commit time**.

```sql
-- Optimistic approach (MVCC)
BEGIN TRANSACTION;
  SELECT * FROM accounts WHERE id = 'Alice'; -- No lock
  -- Read version: V1
  
  -- Other transactions can read/write concurrently
  
  UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
COMMIT;
  -- At commit: Check if version changed
  -- If version is still V1 → Commit ✅
  -- If version is now V2 (someone else modified) → Retry ❌
```

**Pros:**
- Higher throughput (no waiting)
- Better for read-heavy workloads

**Cons:**
- Transaction may need to retry
- More complex implementation

---

## 7. Real-World Use Cases

### Use Case 1: Banking System

```md
Requirement: Transfer money between accounts
  - Must be atomic (all-or-nothing)
  - Must be consistent (total money unchanged)
  - Must be isolated (no race conditions)
  - Must be durable (survives crashes)
  
Solution: Traditional ACID transactions
  - Strong consistency
  - Synchronous replication
  - Use 2PC for distributed accounts
```

---

### Use Case 2: Ticket Booking System

```md
Requirement: Prevent double-booking
  - High concurrency (many users booking simultaneously)
  - Must guarantee seat availability
  
Solution: Pessimistic locking
  - Lock seat when checking availability
  - Hold lock until booking confirmed
  - Release lock on commit/rollback
```

---

### Use Case 3: Social Media Feed (Eventual Consistency)

```md
Requirement: Post updates visible to followers
  - High throughput (millions of posts/sec)
  - Can tolerate temporary inconsistency
  
Solution: Relax ACID
  - No transactions (eventual consistency)
  - Asynchronous replication
  - Use message queues for fan-out
  
Trade-off: Faster, but temporary inconsistency ✅
```

---

## 8. When to Relax Consistency

### Scenarios Where Eventual Consistency is OK

1. **Social media likes/views**
   - Approximate counts acceptable
   - Example: YouTube view count

2. **Analytics dashboards**
   - Historical data, not real-time critical
   - Example: Google Analytics

3. **Product recommendations**
   - Based on past data, not instantly critical
   - Example: Netflix recommendations

### Scenarios Where Strong Consistency is Required

1. **Financial transactions**
   - Money transfers must be exact
   
2. **Inventory management**
   - Prevent overselling
   
3. **Authentication**
   - Password changes must be immediate

---

## 9. Key Takeaways

1. **Transaction = Unit of work** (all-or-nothing)
2. **ACID Properties:**
   - **Atomicity**: All succeed or all fail
   - **Consistency**: Data remains valid
   - **Isolation**: Concurrent transactions don't interfere
   - **Durability**: Committed data survives crashes
3. **Single-node transactions** are easier than distributed
4. **2PC protocol** coordinates distributed transactions (but has limitations)
5. **Concurrency control**: Pessimistic (locks) vs Optimistic (MVCC)
6. **Trade-offs**: Strong consistency (slow) vs Eventual consistency (fast)

### Decision Matrix

| Use Case | ACID Level | Approach |
|----------|-----------|----------|
| Banking | Full ACID | Pessimistic locking |
| E-commerce checkout | Full ACID | Optimistic (MVCC) |
| Social media likes | Relaxed | Eventual consistency |
| Ticket booking | Full ACID | Pessimistic locking |
| Analytics | Relaxed | Eventual consistency |
| Authentication | Full ACID | Strong consistency |
