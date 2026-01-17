# Chapter 2: Data Encoding & Schema Evolution

## Overview

This chapter explores how databases and distributed systems handle changes over time, the need for data encoding, different serialization formats, and the critical concepts of backward and forward compatibility.

---

## 1. How Do Databases Handle Changes?

When your application evolves, you need to handle schema changes gracefully. There are two main approaches:

### 1.1 Relational Database (Schema-on-Write)

### What is Schema-on-Write?

In traditional relational databases, the **schema is enforced when data is written**. You must define the structure upfront (tables, columns, types, constraints).

#### Characteristics

- **Explicit schema**: You define tables and columns before inserting data
- **Strict validation**: Database rejects data that doesn't match the schema
- **Migrations required**: Schema changes need ALTER TABLE statements
- **Strong consistency**: All data conforms to the current schema

#### Example

```sql
-- Original schema
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

-- Adding a new column requires migration
ALTER TABLE users ADD COLUMN age INT;
```

#### Pros

- Data integrity and validation at write time
- Clear structure and documentation
- Optimized query planning

#### Cons

- Schema changes can be slow and risky on large tables
- Less flexible for evolving requirements
- Downtime or careful coordination needed for migrations

---

### 1.2 Schema-on-Read (NoSQL/Document Databases)

### What is Schema-on-Read?

The schema is **applied when data is read**, not when it's written. The database stores semi-structured data (like JSON documents) without enforcing a rigid schema upfront.

#### Characteristics

- **Flexible schema**: Each document can have different fields
- **No upfront schema**: Write data without predefined structure
- **Application-level validation**: Your code handles schema interpretation
- **Easy evolution**: Add new fields without migrations

#### Example

```javascript
// Original document
{ "id": 1, "name": "Alice", "email": "alice@example.com" }

// New document with additional field (no migration needed)
{ "id": 2, "name": "Bob", "email": "bob@example.com", "age": 30 }
```

#### Pros

- Fast schema evolution
- No downtime for schema changes
- Better for rapidly changing requirements

#### Cons

- Data quality issues if not validated properly
- Application logic becomes more complex
- Inconsistent data structures possible

---

## 2. Rolling Updates / Staged Rollouts

When deploying new versions of your application that change the schema, you need to ensure **both old and new versions can coexist** during the update period.

### 2.1 Server-Side Rolling Updates

### What is a Rolling Update?

Instead of updating all servers at once (which causes downtime), you update them **gradually** — one or a few at a time.

#### Process

1. Deploy new version to Server 1 (others still run old version)
2. Monitor for issues
3. Deploy to Server 2, 3, etc., progressively
4. Eventually all servers run the new version

#### Benefits

- **Zero downtime**: Some servers always available
- **Easy rollback**: If issues arise, stop the rollout
- **Lower risk**: Problems affect fewer users initially

#### Requirements

- **Backward compatibility**: New code must read old data
- **Forward compatibility**: Old code must handle new data (or ignore it)

---

### 2.2 Client-Side Rolling Updates

### What is Client-Side Rolling Update?

Mobile apps, desktop software, and browser apps update at different times — users control when they upgrade.

#### Challenges

- **Long coexistence**: Old and new versions run simultaneously for weeks/months
- **API versioning**: Backend must support multiple client versions
- **Graceful degradation**: Old clients should still work (maybe with reduced features)

#### Example

- WhatsApp releases v2.0 with new features
- Some users update immediately, others weeks later
- Backend must support both v1.0 and v2.0 message formats

---

## 3. Backward and Forward Compatibility

### 3.1 Backward Compatibility

#### Definition

#### New code can read data written by old code.

### Why

It Matters

When you deploy a new version, it must be able to process data created by the previous version.

#### Example

- Old version stores: `{ "name": "Alice" }`
- New version expects: `{ "name": "Alice", "age": 25 }`
- New version must handle missing `age` field (use default or null)

