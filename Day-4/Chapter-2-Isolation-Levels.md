# Chapter 2: Isolation Levels & Concurrency Anomalies

## Overview

This chapter covers **isolation levels** — how databases handle concurrent transactions. We'll explore Read Committed, Snapshot Isolation, and the concurrency anomalies they prevent (dirty reads, dirty writes, non-repeatable reads, phantom reads).

---

## 1. What is Isolation?

### Definition

**Isolation** determines how concurrent transactions affect each other's visibility of data.

### Real-World Analogy: Kitchen 🍳

```md
Scenario: Two chefs preparing different dishes

No Isolation (Chaos):
  Chef A: Grabs eggs from fridge
  Chef B: Grabs same eggs
  Chef A: Cracks eggs
  Chef B: "Where are my eggs?" ❌
  
With Isolation:
  Chef A: Takes eggs, locks fridge drawer
  Chef B: Waits for Chef A to finish
  Chef A: Done, unlocks drawer
  Chef B: Gets eggs ✅
```

---

## 2. Isolation Levels Overview

SQL Standard defines 4 isolation levels (from weakest to strongest):

| Level | Dirty Read | Dirty Write | Non-Repeatable Read | Phantom Read |
|-------|-----------|-------------|---------------------|--------------|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| **Read Committed** | ❌ Prevented | ❌ Prevented | ✅ Possible | ✅ Possible |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | ❌ Prevented | ✅ Possible |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |

---

## 3. Read Committed Isolation

### What is Read Committed?

The **most common default isolation level** in databases.

**Guarantees:**
1. **No dirty reads**: Can't see uncommitted changes from other transactions
2. **No dirty writes**: Can't overwrite uncommitted changes from other transactions

---

### Dirty Reads

**Problem:** Reading data that hasn't been committed yet.

#### Example: Alice and Bob's Chocolate Budget

```md
Scenario: Shopping list app
Initial: Budget = $100, List = []

Alice's Transaction:
  T1: BEGIN
  T2: Spend $50 on chocolates
  T3: Budget = $100 - $50 = $50 (NOT COMMITTED)
  T4: ... (still processing)
  
Bob's Transaction (runs concurrently):
  T1: BEGIN
  T2: Check budget
  T3: Sees budget = $50 ← Dirty read! ❌
  T4: Decides not to buy cookies (thinks budget is low)
  T5: COMMIT
  
Alice's Transaction:
  T5: Error occurs, ROLLBACK
  T6: Budget restored to $100
  
Result: Bob saw $50 (uncommitted), made wrong decision ❌
```

#### SQL Example:

```sql
-- Alice's transaction
BEGIN TRANSACTION;
  UPDATE budget SET amount = amount - 50 WHERE user_id = 'alice';
  -- amount = 50 (uncommitted)
  -- ... slow processing ...
ROLLBACK; -- Oops, error occurred

-- Bob's transaction (with dirty read)
BEGIN TRANSACTION;
  SELECT amount FROM budget WHERE user_id = 'alice';
  -- Sees amount = 50 ❌ (Alice's uncommitted change)
COMMIT;
```

#### Impact: Bank Example

```md
Alice transfers $1000 to Bob:
  1. Deduct $1000 from Alice → Balance: $0 (uncommitted)
  2. Alice checks balance (dirty read) → Sees $0
  3. Alice panics: "My money is gone!" 🚨
  4. Add $1000 to Bob
  5. COMMIT
  
Problem: Alice saw intermediate state (dirty read) ❌
```

---

### How Read Committed Prevents Dirty Reads

**Solution: Show only committed data.**

```md
Alice's Transaction:
  T1: BEGIN
  T2: UPDATE budget SET amount = 50
  T3: (not committed)
  
Bob's Transaction:
  T1: BEGIN
  T2: SELECT amount FROM budget
  T3: Database returns OLD value ($100) ✅
  T4: COMMIT
  
Alice's Transaction:
  T5: COMMIT
  
Now Bob sees new value ($50) ✅
```

**Implementation (Row Versioning):**

```md
Database keeps multiple versions:
  Version 1 (committed): amount = $100
  Version 2 (uncommitted): amount = $50
  
Bob's query sees only Version 1 (committed) ✅
```

