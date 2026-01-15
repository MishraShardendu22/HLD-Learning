# Chapter 4: Database

## 1. Data Model

A **data model** defines:

* how data is structured
* how entities relate
* how data is stored and queried

Choosing the wrong model causes:

* performance issues
* schema rigidity
* scaling pain

---

## 2. Network Model

Old database model where:

* records are connected via **pointers**
* entities can have **multiple parents**

Pros:

* efficient for complex relationships

Cons:

* hard to modify
* tightly coupled structure
* poor maintainability

Mostly obsolete today.

---

## 3. XML Data Model

Data stored as **tagged, nested documents**.

Characteristics:

* semi-structured
* self-describing
* hierarchical

Problems:

* verbose
* slow parsing
* poor query performance

Used mainly for data exchange, not primary storage.

---

## 4. Hierarchical Model

Tree-like structure:

* one parent → many children
* no cycles

Pros:

* simple
* fast traversal

Cons:

* rigid
* cannot represent many-to-many relationships well

Example:

* file systems
* org charts

---

## 5. Relational Model (SQL)

Data stored in **tables (rows + columns)**.

Core ideas:

* strict schema
* relations via foreign keys
* powerful querying (SQL)

Pros:

* strong consistency
* mature ecosystem
* complex queries

Cons:

* schema rigidity
* horizontal scaling is hard

---

## 6. ACID Properties (Relational)

Guarantees for transactions:

* **Atomicity** – all or nothing
* **Consistency** – valid state only
* **Isolation** – concurrent transactions don’t interfere
* **Durability** – committed data survives crashes

Used in:

* banking
* financial systems

---

## 7. BASE Properties (NoSQL)

Trade-off for scalability:

* **Basically Available**
* **Soft state**
* **Eventual consistency**

Used when:

* availability > strict consistency
* large distributed systems

---

## 8. NoSQL Overview

Non-relational databases designed for:

* horizontal scaling
* flexible schemas
* high availability

Sacrifice:

* joins
* strict consistency

---

## 9. NoSQL Data Types & Relationship Handling

### a) Key–Value

* key → opaque value
* no structure

Use:

* caching
* session storage

Examples:

* Redis
* DynamoDB (core)

---

### b) Document Database

* JSON-like documents
* schema-flexible
* nested data

Use:

* evolving schemas
* content systems

Examples:

* MongoDB
* CouchDB

---

### c) Graph Database

* nodes + edges + properties
* optimized for relationships

Use:

* social networks
* recommendation engines

Examples:

* Neo4j

---

## 10. Object–Relational Mismatch

Mismatch between:

* **OOP world** (objects, inheritance, references)
* **Relational world** (tables, rows, joins)

Problems:

* complex joins
* impedance mismatch
* ORMs hide but don’t remove complexity

---

## 11. OOP vs Relational Database

**OOP**:

* objects
* nested structures
* references

**Relational**:

* flat tables
* foreign keys
* joins

Mapping between them is inherently leaky.

---

## 12. Relationships

### One-to-Many

* one record linked to many
* common (user → orders)

### Many-to-One

* inverse of above
* many children reference one parent

### Many-to-Many

* requires join table (SQL)
* direct edges (graph DB)

---

## 13. Document Database (Deeper)

Stores data as **self-contained documents**.

Pros:

* fewer joins
* natural data representation

Cons:

* duplication
* harder consistency

Best when:

* read-heavy
* aggregate-oriented access

---

## 14. Network Model Revisited

Modern NoSQL (graph DBs) revive the **network model** idea but fix:

* flexibility
* queryability
* maintainability

Graph DBs = modern, usable network models.

---
