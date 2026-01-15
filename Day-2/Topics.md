# High-Level Design (HLD) Topics - Day 2

## Overview

This file summarizes the key HLD topics covered in Day-2, focusing on data warehousing, encoding/serialization, communication patterns, and distributed systems. Each section is cross-referenced to detailed chapter files for in-depth learning.

---

## Video 1-2 — OLTP vs OLAP & Data Warehousing (see [Chapter-1](Chapter-1-OLTP-vs-OLAP-DataWarehousing.md)) 💾

### OLTP (Online Transaction Processing)

- Transaction processing systems
- Main read/write patterns: small, frequent transactions
- Users: End users (thousands to millions)
- Data: Current operational data, normalized schema
- Volume: High transaction volume, low latency requirements

### OLAP (Online Analytical Processing)

- Analytical query processing
- Main read/write patterns: Complex aggregations, large scans
- Users: Business analysts, data scientists (dozens)
- Data: Historical data, denormalized schema
- Volume: Large data per query (GB-TB), read-heavy

### Hybrid Model (HTAP)

- Combines OLTP and OLAP on same database
- Real-time analytics on transactional data
- Challenges: Resource contention, complexity

### Data Warehousing

- Centralized repository for analytical queries
- Difference between OLTP and data warehouse: Purpose, schema, storage, users
- Benefits: Performance isolation, data integration, historical analysis, optimized analytics
- Tools for building: Redshift, BigQuery, Snowflake, Hive, ClickHouse
- Challenges: Data quality, ETL complexity, latency, scalability, schema evolution, cost

### Column-Oriented Storage

- Stores data by column (not row) for OLAP efficiency
- Compression techniques:
  - **Run-Length Encoding (RLE)**: Compress repeated values
  - **Dictionary Encoding**: Replace strings with numeric IDs
- Benefits: Better compression, reduced I/O, vectorized processing
- Use cases: Data warehouses, analytics databases, Parquet/ORC formats

---

## Video 3-7 — Data Encoding & Schema Evolution (see [Chapter-2](Chapter-2-Data-Encoding-Schema-Evolution.md)) 📝

### How Databases Handle Changes

1. **Relational DB (Schema-on-Write)**: Strict schema enforcement, migrations required
2. **Schema-on-Read (NoSQL)**: Flexible, application-level validation

### Rolling Updates / Staged Rollouts

1. **Server-side**: Gradual deployment across servers
2. **Client-side**: Mobile/desktop apps update at different times

### Compatibility

- **Backward compatibility**: New code reads old data
- **Forward compatibility**: Old code reads new data
- Critical for zero-downtime deployments

### Why Encoding/Decoding?

- Convert in-memory structures to bytes for storage/network
- Enable cross-language, cross-platform communication

### Encoding Formats & Problems

1. **JSON**: Human-readable, flexible, but verbose and type-ambiguous
2. **XML**: Self-documenting but extremely verbose
3. **Binary**: Compact and fast (Protobuf, Thrift, Avro)
4. **CSV**: Simple tabular data, but no schema, escaping issues

### Real Problem Example

- **Twitter user ID issue**: JSON couldn't handle large integers (>2^53)
- Solution: Send IDs as both number and string

### Schema Importance

- Enforces structure and consistency
- Benefits: Compact data, self-documentation, code generation
- Challenges: Learning curve, schema evolution management

---

## Video 4-6 — Binary Serialization Formats (see [Chapter-3](Chapter-3-Binary-Serialization-Formats.md)) 🔧

### Thrift vs Protobuf

- **Protobuf**: Google's format, field numbers for encoding, widely adopted
- **Thrift**: Facebook's format, required/optional modifiers, built-in RPC

### Apache Avro

- **Avro's IDL**: JSON-based schema definition language
- **Writer & Reader Schema**: Schema resolution at read time (unique to Avro)
- **Why Avro**: Dynamic schemas, excellent schema evolution, Hadoop/Kafka ecosystem

### Compatibility in Binary Formats

- **Backward**: New code reads old data (add fields with defaults)
- **Forward**: Old code reads new data (ignore unknown fields)
- **Never change field numbers** in Protobuf/Thrift

### Key Differences

- **Protobuf**: Field numbers, fastest, gRPC support
- **Thrift**: Field IDs, multiple protocols, built-in RPC
- **Avro**: Field names, schema registry, best for data pipelines

### Real-World Use Cases

- **Protobuf**: Microservices, gRPC APIs, Kubernetes
- **Thrift**: Facebook, Twitter internal services
- **Avro**: Kafka streams, Hadoop/Spark jobs, data lakes

### Limitations

- Not human-readable (need tools)
- Schema distribution required
- Schema evolution discipline needed

---

## Video 8-10 — Data Flow & Communication (see [Chapter-4](Chapter-4-Data-Flow-Communication.md)) 🌐

### Modes of Data Flow

1. **Via Databases**: Asynchronous, persistent
2. **Via Service Calls**: Synchronous (RPC, REST)
3. **Via Message Passing**: Asynchronous (queues, pub-sub)

### Web Services & Protocols