---

### Dirty Writes

**Problem:** Overwriting another transaction's uncommitted changes.

#### Example: Alice and Bob Update Same Row

```md
Initial: A = 5

Alice's Transaction:
  T1: BEGIN
  T2: SET A = 10 (uncommitted)
  
Bob's Transaction:
  T1: BEGIN
  T2: SET A = 15 (overwrites Alice's uncommitted change) ❌
  T3: COMMIT
  
Alice's Transaction:
  T3: ROLLBACK
  
What should A be?
  - Alice expects: 5 (rollback)
  - Bob expects: 15 (committed)
  - Actual: 15 ← Bob's write, but based on dirty data! ❌
```

#### Danger: Lost Updates

```sql
-- Alice's transaction
BEGIN TRANSACTION;
  UPDATE products SET stock = 10 WHERE id = 'laptop';
  -- stock = 10 (uncommitted)
  
-- Bob's transaction
BEGIN TRANSACTION;
  UPDATE products SET stock = 5 WHERE id = 'laptop';
  -- Overwrites Alice's uncommitted change ❌
  COMMIT;
  
-- Alice's transaction
ROLLBACK;

-- Result: stock = 5 (Bob's value)
-- But Alice's change was never committed!
-- This is confusing and error-prone ❌
```

---

### How Read Committed Prevents Dirty Writes

**Solution: Lock rows being modified.**

```md
Alice's Transaction:
  T1: BEGIN
  T2: UPDATE products SET stock = 10 → Acquire lock 🔒
  
Bob's Transaction:
  T1: BEGIN
  T2: UPDATE products SET stock = 5 → Wait for lock ⏳
  
Alice's Transaction:
  T3: COMMIT → Release lock 🔓
  
Bob's Transaction:
  T2: Acquire lock, apply update ✅
  T3: COMMIT
```

---

## 4. Concurrency Anomalies in Read Committed

### Non-Repeatable Reads

**Problem:** Reading same data twice within transaction yields different results.

#### Example: Alice's Bank Balance

```md
Alice's Transaction:
  T1: BEGIN
  T2: SELECT balance FROM accounts WHERE id = 'alice';
      Result: $1000
  T3: ... (some computation)
  
Bob's Transaction (runs concurrently):
  T1: BEGIN
  T2: UPDATE accounts SET balance = 500 WHERE id = 'alice';
  T3: COMMIT
  
Alice's Transaction:
  T4: SELECT balance FROM accounts WHERE id = 'alice';
      Result: $500 ← Different from T2! ❌
  T5: COMMIT
  
Problem: Alice saw $1000 then $500 (non-repeatable read)
```

#### Real-World Impact: Database Backup

```md
Backup Process:
  T1: Backup users table
  T2: (1 hour passes)
  T3: Backup orders table
  
Meanwhile, user transaction:
  T1: Transfer $100 from users to orders
  T2: COMMIT
  
Backup sees:
  - Users: $100 deducted ✅
  - Orders: $100 not yet added ❌ (backed up before update)
  
Result: Backup shows $100 missing! ❌
```

---

### Phantom Reads

**Problem:** New rows appear/disappear between queries.

#### Example: Product Count

```md
Alice's Transaction:
  T1: BEGIN
  T2: SELECT COUNT(*) FROM products WHERE category = 'laptops';
      Result: 10
  T3: ... (some processing)
  
Bob's Transaction:
  T1: BEGIN
  T2: INSERT INTO products (name, category) VALUES ('New Laptop', 'laptops');
  T3: COMMIT
  
Alice's Transaction:
  T4: SELECT COUNT(*) FROM products WHERE category = 'laptops';
      Result: 11 ← Phantom row appeared! ❌
  T5: COMMIT
```

---

### Difference: Non-Repeatable vs Phantom

```md
Non-Repeatable Read:
  - Existing row CHANGED (UPDATE/DELETE)
  - Example: Balance changed from $1000 to $500
  
Phantom Read:
  - New row APPEARED or DISAPPEARED (INSERT/DELETE)
  - Example: Count changed from 10 to 11
```

---

## 5. Snapshot Isolation

### What is Snapshot Isolation?