### How

to Achieve

- **Optional fields**: New fields should have defaults
- **Ignore unknown fields**: Don't fail on missing data
- **Type compatibility**: Avoid breaking type changes

---

### 3.2 Forward Compatibility

#### Definition

#### Old code can read data written by new code.

### Why

It Matters

During rolling updates, old servers still running must handle data from updated servers.

#### Example

- New version stores: `{ "name": "Alice", "age": 25, "preferences": {...} }`
- Old version expects: `{ "name": "Alice" }`
- Old version must ignore unknown fields (`age`, `preferences`)

### How

to Achieve

- **Ignore unknown fields**: Old code skips fields it doesn't recognize
- **Additive changes only**: Never remove or rename required fields
- **Versioning**: Include version numbers in data

---

## 4. Why Do We Need Encoding?

### The Problem: In-Memory vs On-Disk/Network

**In**- -Memory Representation

- Data structures: objects, arrays, pointers
- Optimized for CPU access
- Platform-specific (e.g., Java objects, Python dicts)

**On**- -Disk / Network Representation

- Self-contained byte sequences
- Platform-independent
- No pointers (meaningless across processes/machines)

### What is Encoding?

**Encoding (Serialization)** is the process of converting in-memory data structures into a byte sequence for storage or transmission.

### What is Decoding?

**Decoding (Deserialization)** is the reverse process — converting byte sequences back into in-memory data structures.

### Why Encoding is Essential

1. **Persistence**: Store data to disk/database
2. **Network communication**: Send data between services
3. **Language independence**: Different services can use different languages
4. **Efficiency**: Compact representations save bandwidth and storage

---

## 5. Formats for Encoding

### 5.1 Language-Specific Formats

#### Examples

- **Java**: `java.io.Serializable`
- **Python**: `pickle`
- **Ruby**: `Marshal`

#### Problems

- **Language lock-in**: Only works with one language
- **Security risks**: Deserialization can execute arbitrary code
- **Versioning issues**: Hard to maintain backward/forward compatibility
- **Inefficient**: Often verbose and slow

### When

to Use

- Quick prototyping within a single language
- Internal tools with no cross-language requirements

**❌ Not recommended for production systems**

---

### 5.2 JSON (JavaScript Object Notation)

### What is JSON?

A text-based, human-readable format widely used for web APIs.

#### Example

```json
{
  "name": "Alice",
  "age": 30,
  "email": "alice@example.com"
}
```

#### Pros

- **Human-readable**: Easy to debug
- **Language-independent**: Supported by all major languages
- **Web-friendly**: Native JavaScript support
- **Flexible schema**: Easy to add/remove fields

#### Cons

- **No schema enforcement**: Easy to make mistakes
- **Type ambiguity**: No distinction between integer and float
- **Inefficient**: Text encoding is verbose (larger size)
- **No binary data support**: Must base64-encode binaries
- **Precision issues**: Numbers can lose precision (e.g., large integers)

### Example

Problem

```json
// JSON doesn't distinguish between int and float
{ "price": 19.99 }  // Is this a float or just how it's displayed?

// Large integers lose precision in JavaScript
{ "id": 9007199254740993 }  // May become 9007199254740992
```

---

### 5.3 XML (eXtensible Markup Language)

### What is XML?

A verbose, tag-based text format popular in enterprise systems.

#### Example

```xml
<user>
  <name>Alice</name>
  <age>30</age>
  <email>alice@example.com</email>
</user>
```

#### Pros

- **Self-documenting**: Tags describe the data
- **Schema validation**: XSD (XML Schema Definition) for structure enforcement
- **Namespaces**: Avoid naming conflicts

#### Cons

- **Extremely verbose**: More bytes than JSON for same data
- **Complex parsing**: Slower to parse than JSON
- **Type ambiguity**: Everything is a string unless specified
- **Declining popularity**: JSON has largely replaced it

---

