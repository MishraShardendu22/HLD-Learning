# Chapter 3: Lost Updates, Write Skew & Serializability

## Overview

This chapter covers advanced concurrency problems: **lost updates**, **write skew**, and **phantom writes**. We'll explore prevention techniques and understand **Serializable isolation** — the strongest isolation level that prevents all anomalies.

---

## 1. The Lost Update Problem

### What is a Lost Update?

When two transactions read the same value, modify it, and write it back — one update is "lost."

### Example: Alice and Bob's Bank Account

```md
Initial: balance = $100

Alice's Transaction:
  T1: READ balance → $100
  T2: Calculate: $100 + $50 = $150
  T3: WRITE balance = $150
  
Bob's Transaction (concurrent):
  T1: READ balance → $100 (reads same value)
  T2: Calculate: $100 + $25 = $125
  T3: WRITE balance = $125
  
Final: balance = $125 ❌

Expected: balance = $175 ($100 + $50 + $25)
Lost: Alice's $50 increment 💸
```

---

### Real-World Scenarios

#### 1. **Counter Increment**

```sql
-- User likes a post
UPDATE posts SET likes = likes + 1 WHERE id = 123;

-- Concurrent users:
Transaction A: Read likes = 10 → Write likes = 11
Transaction B: Read likes = 10 → Write likes = 11
Final: likes = 11 ❌ (Expected: 12)
```

#### 2. **Shopping Cart**

```javascript
// User A adds item to cart
const cart = await db.get('cart_123'); // {items: ['laptop']}
cart.items.push('mouse');
await db.save('cart_123', cart); // {items: ['laptop', 'mouse']}

// User B adds item (concurrent)
const cart = await db.get('cart_123'); // {items: ['laptop']}
cart.items.push('keyboard');
await db.save('cart_123', cart); // {items: ['laptop', 'keyboard']}

// Final: {items: ['laptop', 'keyboard']} ❌
// Lost: 'mouse' 💸
```

#### 3. **Wiki Page Edit**

```md
Alice and Bob edit same wiki page:

Alice:
  T1: Read page content: "Hello World"
  T2: Edit: "Hello Beautiful World"
  T3: Save
  
Bob:
  T1: Read page content: "Hello World"
  T2: Edit: "Hello Wonderful World"
  T3: Save
  
Final: "Hello Wonderful World" ❌
Lost: Alice's edit "Beautiful" 💸
```

---

## 2. Preventing Lost Updates

### Method 1: Atomic Write Operations

Use database's built-in atomic operations.

```sql
-- ❌ Lost update possible
UPDATE posts SET likes = likes + 1 WHERE id = 123;

-- ✅ Atomic increment (safe)
UPDATE posts SET likes = likes + 1 WHERE id = 123;
-- Database ensures atomicity internally
```

**How it works:**

```md
Database internally locks the row:
  1. Acquire exclusive lock on row
  2. Read current value
  3. Increment
  4. Write new value
  5. Release lock
  
Result: No lost updates ✅
```

**Example: Redis Atomic Increment**

```javascript
// ✅ Atomic
await redis.incr('post:123:likes');

// ❌ Not atomic (lost update possible)
const likes = await redis.get('post:123:likes');
await redis.set('post:123:likes', likes + 1);
```

---

### Method 2: Pessimistic Locking (SELECT FOR UPDATE)

**Explicitly lock rows** before modifying.

```sql
BEGIN TRANSACTION;
  -- Acquire exclusive lock
  SELECT balance FROM accounts WHERE id = 'alice' FOR UPDATE;
  -- Other transactions wait here ⏳
  
  -- Safe to modify
  UPDATE accounts SET balance = balance + 50 WHERE id = 'alice';
COMMIT;
-- Lock released 🔓
```

**Real-World Example: Ticket Booking**

```sql
BEGIN TRANSACTION;
  -- Lock seat row
  SELECT available FROM seats WHERE seat_id = 'A12' FOR UPDATE;
  -- Result: available = true
  
  -- Check availability
  IF available = true THEN
    -- Book seat
    UPDATE seats SET available = false, user_id = 'user_123' WHERE seat_id = 'A12';
  END IF;
COMMIT;

-- Concurrent transaction waits for lock
-- Ensures no double-booking ✅
```

**Pros:**

- Prevents all lost updates
- Strong consistency guarantee

**Cons:**

- Lower throughput (transactions wait)
- Risk of deadlocks