Transaction sees a **consistent snapshot** of the database at start time — like taking a photograph.

### How It Works

```md
Transaction Start:
  1. Take snapshot of database state
  2. All reads see data as of snapshot time
  3. Ignores changes from other transactions
  
Timeline:
  T0: Snapshot taken → Database state frozen for this transaction
  T1: Other transactions modify data
  T2: This transaction reads → Still sees T0 snapshot ✅
  T3: This transaction commits
```

### Example: Alice's Bank Balance (Fixed)

```md
Alice's Transaction:
  T0: BEGIN → Snapshot taken (balance = $1000)
  T1: SELECT balance → Sees $1000 (from snapshot)
  T2: ... (computation)
  
Bob's Transaction:
  T0: BEGIN
  T1: UPDATE balance = $500
  T2: COMMIT
  
Alice's Transaction:
  T3: SELECT balance → Still sees $1000 (snapshot) ✅
  T4: COMMIT
  
Result: Consistent view throughout transaction ✅
```

---

### Multi-Version Concurrency Control (MVCC)

**How databases implement Snapshot Isolation.**

#### How MVCC Works

Database keeps **multiple versions** of each row.

```md
Row Versions:
  Version 1: {id: 1, name: "Alice", created_by: TX-100, deleted_by: NULL}
  Version 2: {id: 1, name: "Alice Smith", created_by: TX-200, deleted_by: NULL}
  
Transaction TX-150 (started between TX-100 and TX-200):
  - Sees Version 1 (created before TX-150)
  - Ignores Version 2 (created after TX-150)
  
Transaction TX-250 (started after TX-200):
  - Sees Version 2 ✅
```

#### Visibility Rules

```md
Transaction T reads row version V:
  V is visible if:
    1. V.created_by < T.start_time (created before T started)
    2. V.deleted_by is NULL OR V.deleted_by > T.start_time
  
Example:
  T100 creates row → created_by = 100
  T200 reads row (started at 150)
    → Checks: 100 < 150? Yes ✅
    → Sees row
  
  T300 deletes row → deleted_by = 300
  T400 reads row (started at 250)
    → Checks: 300 > 250? Yes ✅
    → Sees row (deletion happened after T400 started)
```

---

### Real-World Example: PostgreSQL MVCC

```sql
-- Create table
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance INT
);

INSERT INTO accounts VALUES (1, 1000);

-- Transaction 1 (started at T=100)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  -- Snapshot taken at T=100
  SELECT balance FROM accounts WHERE id = 1;
  -- Result: 1000
  
-- Transaction 2 (runs concurrently at T=150)
BEGIN TRANSACTION;
  UPDATE accounts SET balance = 500 WHERE id = 1;
  COMMIT; -- T=200
  
-- Transaction 1 continues at T=250
  SELECT balance FROM accounts WHERE id = 1;
  -- Result: 1000 (still sees snapshot from T=100) ✅
COMMIT;

-- New transaction at T=300
BEGIN TRANSACTION;
  SELECT balance FROM accounts WHERE id = 1;
  -- Result: 500 (sees latest committed value) ✅
COMMIT;
```

---

### Benefits of Snapshot Isolation

#### 1. **Consistent Backups**

```md
Backup starts at T=0:
  - Takes snapshot of entire database
  - Backup sees consistent state (all at T=0)
  - Even if transactions modify data during backup
  
Result: Consistent backup ✅
```

#### 2. **Long-Running Analytics**

```md
Analytics query:
  - Runs for 1 hour
  - Sees consistent data throughout
  - Not affected by concurrent updates
  
Result: Accurate analytics ✅
```

#### 3. **No Blocking Reads**

```md
Readers don't block writers, writers don't block readers
  - High concurrency ✅
  - Better performance
```

---

## 6. Implementation: MVCC Indexing

### Challenge: Indexes with Multiple Versions

```md
Row has 3 versions:
  V1: {id: 1, name: "Alice"}
  V2: {id: 1, name: "Alice Smith"}
  V3: {id: 1, name: "Alice Johnson"}
  
Index on name:
  - "Alice" → V1
  - "Alice Smith" → V2
  - "Alice Johnson" → V3
  
Problem: Index grows with versions 📈
```

