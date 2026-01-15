# Chapter 5: Indexing

## How Databases Store Data (Core Idea)

Databases are fundamentally about **trade-offs between write speed, read speed, and storage structure**.

Two main storage approaches:

1. **Log-Structured Storage**:
2. **Page-Oriented Storage (B+ Trees)**:

---

## 1. Log-Structured Storage Database (LSM-based)

### Structure

* Data is **appended sequentially** to a log file.
* No in-place updates.
* Writes go to the end of the file.

### Sequential Writes

* **Fast writes** (disk-friendly)
* **Slow search** (must scan logs or use indexes)

Why fast:

* Disks are optimized for sequential writes.
* No seek cost.

---

## 2. Page-Oriented Database (B+ Trees)

* Disk divided into fixed-size **pages**:
* Records stored inside pages
* Pages linked via **B+ Tree index**:

### Characteristics

* **Fast reads**:
* **Slower writes** (page splits, random I/O)

Why slower writes:

* Updating data may require:

  * page modification
  * page split
  * index rebalance

---

## Indexing Trade-off

> Indexing speeds up reads but slows down writes.

Reason:

* Every write must update:

  * data page
  * index page(s)

---

## Types of Indexing

### 1. Hash Index

* Key → hash function → bucket

**Pros**:

* O(1) lookup
* Extremely fast exact-key search

**Cons**:

* No ordering
* No range queries

---

### 2. B+ Tree Index

* Ordered, balanced tree
* Keys sorted

**Pros**:

* Range queries
* Prefix searches
* Disk-friendly

**Cons**:

* Slower writes than hash
* Tree rebalancing

---

### 3. LSM Tree (Log-Structured Merge Tree)

Used in:

* Cassandra
* LevelDB
* RocksDB

**Idea**:

* Writes go to memory (memtable)
* Periodically flushed to disk (SSTables)
* Background merge (compaction)

Optimized for:

* write-heavy workloads

---

## How Hashing Works

1. Input key
2. Hash function computes hash value
3. Hash maps to bucket location
4. Value stored/retrieved

Problems:

* collisions
* resizing cost
* no ordering

---

## Improvements – Bitcask (Log-Structured KV Store)

### Core Idea

* Append-only log
* In-memory hash index → file offset

Read = hash lookup + single disk seek.

---

## Compaction

Process of merging logs to remove:

* stale values
* deleted records

### Why Needed

Because append-only logs grow endlessly.

---

### Compaction Solves

#### 1. Deletion Records

* Deletes are written as **tombstones**:
* Compaction removes old values permanently

#### 2. Crash Recovery

* On restart:

  * replay log
  * rebuild in-memory index
* Log is source of truth

#### 3. Concurrency

* Writes are append-only → minimal locking
* Reads use immutable files

---

## Limitations of Log-Structured / Hash-Based Storage

### Memory Limit

* Hash index often kept in memory
* Large datasets → high RAM usage

---

### No Range Queries

* Keys not ordered
* Cannot do:

  * `BETWEEN`
  * prefix scans
  * ordered traversal

---

## Pros of Log-Structured Storage

* **Very fast writes**:
* **Excellent for frequent updates**:
* Sequential disk I/O
* Crash-safe by design

---