### 5.4 CSV (Comma-Separated Values)

### What is CSV?

A simple text format for tabular data.

#### Example

```csv
name,age,email
Alice,30,alice@example.com
Bob,25,bob@example.com
```

#### Pros

- **Simple**: Easy to read and write
- **Compact**: More compact than JSON/XML for tabular data
- **Widely supported**: Excel, databases, data tools

#### Cons

- **No schema**: Column names optional, no type information
- **Escaping issues**: Commas, quotes, newlines need special handling
- **No nested structures**: Flat data only
- **Inconsistent standards**: Different delimiters, quote styles

---

### 5.5 Binary Formats

Binary formats encode data as compact byte sequences instead of human-readable text.

#### Examples

- **MessagePack**: Binary JSON
- **BSON**: Binary JSON (used by MongoDB)
- **Protocol Buffers (Protobuf)**: Google's format
- **Thrift**: Facebook's format
- **Avro**: Apache's format

#### Pros

- **Compact**: Much smaller than text formats
- **Fast parsing**: Binary parsing is faster
- **Schema support**: Enforce structure and types
- **Efficient**: Better for network/storage

#### Cons

- **Not human-readable**: Need tools to inspect
- **Schema required**: Must distribute schema definitions
- **Complexity**: More setup than JSON

---

## 6. Problems with Text Formats (JSON/XML) and Solutions

### Problem 1: No Schema Enforcement

- **Issue**: Easy to send wrong data types or missing fields
- **Solution**: Use binary formats with schemas (Protobuf, Avro, Thrift)

### Problem 2: Type Ambiguity

- **Issue**: JSON doesn't distinguish int vs float, no date/time types
- **Real example**: Twitter API initially sent user IDs as numbers, but JavaScript couldn't handle large integers (> 2^53), so Twitter switched to sending IDs as strings

```json
// Original (problematic)
{ "user_id": 123456789012345678 }  // Lost precision in JavaScript

// Fixed
{ "user_id": "123456789012345678", "user_id_str": "123456789012345678" }
```

### Problem 3: Size and Bandwidth

- **Issue**: Text formats are verbose (spaces, quotes, tags)
- **Solution**: Binary formats compress better and use less bandwidth

### Problem 4: Parsing Performance

- **Issue**: Text parsing (especially XML) is slow
- **Solution**: Binary formats parse faster with less CPU

### Problem 5: No Versioning

- **Issue**: Hard to evolve schemas safely
- **Solution**: Formats like Protobuf, Avro have built-in versioning and compatibility rules

---

## 7. When to Use Each Format

| Format | Best For | Avoid For |

|--------|----------|-----------|
| **JSON** | Web APIs, config files, human readability | High-performance systems, large datasets |
| **XML** | Legacy enterprise systems, document markup | New projects (prefer JSON) |
| **CSV** | Data export/import, spreadsheets | Nested data, complex structures |
| **Binary (Protobuf/Thrift/Avro)** | Internal microservices, high-throughput systems, large data | Public APIs (less accessible), quick prototyping |

---

## Summary

### Key Takeaways

1. **Schema-on-write** (SQL) enforces structure upfront; **schema-on-read** (NoSQL) is flexible but requires application-level validation
2. **Rolling updates** require both **backward** and **forward compatibility**
3. **Encoding** converts in-memory data to bytes for storage/network transfer
4. **Text formats** (JSON, XML) are human-readable but inefficient
5. **Binary formats** (Protobuf, Thrift, Avro) are compact, fast, and type-safe but need schemas
6. **Type ambiguity** in JSON/XML can cause bugs (e.g., Twitter's user ID issue)

### Next Steps

- Dive deeper into **Protobuf, Thrift, and Avro** (Chapter 3)
- Learn about **data flow patterns** (databases, RPC, message passing) (Chapter 4)

---

**Related Topics**: Serialization, Schema migration, API versioning, Data contracts
