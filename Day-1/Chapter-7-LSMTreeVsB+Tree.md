# B-Trees vs LSM Trees — Detailed Explanation

Both are **storage engines**, not databases. They solve the same problem differently.

---

## 1. Core Philosophy Difference

### B-Tree

* Optimized for **reads**
* Keeps data **sorted on disk**
* Updates happen **in place**

### LSM Tree

* Optimized for **writes**
* Uses **append-only logs**
* Updates are merged later

---

## 2. LSM vs B-Tree (Big Picture)

| Aspect        | B-Tree        | LSM Tree           |
| ------------- | ------------- | ------------------ |
| Write pattern | Random I/O    | Sequential I/O     |
| Read pattern  | Direct lookup | Multi-level lookup |
| Update style  | In-place      | Append + merge     |
| Best for      | Read-heavy    | Write-heavy        |

---

## 3. Writes — Why LSM is Faster

### B-Tree Write Path

1. Find leaf node
2. Modify page
3. Possibly split page
4. Update parent nodes
5. Write pages back to disk

This causes:

* random disk writes
* page locking

### Real-World Example

Editing a **book**:

* erase page
* rewrite page
* fix table of contents

---

### LSM Write Path

1. Append to WAL
2. Insert into memtable
3. Sequential flush to disk
4. Background compaction

No random I/O on write path.

### Real-World Example - Keeping a **daily journal**

* always write at the end
* organize later

---

## 4. Reads — Why B-Trees Are Faster

### B-Tree Reads

* One tree traversal
* O(log n) page reads
* Data found immediately

### Example

Library catalog:

* go directly to shelf

---

### LSM Reads

* Check memtable
* Check multiple SSTables
* Possibly scan several files

Mitigated using:

* Bloom filters
* Index blocks

### Example - Searching multiple notebooks for one note

---

## 5. Storage Efficiency

### B-Trees

* Compact
* No duplicate versions
* Minimal disk usage

### LSM Trees

* Multiple versions until compaction
* Tombstones
* Temporary disk bloat

Example:

* keeping old receipts until audit

---

## 6. Write Amplification

### Definition

Amount of **extra data written** per logical write.

### B-Tree -

* One logical write → multiple page writes
* Moderate write amplification

### LSM Tree -

* Data rewritten multiple times across levels
* High write amplification

But:

* sequential I/O is cheaper

---

## 7. Compaction (LSM-Only)

### What It Does

* Merge SSTables
* Remove old versions
* Apply deletes

### Cost

* CPU + I/O heavy
* Can impact latency

### Example -

Re-organizing warehouse inventory at night.

---

## 8. Transactional Semantics

### B-Tree Databases

* Strong **ACID**
* Multi-row transactions
* Strict isolation

Used in:

* Banking
* Payments
* Inventory

---

### LSM-Based Databases

* Usually **eventual consistency**
* Limited transactions
* Atomicity per key

Used in:

* Metrics
* Logs
* Feeds

---

## 9. When to Use What (Real Systems)

### Use B-Tree When

* Read-heavy workload
* Strong consistency needed
* Complex queries

Examples:

* MySQL
* PostgreSQL

---

### Use LSM Tree When

* Write-heavy workload
* Horizontal scaling
* Append-only data

Examples:

* Cassandra
* HBase
* RocksDB
* Elasticsearch

---

## Final One-Line Rule

> **B-Trees pay the cost on writes to make reads cheap.
> LSM Trees pay the cost on reads to make writes cheap.**

---
