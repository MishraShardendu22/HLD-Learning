# LSM Tree (Log-Structured Merge Tree)

LSM trees are designed to **optimize write throughput** by turning random writes into **sequential writes**, and paying the cost later during reads and compaction.

Used when:

* write volume is high
* data is continuously updated
* scale matters more than low-latency point reads

---

## 1. Structure of an LSM Tree

LSM has **two layers**:

1. **In-memory layer** (fast writes)
2. **Disk layer** (immutable, sorted files)

Writes flow **top → bottom**.

---

## 2. Memtable

### What it is

* In-memory data structure
* Stores recent writes
* Sorted by key

### Why it exists

* RAM writes are fast
* Avoids random disk writes

### Real-World Example

Think of a **notebook at a restaurant**:

* Orders written quickly
* Later copied into a register system

---

## 3. Red-Black Tree (Memtable Implementation)

### Why Red-Black Tree

* Self-balancing binary search tree
* Guarantees O(log n) insert/search
* Maintains sorted order

### Why not HashMap

* LSM needs sorted keys for:

  * range scans
  * efficient flushing to disk

### Real-World Analogy

A **sorted filing cabinet**, not a dump box.

---

## 4. SSTable (Sorted String Table)

### What it is

* Immutable disk file
* Contains sorted key-value pairs
* Written sequentially

### Properties

* Never modified in place
* Once written, read-only

### Real-World Example

A **printed ledger**:

* Cannot be edited
* New version printed instead

---

## 5. Levels in LSM

### Why Levels Exist

Without structure, SSTables grow uncontrollably.

### How Levels Work

* Level 0: freshly flushed SSTs
* Lower levels:

  * fewer files
  * larger files

Each level:

* size increases exponentially
* has stricter ordering

### Example

Like **warehouse floors**:

* ground floor = small boxes
* basement = huge pallets

---

## 6. Compaction

### What it is

Background process that:

* merges SSTables
* removes duplicates
* applies deletes

### Why Needed

* Reads check multiple SSTs
* Disk usage explodes
* Old values remain

### How It Works

1. Pick SSTs from a level
2. Merge + sort
3. Write new SST
4. Delete old ones

### Real-World Example

Monthly **account reconciliation**:

* merge statements
* discard outdated entries

---

## 7. Making LSM Reliable

LSM is fast but needs protection against crashes and read amplification.

---

### a) Write-Ahead Log (WAL)

#### What it is

* Every write logged to disk **before** memtable update

#### Purpose

* Crash recovery

#### Recovery

* Replay WAL
* Rebuild memtable

#### Analogy

Cash register **receipt roll**:

* proof of every transaction

---

### b) Bloom Filters

#### What they do

* Probabilistic structure
* Answers: “Key **definitely not** here?”

#### Why Important

* Avoid unnecessary disk reads
* Reduce read amplification

#### Trade-off

* False positives allowed
* No false negatives

#### Real-World Example

Airport **security pre-check**:

* quickly discard impossible candidates

---

### c) Garbage Collection

#### Purpose

* Remove:

  * overwritten values
  * tombstones (deleted keys)

Done during compaction.

Without GC:

* disk fills
* reads slow

---

## 8. Real-World Use Cases of LSM Trees

---

### a) High Write Systems

Examples:

* user activity logs
* metrics ingestion
* clickstreams

Why LSM:

* handles millions of writes/sec
* sequential I/O

---

### b) Embedded Storage Engines

Examples:

* RocksDB
* LevelDB

Used inside:

* Kafka
* Flink
* Redis (disk-backed)

Reason:

* predictable performance
* no external DB dependency

---

### c) Real-Time Search & Analytics

Examples:

* Elasticsearch
* OpenSearch

Why:

* continuous ingestion
* near-real-time queries
* eventual consistency acceptable

---

### d) Key-Value Stores

Examples:

* Cassandra
* HBase
* DynamoDB (LSM-inspired)

Why:

* write-heavy workloads
* distributed storage

---

## Key Trade-offs (Important)

**Pros**

* Extremely fast writes
* Crash-safe
* Scales horizontally

**Cons**

* Read amplification
* Write amplification
* Compaction overhead

---