**Analogy: Locking a Shared Document 📄**

```md
Alice wants to edit document:
  1. Lock document 🔒
  2. Edit
  3. Save
  4. Unlock 🔓
  
Bob wants to edit (waits for Alice's lock)
  1. Wait... ⏳
  2. Alice unlocks
  3. Bob locks
  4. Edit ✅
```

---

### Method 3: Automatic Detection (Snapshot Isolation)

Database detects lost updates at commit time.

```md
Transaction A:
  T1: Read balance = $100 (version V1)
  T2: Calculate: $100 + $50 = $150
  T3: COMMIT
      - Check: Is version still V1? Yes ✅
      - Write balance = $150 (version V2)
  
Transaction B (concurrent):
  T1: Read balance = $100 (version V1)
  T2: Calculate: $100 + $25 = $125
  T3: COMMIT
      - Check: Is version still V1? No! (now V2) ❌
      - Abort transaction
      - User notified: "Retry transaction"
```

**PostgreSQL Example:**

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT balance FROM accounts WHERE id = 'alice';
  -- Read: balance = $100
  
  UPDATE accounts SET balance = balance + 50 WHERE id = 'alice';
COMMIT;
-- If concurrent update happened → Error: "could not serialize access"
-- Application must retry ♻️
```

**Pros:**

- No manual locking needed
- Higher concurrency than pessimistic locking

**Cons:**

- Transaction may abort (need retry logic)
- User sees error message

---

### Method 4: Compare-and-Set (Optimistic Locking)

Include version number or timestamp in UPDATE condition.

```sql
-- Add version column
CREATE TABLE accounts (
  id VARCHAR(50) PRIMARY KEY,
  balance DECIMAL(10,2),
  version INT
);

-- Transaction
SELECT balance, version FROM accounts WHERE id = 'alice';
-- Result: balance = $100, version = 5

UPDATE accounts 
SET balance = $150, version = version + 1
WHERE id = 'alice' AND version = 5;
-- If version is still 5 → Update succeeds ✅
-- If version is now 6 (someone else modified) → Update fails ❌

-- Check affected rows
IF affected_rows = 0 THEN
  -- Someone else modified, retry
  ROLLBACK;
END IF;
```

**Web Application Example:**

```html
<!-- Edit form includes version -->
<form action="/update-profile" method="POST">
  <input type="hidden" name="version" value="5">
  <input type="text" name="email" value="alice@example.com">
  <button type="submit">Update</button>
</form>
```

```javascript
// Server-side handler
app.post('/update-profile', async (req, res) => {
  const result = await db.query(
    'UPDATE users SET email = $1, version = version + 1 WHERE id = $2 AND version = $3',
    [req.body.email, req.session.userId, req.body.version]
  );
  
  if (result.rowCount === 0) {
    // Version mismatch → Someone else modified
    return res.status(409).json({
      error: 'Profile was modified by someone else. Please refresh and try again.'
    });
  }
  
  res.json({ success: true });
});
```

**Without Version Column:**

```sql
-- Compare actual data fields
UPDATE users
SET email = 'new@example.com'
WHERE id = 'alice' 
  AND email = 'old@example.com'  -- Must match current value
  AND name = 'Alice Smith';      -- Must match current value
```

**Pros:**

- No database locking
- Good for web applications (form submissions)

**Cons:**

- Application must handle retries
- User sees "conflict" errors

---

### Method 5: Conflict-Free Replicated Data Types (CRDTs)

Use data structures that **automatically merge** concurrent updates.

#### CRDT Counter Example

```javascript
// Traditional counter (lost update problem)
let counter = 0;
counterA++; // counter = 1
counterB++; // counter = 1 (read same initial value)
// Result: counter = 1 ❌ (Expected: 2)

// CRDT Counter (no lost updates)
class CRDTCounter {
  constructor() {
    this.increments = {}; // {nodeId: count}
  }
  
  increment(nodeId) {
    this.increments[nodeId] = (this.increments[nodeId] || 0) + 1;
  }
  
  value() {
    return Object.values(this.increments).reduce((a, b) => a + b, 0);
  }
  
  merge(other) {
    for (let nodeId in other.increments) {
      this.increments[nodeId] = Math.max(
        this.increments[nodeId] || 0,
        other.increments[nodeId]
      );
    }
  }
}

// Usage:
const counterA = new CRDTCounter();
counterA.increment('nodeA'); // {nodeA: 1}