- **What**: Enable systems to communicate over networks
- **Protocols**: HTTP/REST, gRPC, WebSockets, AMQP
- **Why important**: Integration, microservices, API economy

### Types of Communication

1. **Synchronous**: Caller waits (RPC, REST) — simple but blocking
2. **Asynchronous**: Caller continues (message queues) — scalable, complex
3. **One-Way**: Fire-and-forget (logging, metrics)
4. **Two-Way**: Request-response (APIs)

### Real-World Examples

- **OTP verification**: Synchronous (user waits)
- **Email processing**: Asynchronous (background job)
- **Notifications**: Pub-sub (event-driven)

### Publisher-Subscriber Model

- **Event-driven communication**: Loose coupling, scalability
- **Broker**: Kafka, RabbitMQ, AWS SNS
- **Benefits**: Decoupling, flexibility, reliability

### RPC (Remote Procedure Call)

- **What**: Make remote calls look like local function calls
- **How it works**: Serialization, network transport, deserialization
- **Pizza analogy**: Call pizza place instead of cooking at home 🍕
- **Benefits**: Abstraction, simplified communication, scalability
- **Challenges**: Network dependency, failure handling, latency, versioning

### Asynchronous Messaging & Message Queues

- **Benefits**: Decoupling, load leveling, reliability, fault tolerance
- **Example**: E-commerce order processing (instant response, background work)
- **Tools**: RabbitMQ (queues), Kafka (streams), SQS (managed), Redis Streams

### Actor Model

- Independent actors with isolated state
- Communicate via async messages
- Use cases: Erlang (WhatsApp), Akka (JVM), gaming, IoT

---

## Video 11-12 — Replication & Distributed Systems (see [Chapter-5](Chapter-5-Replication-Distributed-Systems.md)) 🔄

### What is Replication?

- Maintaining multiple copies of data on different nodes
- **Why**: Reduced latency, high availability, scalability
- **Coffee shop analogy**: Multiple locations serve more customers ☕

### Leader-Based Replication (Master-Slave)

- **Leader**: Handles all writes
- **Followers**: Replicate data, handle reads
- **How it works**: Leader propagates changes to followers

### Writes vs Reads

- **Writes**: Always to leader (consistency)
- **Reads**: Leader or followers (load distribution)

### Synchronous Replication

- Leader waits for follower ACK
- **Group chat analogy**: Wait for "✓✓" read receipt 💬
- **Pros**: Strong durability, consistency
- **Cons**: Higher latency, availability risk

### Asynchronous Replication

- Leader doesn't wait for followers
- **Email analogy**: Instant "Sent" confirmation 📧
- **Pros**: Low latency, high availability
- **Cons**: Potential data loss, replication lag

### Failover & Node Failure

- **What happens when leader fails**: Automatic promotion of follower
- **Head chef analogy**: Cooks elect new head chef 👨‍🍳
- **Failover process**:
  1. Detect failure (timeout mechanism, heartbeats)
  2. Leader election (consensus, priority, coordinator)
  3. Reconfigure system (update routing)
  4. Catch-up recovery (old leader rejoins as follower)

### Challenges

- **Unreplicated writes**: Data loss if leader crashes before replicating
- **Split-brain**: Multiple leaders (network partition)
- **Timeout tuning**: Balance false positives vs downtime
- **Replication lag**: Followers behind leader (stale reads)

### How Systems Recover

- **Catch-up recovery**: Replay missed changes from replication log
- **Smooth recovery**: No service disruption for users

---

## Related Chapter Files

- [Chapter-1-OLTP-vs-OLAP-DataWarehousing.md](Chapter-1-OLTP-vs-OLAP-DataWarehousing.md) — OLTP, OLAP, data warehousing, column storage
- [Chapter-2-Data-Encoding-Schema-Evolution.md](Chapter-2-Data-Encoding-Schema-Evolution.md) — Encoding formats, schema evolution, compatibility
- [Chapter-3-Binary-Serialization-Formats.md](Chapter-3-Binary-Serialization-Formats.md) — Protobuf, Thrift, Avro deep dive
- [Chapter-4-Data-Flow-Communication.md](Chapter-4-Data-Flow-Communication.md) — RPC, REST, message queues, pub-sub, actor model
- [Chapter-5-Replication-Distributed-Systems.md](Chapter-5-Replication-Distributed-Systems.md) — Replication strategies, failover, distributed systems

---

## Quick Reference

### When to Use What?

| Scenario | Choose |
|----------|--------|

| Transactional workload | OLTP database (PostgreSQL, MySQL) |
| Analytical workload | OLAP / Data warehouse (Redshift, BigQuery) |
| Public REST API | JSON (human-readable, accessible) |
| Internal microservices | Protobuf + gRPC (fast, type-safe) |
| Kafka data pipelines | Avro (schema evolution, registry) |
| User-facing request | Synchronous (RPC/REST) |
| Background task | Asynchronous (message queue) |
| Strong consistency | Synchronous replication |
| Low latency writes | Asynchronous replication |

---

> **Note**: Each chapter file contains detailed explanations, examples, real-world analogies, and practical use cases. Start with the Topics file for overview, then dive into specific chapters for comprehensive understanding. 🚀
