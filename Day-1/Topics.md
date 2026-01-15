# High-Level Design (HLD) Topics

## Overview -

This file summarizes the key HLD topics covered across the chapter files. Each section is grouped by lecture/video and cross-referenced to the relevant chapter where applicable.

---

### Video 1 — Scalability (see `Chapter-1-Scalibility.md`) ✅

- Vertical vs horizontal scaling: pros and cons
- Cloud scalability: elasticity, scaling strategies
- CAP theorem (when to apply)
- Load balancing and health checks
- Monolithic vs microservices architecture
- CDN for geo-distribution and caching
- Serverless computing and autoscaling
- Stateless design principles
- Database optimization for scalability
- Monitoring and alerting (observability)

---

### Video 2 — Reliability (see `Chapter-2-Reliability.md`) ⚠️

- Outage types and failure modes
- Reliability measures: SLA, SLO, SLI
- Key metrics: latency, error rate, throughput
- Redundancy and failover mechanisms
- Load balancers, health checks, and monitoring
- Replication strategies and consistency trade-offs
- Data partitioning and sharding
- Caching for availability and performance
- Advanced topics: distributed systems, CAP theorem, eventual consistency
- Testing for reliability: chaos engineering (Chaos Monkey)

---

### Video 3 — Maintainability (see `Chapter-3-Maintainibility.md`) 🔧

- Core principles: modularity, abstraction, encapsulation
- Clear data models and schema design
- Testability and automated testing
- Observability: Prometheus, logging, tracing
- Tools and practices: version control, CI/CD, code review, monitoring

---

### Video 4 — Data Model (see `Chapter-4-Database.md`) 💡

- Data model varieties: network, hierarchical, XML
- Relational model and SQL; ACID vs BASE
- NoSQL models: key-value, document, column, graph
- Evolving datatypes and relationship changes
- Object-relational impedance mismatch (O/R mismatch)
- Relationship cardinality: 1:N, N:1, M:N
- Document databases and revisiting network models

#### How databases store data

- Log-structured storage (append-only): fast writes, slower random reads
- Page-oriented storage (B+ trees): balanced read/write trade-offs
- Indexing: speeds up reads but can slow writes
- Index types: hash, B+ tree, LSM-tree
- Hashing: basics and limitations (no natural range queries)
- Improvements and approaches: Bitcask-style, compaction strategies
  - Handling deletions, crash recovery, concurrency
- Limitations: memory bounds, range-query support
- Use cases: fast-write workloads, frequent updates

---

### Video 5 — LSM Tree (see `Chapter-6-LSMTree.md`) 🌲

- Structure: memtable (in-memory) and SSTables (on-disk)
- Common memtable implementations (skip lists, balanced trees)
- SSTable levels and compaction strategy
- Compaction types and their trade-offs
- Reliability features: WAL (write-ahead log), Bloom filters, GC
- Real-world use cases: high-write systems, key-value stores, embedded engines, real-time analytics

---

### Video 6 — B-Trees vs LSM Trees (see `Chapter-7-LSMTreeVsB+Tree.md`) 🔁

- Read/write patterns: B-trees favor reads, LSM favors writes
- Storage efficiency and layout differences
- Write amplification and compaction costs (LSM)
- Concurrency and on-disk layout implications
- Transactional semantics and recovery strategies

---

### Video 7 — Indexing & Search (see `Chapter-5-Indexing.md`) 🔎

- Primary vs secondary indexes
- Clustered vs non-clustered (heap file, clustered index)
- Covering indexes and storing values in indexes
- Multi-column indexes and multidimensional indexes
- Space-filling curves and R-trees for spatial data
- Full-text search and fuzzy matching: Lucene, Levenshtein distance
- In-memory databases and trade-offs for latency

---

## Related chapter files

- `Chapter-1-Scalibility.md` — Scalability
- `Chapter-2-Reliability.md` — Reliability
- `Chapter-3-Maintainibility.md` — Maintainability
- `Chapter-4-Database.md` — Data models & storage
- `Chapter-5-Indexing.md` — Indexing & search
- `Chapter-6-LSMTree.md` — LSM tree details
- `Chapter-7-LSMTreeVsB+Tree.md` — Comparison: LSM vs B+ Tree

---
