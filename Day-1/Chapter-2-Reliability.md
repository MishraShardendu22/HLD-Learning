# Chapter 2: Reliability

## **1) Reliability & Outage**

* **Reliability** = system’s ability to operate **without failure** over time.
* **Outage** = period when service is **unavailable** or failing.

Reliability goals are defined **before** outages occur (SLA/SLO/SLI) and measured by uptime % and error metrics.

---

## **2) Measure of Reliability — SLA, SLO, SLI**

**SLI (Service Level Indicator)**:
Raw **metric** of performance/reliability:

* latency percentiles
* error rate
* throughput
* availability %

**SLO (Service Level Objective)**:
Internal **target** for SLIs:

* “99.9% of p99 latency < 200ms”
* “error rate < 0.1%”

**SLA (Service Level Agreement)**:
External **contract** with customers tied to **penalties/credits** if SLOs aren’t met.

---

## **3) Latency, Error Rate, Throughput**

* **Latency**: response time for a request. Often measured p50/p95/p99.
* **Error Rate**: % of requests that fail (5xx/timeouts).
* **Throughput**: number of requests processed per second (TPS/RPS/QPS).

These are key SLIs.

---

## **4) Redundancy**

Keep **duplicate components** so failure of one doesn’t break system:

* multiple servers
* multiple data replicas
* multiple paths to same resource
  Goal: no single point of failure.

---

## **5) Failover Mechanism**

Automatic switch to **backup/secondary** when primary fails:

* active-passive or active-active
* database primary → replica promotion
* multi-region failover

Ensures service continuity.

---

## **6) Load Balancer**

Distributes requests across multiple servers to:

* balance load
* enable scale-out
* improve availability
  Health checks keep LB from sending traffic to failed nodes.

---

## **7) Health Check & Monitoring**

**Health check**:

* periodic probes to see if service is alive
* used by LB and orchestrators

**Monitoring**:

* streams metrics (latency, error rate, throughput, saturation)
* dashboards + alerts
* detect degradation early.

Golden signals: latency, traffic, errors, saturation.

---

## **8) Replication**

Copy data across multiple nodes to improve:

* availability
* read throughput
* fault tolerance

Types:

* **Synchronous**: updates applied to all replicas before returning success (stronger consistency, slower).
* **Asynchronous**: primary commits and replicates later (faster but can be stale).

Replication supports reliability but introduces replica lag.

---

## **9) Data Partitioning (Sharding)**

Split dataset into **independent partitions** based on a key.
Each shard handles subset of data → parallel work → **scale out**.

---

## **10) Caching**

Store frequently accessed data in fast memory (Redis, CDN) to:

* reduce latency
* reduce DB load

Cache invalidation strategies are critical.

---

## **Advanced Topic — Distributed Structure**

Systems composed of many nodes working together. They must handle:

* node failures
* network delays
* data replication and routing

Distributed systems aim for **fault tolerance and scalability** at the cost of complexity.

---

## **CAP Theorem**

In a distributed system you can only simultaneously guarantee **two** of:

* **Consistency**: same view of data everywhere
* **Availability**: always respond to requests
* **Partition Tolerance**: tolerate network partitions

Real systems accept partitions → choose between CA, CP or AP trade-offs.

---

## **Eventual Consistency**

A weak consistency model where replicas converge to same data **over time** if no new writes happen.
Temporary inconsistency is allowed for better availability/latency.
Used when immediate consistency isn’t critical (social feeds, caching).

---

## **Testing for Reliability – Chaos Engineering**

Deliberately inject failures into production or staging to probe resilience.

* simulated outages, latency, dropped packets
* observe system behavior and fix weak points

**Chaos Monkey** is a famous tool that randomly disables components to test resilience.

Purpose: validate the system doesn’t break under **realistic failure scenarios** rather than just ideal tests.

### Failover Mechanism (Focused Explanation)

**Failover** = automatic transition from a **failed component** to a **healthy backup** to keep the system available.

Core idea:

> Failure is expected. Recovery must be automatic.

---

## 1. Active–Passive Failover

**Structure**:

* One **active (primary)** node serves traffic
* One or more **passive (standby)** nodes stay idle or in sync

**Flow**:

1. Primary handles requests
2. Health checks detect failure
3. Passive is **promoted to active**:
4. Traffic is redirected

**Pros**:

* Simple
* Strong consistency
* Easier data management

**Cons**:

* Wasted idle resources
* Short downtime during promotion

**Use**:

* Databases
* State-heavy systems

---

## 2. Active–Active Failover

**Structure**:

* Multiple nodes are **active simultaneously**:
* All serve traffic

**Flow**:

* Load balancer distributes requests
* If one node fails, others continue

**Pros**:

* No downtime
* Better resource utilization
* High availability

**Cons**:

* Harder consistency
* Conflict resolution needed
* More complex

**Use**:

* Stateless services
* Read-heavy workloads

---

## 3. Database Failover (Primary → Replica Promotion)

**Structure**:

* One **primary** (writes)
* One or more **replicas** (reads, backups)

**Failure**:

* Primary crashes

**Failover Steps**:

1. Detect failure (heartbeat timeout)
2. Elect a replica
3. Promote it to primary
4. Redirect writes

**Risks**:

* Data loss (async replication)
* Split-brain if promotion is wrong

**Mitigation**:

* Quorum-based election
* Write-ahead logs
* Fencing / leader locks

---

## 4. Multi-Region Failover

**Structure**:

* Same system deployed in **multiple geographic regions**:

**Types**:

* **Active–Passive**: one region live, others standby
* **Active–Active**: multiple regions serve traffic

**Failover Trigger**:

* Region outage
* Network isolation

**Traffic Shift**:

* DNS-based
* Global load balancer

**Trade-offs**:

* Higher latency
* Higher cost
* CAP trade-offs unavoidable

---

## Key Failure Detection Mechanisms

* Heartbeats
* Health checks
* Timeouts
* Consensus protocols (Raft, Paxos)

---

## Common Pitfalls

* Slow failure detection → long downtime
* Split-brain
* Stale replicas
* Manual failover (unacceptable)

---

### One-line summary

> Failover is not about preventing failure — it is about **surviving failure automatically**.