### Solution 1: Index All Versions (PostgreSQL)

```md
B-Tree index contains all versions:
  "Alice" → [V1]
  "Alice Smith" → [V2]
  "Alice Johnson" → [V3]
  
On lookup:
  1. Find matching versions
  2. Apply visibility rules
  3. Return visible version
  
Cleanup: VACUUM removes old versions
```

---

### Solution 2: Append-Only B-Trees (CouchDB)

```md
Never update index in-place:
  - Create new version of index for each update
  - Old versions kept for transactions that need them
  - Garbage collect old versions later
  
Advantages:
  - No locking (append-only)
  - Easy snapshots
  
Disadvantages:
  - More disk space
  - Periodic compaction needed
```

---

## 7. Repeatable Read vs Snapshot Isolation

### Naming Confusion

```md
SQL Standard:
  "Repeatable Read" defined (prevents non-repeatable reads)
  
Real World:
  - PostgreSQL "Repeatable Read" = Snapshot Isolation ✅
  - MySQL "Repeatable Read" = Snapshot Isolation ✅
  - Oracle "Serializable" = Snapshot Isolation ❌ (confusing!)
  
Advice: Read your database documentation! 📖
```

### PostgreSQL Example

```sql
-- PostgreSQL calls it "Repeatable Read" but it's Snapshot Isolation
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT * FROM users WHERE id = 1;
  -- Sees consistent snapshot ✅
COMMIT;
```

---

## 8. Real-World Use Cases

### Use Case 1: Database Backup

```md
Requirement: Consistent backup while database is live

Solution: Snapshot Isolation
  1. Start backup transaction
  2. Take snapshot
  3. Backup all tables (sees consistent state)
  4. Commit
  
Tools: mysqldump, pg_dump use snapshots
```

---

### Use Case 2: Analytics Query

```md
Requirement: Calculate monthly revenue (takes 10 minutes)

Problem with Read Committed:
  - Sales happening during query
  - Inconsistent results
  
Solution: Snapshot Isolation
  - Query sees data as of start time
  - Consistent results ✅
```

---

### Use Case 3: Multi-Step Workflow

```md
Scenario: Approve multiple purchase orders

Workflow:
  1. Read PO list
  2. Check budget for each
  3. Approve POs
  
Problem with Read Committed:
  - POs might change between steps
  - Budget might change
  
Solution: Snapshot Isolation
  - Consistent view throughout workflow ✅
```

---

## 9. Limitations of Snapshot Isolation

### Write Skew (Covered in next chapter)

Snapshot Isolation does NOT prevent all anomalies.

```md
Example: On-call doctor constraint
  - Rule: At least 1 doctor must be on-call
  - Current: Alice and Bob both on-call
  
Alice's transaction:
  1. Check: Bob is on-call ✅
  2. Remove self from on-call
  
Bob's transaction (concurrent):
  1. Check: Alice is on-call ✅
  2. Remove self from on-call
  
Both commit:
  Result: 0 doctors on-call! ❌ Constraint violated
```

**Solution:** Use Serializable isolation (next topic).

---

## 10. Key Takeaways

1. **Isolation levels** trade off consistency vs performance
2. **Read Committed** (most common):
   - ✅ Prevents dirty reads, dirty writes
   - ❌ Allows non-repeatable reads, phantom reads
3. **Snapshot Isolation**:
   - ✅ Prevents non-repeatable reads
   - ✅ Provides consistent snapshot
   - ✅ Great for backups, analytics
   - ❌ Still allows some anomalies (write skew)
4. **MVCC** enables Snapshot Isolation:
   - Multiple row versions
   - Visibility rules based on transaction timestamps
5. **Read your database docs** — naming varies!

### Choosing Isolation Level

| Use Case | Isolation Level | Reason |
|----------|----------------|---------|
| Banking transactions | Serializable | Strongest guarantees |
| E-commerce checkout | Repeatable Read | Balance consistency & performance |
| Database backup | Snapshot Isolation | Consistent point-in-time |
| Analytics | Snapshot Isolation | Long-running queries |
| Simple CRUD | Read Committed | Good enough, fast |
| High throughput | Read Committed | Less locking overhead |