const counterB = new CRDTCounter();
counterB.increment('nodeB'); // {nodeB: 1}

// Merge:
counterA.merge(counterB);
console.log(counterA.value()); // 2 ✅
```

**Real-World Example: Collaborative Editing (Google Docs)**

```md
Operational Transforms:
  Alice types "Hello" at position 0
  Bob types "World" at position 0 (concurrently)
  
  System transforms operations:
    Alice's "Hello" at 0
    Bob's "World" at 5 (transformed: 0 + length of "Hello")
  
  Result: "HelloWorld" ✅
  No lost updates!
```

---

### Method 6: Last Write Wins (LWW) — Lossy

Use timestamps, accept data loss.

```sql
UPDATE posts 
SET content = 'New content', updated_at = NOW() 
WHERE id = 123;

-- On conflict: Latest timestamp wins
-- Earlier writes are discarded ❌
```

**Used by:** Riak, Cassandra (default)

**When to Use:**

- Non-critical data (view counts, session data)
- When data loss is acceptable
- High availability > consistency

**When NOT to Use:**

- Financial data
- Inventory
- User-generated content

---

## 3. Write Skew

### What is Write Skew?

Two transactions read the same data, make decisions based on it, and write to **different rows** — violating a constraint.

### Example: On-Call Doctor Constraint

```md
Rule: At least 1 doctor must be on-call at all times

Current state:
  Alice: on_call = true
  Bob: on_call = true
  
Alice's Transaction:
  T1: Check: Is Bob on-call?
      Query: SELECT on_call FROM doctors WHERE id = 'bob';
      Result: true ✅ (Bob is on-call, safe to leave)
  T2: Remove self from on-call:
      UPDATE doctors SET on_call = false WHERE id = 'alice';
  T3: COMMIT
  
Bob's Transaction (concurrent):
  T1: Check: Is Alice on-call?
      Query: SELECT on_call FROM doctors WHERE id = 'alice';
      Result: true ✅ (Alice is on-call, safe to leave)
  T2: Remove self from on-call:
      UPDATE doctors SET on_call = false WHERE id = 'bob';
  T3: COMMIT
  
Final state:
  Alice: on_call = false
  Bob: on_call = false
  
Constraint violated: 0 doctors on-call! 🚨
```

---

### Why Snapshot Isolation Doesn't Prevent Write Skew

```md
Problem: Transactions write to DIFFERENT rows
  - Alice updates her row
  - Bob updates his row
  - No direct conflict detected ❌
  
Snapshot Isolation:
  - Prevents lost updates on SAME row ✅
  - Doesn't prevent write skew on DIFFERENT rows ❌
```

---

### Real-World Write Skew Examples

#### Example 1: Meeting Room Booking

```md
Rule: Max 1 booking per room at same time slot

Current bookings:
  Room A, 2PM-3PM: None
  
Alice's Transaction:
  T1: Check: Any booking for Room A at 2PM?
      Result: No ✅
  T2: Book: INSERT INTO bookings (room, time, user) VALUES ('A', '2PM', 'alice');
  T3: COMMIT
  
