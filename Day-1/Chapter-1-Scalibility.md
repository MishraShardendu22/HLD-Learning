# Chapter 1: Scalability

## 1. Scalability

Ability of a system to handle **growth in users, traffic, or data** without performance collapse.

Key goal:

* Maintain **performance, availability, reliability** as load increases.

---

## 2. Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

Increase resources on **one machine**.

* CPU, RAM, disk

**Pros**

* Simple
* Low latency
* No distributed complexity

**Cons**

* Hardware limit
* Expensive
* Single point of failure

Use when:

* Small/medium systems
* Predictable load

---

### Horizontal Scaling (Scale Out)

Add **more machines**, distribute load.

**Pros**

* Fault tolerant
* Near infinite scale
* Industry standard

**Cons**

* Complex
* Needs load balancing
* Network latency

Use when:

* High traffic
* Global users
* Large systems

---

## 3. Cloud

On-demand infrastructure over the internet.

* AWS, GCP, Azure

What it gives:

* Compute
* Storage
* Networking
* Managed services

Core benefit:

* **No physical hardware limits**
* Pay-as-you-go

---

## 4. CAP Theorem

In a distributed system, you can guarantee **only 2 of 3**:

* **C – Consistency**: All nodes see same data
* **A – Availability**: Every request gets a response
* **P – Partition Tolerance**: System works despite network failure

Reality:

* Network failure is unavoidable → **P is mandatory**
* Choice becomes: **CP or AP**

Examples:

* Banking → CP
* Social media feeds → AP

---

## 5. Load Balancing

Distributes incoming traffic across servers.

Why needed:

* Prevent overload
* Improve availability
* Enable horizontal scaling

Types:

* Round Robin
* Least connections
* Hash-based

Tools:

* Nginx
* AWS ELB / ALB

---

## 6. Monolithic Architecture

Entire application as **one unit**.

**Pros**

* Easy to build initially
* Simple deployment

**Cons**

* Hard to scale
* Single failure kills whole app
* Slow development as codebase grows

---

## 7. Microservices Architecture

Application split into **independent services**.

Each service:

* Own logic
* Own database (ideally)
* Scales independently

**Pros**

* High scalability
* Fault isolation
* Faster teams

**Cons**

* Network overhead
* Data consistency issues
* Complex ops

---

## 8. CDN (Content Delivery Network)

Caches **static content** close to users.

What it serves:

* Images
* JS/CSS
* Videos

Benefits:

* Low latency
* Reduced server load
* Faster global access

Examples:

* Cloudflare
* Akamai

---

## 9. Scalability in Cloud

Cloud enables **automatic scaling** without manual provisioning.

How:

* Add/remove servers dynamically
* Auto-scale based on metrics

Key idea:

* Scale only when needed
* Shrink when idle

---

## 10. Elasticity

Ability to **scale up AND down automatically**.

Difference:

* Scalability → can grow
* Elasticity → grows **and shrinks**

Example:

* Traffic spike → add servers
* Traffic drop → remove servers

---

## 11. Serverless Computing

No server management by developer.

You write:

* Functions

Cloud handles:

* Scaling
* Infra
* Availability

Examples:

* AWS Lambda
* Cloud Functions

**Pros**

* Zero ops
* Infinite scaling

**Cons**

* Cold start latency
* Limited execution time
* Vendor lock-in

---

## 12. Stateless Design

Server does **not store session data**.

Why important:

* Any request can go to any server
* Perfect for horizontal scaling

State stored in:

* DB
* Cache (Redis)
* JWT

---

## 13. Database Optimization

Databases are usually the bottleneck.

Techniques:

* **Indexing**
* **Caching**
* **Read replicas**
* **Sharding**
* **Connection pooling**

Goal:

* Reduce latency
* Increase throughput

---

## 14. Monitoring and Alerting

Continuous observation of system health.

Metrics:

* CPU
* Memory
* Latency
* Error rate

Tools:

* Prometheus
* Grafana
* CloudWatch

Purpose:

* Detect failures early
* Scale proactively
* Debug faster