Bob's Transaction (concurrent):
  T1: Check: Any booking for Room A at 2PM?
      Result: No ✅ (Alice's booking not committed yet)
  T2: Book: INSERT INTO bookings (room, time, user) VALUES ('A', '2PM', 'bob');
  T3: COMMIT
  
Final: 2 bookings for same room/time ❌ Double-booking!
```

---

#### Example 2: Username Uniqueness

```md
Rule: Usernames must be unique

Alice's Transaction:
  T1: Check: SELECT * FROM users WHERE username = 'john';
      Result: Empty ✅ (username available)
  T2: INSERT INTO users (id, username) VALUES (1, 'john');
  T3: COMMIT
  
Bob's Transaction (concurrent):
  T1: Check: SELECT * FROM users WHERE username = 'john';
      Result: Empty ✅ (Alice's insert not visible yet)
  T2: INSERT INTO users (id, username) VALUES (2, 'john');
  T3: COMMIT
  
Final: 2 users with username 'john' ❌
```

**Note:** This is usually prevented by UNIQUE constraint, but write skew can happen in more complex scenarios.

---

#### Example 3: Inventory Overselling

```md
Rule: Don't sell more items than in stock

Stock: 1 laptop

Alice's Transaction:
  T1: Check: SELECT stock FROM products WHERE id = 'laptop';
      Result: 1 ✅
  T2: Create order: INSERT INTO orders (user_id, product_id) VALUES ('alice', 'laptop');
  T3: Decrement: UPDATE products SET stock = stock - 1 WHERE id = 'laptop';
  T4: COMMIT
  
Bob's Transaction (concurrent):
  T1: Check: SELECT stock FROM products WHERE id = 'laptop';
      Result: 1 ✅ (sees snapshot before Alice's decrement)
  T2: Create order: INSERT INTO orders (user_id, product_id) VALUES ('bob', 'laptop');
  T3: Decrement: UPDATE products SET stock = stock - 1 WHERE id = 'laptop';
  T4: COMMIT
  
Final: stock = -1 ❌ Oversold!
```

---

## 4. Preventing Write Skew

### Method 1: Serializable Isolation

**Strongest isolation level** — transactions appear to run serially (one after another).

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  -- Check constraint
  SELECT on_call FROM doctors WHERE on_call = true;
  
  -- Make decision based on constraint
  UPDATE doctors SET on_call = false WHERE id = 'alice';
COMMIT;
-- If concurrent transaction violates constraint → Abort ❌
```

**How it works:**

```md
Database detects that two transactions have conflicting dependencies:
  
Transaction A depends on: Bob's on_call status
Transaction B depends on: Alice's on_call status
  
These dependencies form a cycle → Serialization conflict!
  
Database aborts one transaction (usually the later one)
Application must retry ♻️
```

---

### Method 2: Explicit Locking (Materialize Conflicts)

**Create a dummy row to lock.**

```md
Problem: No single row to lock (Alice and Bob update different rows)

Solution: Lock a shared resource
```

```sql
-- Create a lock row
CREATE TABLE on_call_locks (
  lock_id INT PRIMARY KEY DEFAULT 1
);

INSERT INTO on_call_locks (lock_id) VALUES (1);

-- Transaction with explicit lock
BEGIN TRANSACTION;
  -- Lock the shared resource
  SELECT * FROM on_call_locks WHERE lock_id = 1 FOR UPDATE;
  -- Now this transaction has exclusive lock 🔒
  
  -- Check constraint
  SELECT COUNT(*) FROM doctors WHERE on_call = true;
  -- Result: 2 (Alice and Bob)
  
  -- Safe to proceed
  UPDATE doctors SET on_call = false WHERE id = 'alice';
COMMIT;
-- Release lock 🔓

-- Other transactions wait for this lock
```

**Pros:**

- Explicit control
- Works with any isolation level

**Cons:**

- Manual lock management
- Reduces concurrency

---

### Method 3: Database Constraints

Use CHECK constraints or triggers.

```sql
-- Constraint: At least 1 doctor on-call
CREATE TABLE doctors (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(100),
  on_call BOOLEAN
);

-- Add constraint
ALTER TABLE doctors ADD CONSTRAINT min_one_on_call 
CHECK (
  (SELECT COUNT(*) FROM doctors WHERE on_call = true) >= 1
);

-- Now, any update that violates constraint is rejected ✅
```

**Problem:** Not all databases support CHECK constraints with subqueries.

---

### Method 4: Application-Level Checking

**Re-check constraint before commit.**

```javascript
async function removeFromOnCall(doctorId) {
  const transaction = await db.beginTransaction();
  
  try {
    // Check constraint
    const onCallDoctors = await transaction.query(
      'SELECT id FROM doctors WHERE on_call = true'
    );
    
    if (onCallDoctors.length <= 1) {
      throw new Error('Cannot remove last on-call doctor');
    }
    
    // Update
    await transaction.query(
      'UPDATE doctors SET on_call = false WHERE id = ?',
      [doctorId]
    );
    
    // Re-check constraint before commit
    const finalCheck = await transaction.query(
      'SELECT COUNT(*) as count FROM doctors WHERE on_call = true'
    );
    
    if (finalCheck[0].count === 0) {
      throw new Error('Constraint violation: No doctors on-call');
    }
    
    await transaction.commit();
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
}
```

---

## 5. Serializable Isolation

### What is Serializable?

**Strongest isolation level** — guarantees that concurrent transactions produce the same result as if they ran serially (one after another).

```md
Concurrent Execution:
  Transaction A ─────┐
                     ├──→ Interleaved execution
  Transaction B ─────┘

Serializable Guarantee:
  Result is equivalent to:
    Transaction A → Transaction B  OR
    Transaction B → Transaction A
```

---

### Implementation 1: Actual Serial Execution

**Run transactions one at a time** (literally serial).

```md
Transaction Queue:
  1. Transaction A runs (blocking)
  2. Transaction A commits
  3. Transaction B runs (blocking)
  4. Transaction B commits
  
Pros:
  ✅ Simple to implement
  ✅ No concurrency bugs
  
Cons:
  ❌ Low throughput (one transaction at a time)
  ❌ Not practical for high-concurrency systems
```

**Used by:** VoltDB, Redis (single-threaded)

---

### Implementation 2: Two-Phase Locking (2PL)

**Acquire locks before accessing data, hold until commit.**

#### How 2PL Works

```md
Phase 1: Growing Phase
  - Acquire locks as needed
  - Cannot release any locks yet
  
Phase 2: Shrinking Phase
  - Release all locks at commit/rollback
  - Cannot acquire new locks
```

#### Example

```sql
BEGIN TRANSACTION;
  -- Read row → Acquire shared lock 🔓
  SELECT balance FROM accounts WHERE id = 'alice';
  
  -- Write row → Upgrade to exclusive lock 🔒
  UPDATE accounts SET balance = balance + 50 WHERE id = 'alice';
  
  -- Locks held until commit
COMMIT;
  -- Release all locks 🔓
```

**Pros:**

- Prevents all anomalies (serializable)

**Cons:**

- Low concurrency (lots of locking)
- Deadlock risk

**Deadlock Example:**

```md
Transaction A:
  T1: Lock row 1 🔒
  T2: Wait for row 2 (locked by B) ⏳
  
Transaction B:
  T1: Lock row 2 🔒
  T2: Wait for row 1 (locked by A) ⏳
  
Result: Deadlock! 💀
```

**Deadlock Resolution:**

```md
Database detects deadlock cycle
Aborts one transaction (victim)
Other transaction proceeds ✅
Application must retry aborted transaction ♻️
```

---

### Implementation 3: Serializable Snapshot Isolation (SSI)

**Optimistic approach** — detect conflicts at commit time.

#### How SSI Works

```md
1. Transactions execute using snapshot isolation
2. Database tracks read/write dependencies
3. At commit, check for serializability violations
4. If violation detected → Abort transaction
```

#### Example: On-Call Doctor

```md
Transaction A:
  T1: Read: Bob is on-call (snapshot)
  T2: Write: Set Alice off-call
  
Transaction B (concurrent):
  T1: Read: Alice is on-call (snapshot)
  T2: Write: Set Bob off-call
  
At commit:
  Database detects: Both transactions made decisions based on stale reads
  Conflict! One transaction aborted ❌
```

**Pros:**

- Higher concurrency than 2PL
- Only conflicts are aborted (not all transactions)

**Cons:**

- Transactions may abort (retry needed)
- More complex implementation

**Used by:** PostgreSQL (SERIALIZABLE), FoundationDB

---

## 6. Key Takeaways

1. **Lost Update**: Two transactions modify same value, one update is lost
   - **Prevention**: Atomic ops, SELECT FOR UPDATE, compare-and-set, CRDTs

2. **Write Skew**: Transactions read same data, write different rows, violate constraint
   - **Prevention**: Serializable isolation, explicit locking, constraints

3. **Serializable Isolation**: Strongest level, prevents all anomalies
   - **Implementations**: Serial execution, 2PL, SSI

4. **Trade-offs**:
   - Stronger isolation → Lower concurrency
   - Weaker isolation → Higher performance, more anomalies

### Choosing Prevention Method

| Method | Concurrency | Complexity | Use Case |
|--------|------------|------------|----------|
| Atomic operations | High | Low | Simple counters |
| SELECT FOR UPDATE | Low | Medium | Inventory, bookings |
| Snapshot Isolation | Medium | Medium | General purpose |
| Compare-and-set | High | Medium | Web forms, APIs |
| CRDTs | Very High | High | Distributed counters |
| Serializable | Low | High | Financial transactions |

### When to Use What Isolation Level

| Use Case | Isolation Level |
|----------|----------------|
| Banking | Serializable |
| E-commerce checkout | Repeatable Read + SELECT FOR UPDATE |
| Analytics | Snapshot Isolation |
| Social media likes | Read Committed (or eventually consistent) |
| Collaborative editing | CRDTs |
| Booking systems | Serializable or Explicit Locking |
