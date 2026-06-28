# Availability, Reliability, Consistency & Distributed System Theorems

## Table of Contents

1. [Availability](#1-availability)
   - 1.1 [Measuring Availability](#11-measuring-availability)
   - 1.2 [Availability in Sequence vs. in Parallel](#12-availability-in-sequence-vs-in-parallel)
   - 1.3 [SLAs, SLOs, and SLIs](#13-slas-slos-and-slis)
   - 1.4 [High Availability Architecture Patterns](#14-high-availability-architecture-patterns)
2. [Reliability & Fault Tolerance](#2-reliability--fault-tolerance)
   - 2.1 [Reliability vs. Availability](#21-reliability-vs-availability)
   - 2.2 [Redundancy: Active-Passive, Active-Active](#22-redundancy-active-passive-active-active)
   - 2.3 [Failover Mechanisms: Hot, Warm, Cold Standby](#23-failover-mechanisms-hot-warm-cold-standby)
3. [Consistency](#3-consistency)
   - 3.1 [Strong Consistency](#31-strong-consistency)
   - 3.2 [Eventual Consistency](#32-eventual-consistency)
   - 3.3 [Weak Consistency](#33-weak-consistency)
   - 3.4 [Causal Consistency](#34-causal-consistency)
   - 3.5 [Read-Your-Writes Consistency](#35-read-your-writes-consistency)
4. [The CAP Theorem](#4-the-cap-theorem)
   - 4.1 [Consistency, Availability, Partition Tolerance](#41-consistency-availability-and-partition-tolerance)
   - 4.2 [CP Systems vs. AP Systems: Real-World Examples](#42-cp-systems-vs-ap-systems-real-world-examples)
5. [PACELC Theorem: Latency Extension of CAP](#5-pacelc-theorem-latency-extension-of-cap)
   - 5.1 [PA/EL Systems](#51-pael-systems)
   - 5.2 [PC/EC Systems](#52-pcec-systems)
   - 5.3 [PA/EC Systems](#53-paec-systems)
6. [Quick Reference Tables](#6-quick-reference-tables)

---

## 1. Availability

**Availability** is the percentage of time a system is operational and accessible to users.

> A system that works perfectly but is only up 23 hours a day is only **95.8% available** — which means more than **18 days of downtime per year**.

Availability is one of the foundational non-functional requirements of any production system. High availability (HA) means the system continues to serve requests despite failures in individual components.

---

### 1.1 Measuring Availability

Availability is expressed as a percentage of uptime over a time window:

```
Availability = (Total Time - Downtime) / Total Time × 100%
```

Or equivalently:

```
Availability = MTBF / (MTBF + MTTR)
```

Where:
- **MTBF** = Mean Time Between Failures (how often things break)
- **MTTR** = Mean Time To Recovery (how fast you recover when they do)

> **Key insight:** You improve availability either by making failures rarer (MTBF ↑) or by recovering faster (MTTR ↓). Automation and monitoring primarily attack MTTR.

#### The "Nines" Table

| Availability | Downtime / Year | Downtime / Month | Downtime / Day | Common Name |
|---|---|---|---|---|
| 99% | 3.65 days | 7.2 hours | ~14.4 minutes | Two nines |
| 99.9% | 8.77 hours | 43.8 minutes | ~1.44 minutes | Three nines |
| 99.95% | 4.38 hours | 21.9 minutes | ~43.2 seconds | Three and a half nines |
| 99.99% | 52.6 minutes | 4.38 minutes | ~8.6 seconds | Four nines |
| 99.999% | 5.26 minutes | 26.3 seconds | ~0.86 seconds | Five nines |

**Cost vs. nines:** Each additional nine is **exponentially more expensive** to achieve. Going from two nines to three nines is relatively straightforward (load balancing, basic monitoring). Going from four nines to five nines requires massive redundancy, extensive runbook automation, and global failover — typically costing 10–100× more per nine.

> **Practical guidance:** Most production web services target **three nines (99.9%)**. Mission-critical infrastructure (payments, healthcare) targets four to five nines. Note that 100% availability is not a meaningful target — it is both impossible and indistinguishable from 99.999% from a user perspective (other failures in the user's path, like their ISP or WiFi, are far less reliable anyway).

---

### 1.2 Availability in Sequence vs. in Parallel

When a user request passes through multiple components, the overall system availability depends on how those components are arranged.

#### Sequential (In Series)

Every component must be available for the system to work:

```
[Client] ──► [Load Balancer] ──► [App Server] ──► [Cache] ──► [Database]
```

If **any** component fails, the request fails. Availabilities **multiply**:

```
System Availability = A₁ × A₂ × A₃ × ... × Aₙ
```

**Example:**
```
LB (99.9%) × App (99.9%) × DB (99.9%) = 99.7%
```

Four components each at 99.9% → combined: **~99.6%**. Each component added reduces overall availability.

> **The brutal math of series systems:** Three nines × three nines × three nines ≠ three nines. Adding components makes the system *less* available unless redundancy is introduced at each layer.

#### Parallel (Redundant Components)

Multiple instances of the same component; the system fails only if **all** instances fail simultaneously:

```
                         ──► [App Server 1]
[Client] ──► [LB] ──►                        ──► [DB Primary]
                         ──► [App Server 2]
```

```
Parallel Availability = 1 - (1 - A)ⁿ
```

Where `n` = number of redundant instances and `A` = availability of each.

**Example:**
Two app servers at 99.9% each:
```
1 - (1 - 0.999)² = 1 - 0.000001 = 99.9999%
```

Adding a second instance jumps availability from **99.9% to 99.9999%** — four additional nines.

#### Why This Matters in Architecture

- Every critical component in your sequential request path should have **redundant parallel instances**.
- A single database without a replica is a **single point of failure** — one hardware failure drops availability to zero.
- This is why high-availability architectures always show components in pairs (at minimum).

---

### 1.3 SLAs, SLOs, and SLIs

These three concepts form the **language of reliability** used by engineering teams and their customers.

```
SLI (what you measure)
       │
       ▼
SLO (what you target internally)
       │
       ▼
SLA (what you promise externally, with consequences)
```

#### SLI — Service Level Indicator

An **SLI** is the actual metric being measured. It is a quantitative measure of some aspect of the service.

| Category | Example SLI |
|---|---|
| Availability | `(successful requests) / (total requests) × 100%` |
| Latency | `percentage of requests served in < 200ms` |
| Error rate | `HTTP 5xx responses / total responses` |
| Throughput | `requests processed per second` |
| Durability | `data written that can be read back correctly` |

> SLIs are raw numbers from your monitoring system. They are what you measure, not what you promise.

#### SLO — Service Level Objective

An **SLO** is an **internal target** for an SLI. It defines what "good" looks like.

- Latency SLI target: 99% of requests respond in < 300ms.
- Availability SLI target: 99.95% of requests succeed.

SLOs are used internally to:
- Trigger alerts when thresholds are approached.
- Drive engineering priorities.
- Define **error budgets**.

**Error Budget:**
```
Error Budget = 1 - SLO
```

If SLO = 99.9%, then the error budget = **0.1% of the measurement window** — about 43 minutes per month. Teams can "spend" this budget on risky deployments. Once the budget runs out, stability work takes precedence over new features.

> **Key principle (from Google SRE):** SLOs should always be stricter than SLAs. If you promise 99.9% to customers, target 99.95% internally. The gap is your safety net.

#### SLA — Service Level Agreement

An **SLA** is an **external contractual commitment** to a customer, with financial or legal consequences if violated.

```
"If monthly availability drops below 99.9%, the customer receives a 10% service credit."
```

SLAs:
- Are legally binding agreements.
- Are typically less stringent than SLOs (safety buffer).
- Usually specify remedies (service credits, refunds) for violations.
- Cover multiple dimensions: availability, support response time, resolution time.

#### The Hierarchy in Practice

```
Internal SLO: 99.95% availability (what team targets)
External SLA: 99.9%  availability (what we promise customers)
Actual SLI:   99.96% availability (what monitoring reports)

→ SLA is met. SLO is met. Error budget partially consumed.
```

---

### 1.4 High Availability Architecture Patterns

Achieving high availability requires applying patterns at every layer of the stack.

#### Load Balancing

Distribute traffic across multiple instances. If one instance fails, the load balancer detects it via health checks and stops routing to it.

```
                          ──► [Server A] (healthy)
[Users] ──► [Load Balancer]
                          ──► [Server B] (unhealthy → excluded)
```

#### Geographic Redundancy (Multi-Region)

Deploy in multiple geographic regions. If one data center goes down, traffic fails over to another.

```
Region A (Active) ◄──── GSLB ────► Region B (Active/Standby)
     │                                       │
  Users in                               Users in
  North America                            Europe
```

#### Health Checks & Automated Failover

Continuous health checks detect failures and trigger automated failover without human intervention. MTTR drops from minutes/hours (manual) to seconds (automated).

#### Circuit Breaker Pattern

When a downstream service is failing, a circuit breaker **stops sending requests** to it, rather than letting all threads block waiting for timeouts.

```
State: CLOSED (normal) → too many failures → State: OPEN (fast-fail)
                                                      │
                                               after timeout
                                                      ▼
                                             State: HALF-OPEN (probe)
                                             → success → CLOSED
                                             → failure → OPEN
```

#### Graceful Degradation

When a non-critical component fails, the system continues with reduced functionality rather than failing completely.

- Recommendation engine down → show popular items instead.
- Review service down → hide review section, show rest of page.
- Cache miss + DB slow → serve stale data with a staleness warning.

---

## 2. Reliability & Fault Tolerance

### 2.1 Reliability vs. Availability

These terms are often used interchangeably but describe different properties:

| Concept | Definition | Metric |
|---|---|---|
| **Availability** | Is the system *accessible* when a user tries to use it? | % uptime |
| **Reliability** | Does the system *produce correct results consistently* over time? | MTBF, error rates |

> **The key difference:**
> - A system can be **available but unreliable**: it responds to every request but returns wrong data 5% of the time.
> - A system can be **reliable but not always available**: it produces correct results but goes down for maintenance regularly.

**Examples:**
- A buggy payment service that processes requests but double-charges users: **available, unreliable**.
- A database that does planned maintenance windows: **reliable, not fully available**.
- AWS S3: designed for **eleven nines of durability** (data reliability) with **four nines of availability** (access).

**Fault Tolerance** is the system's ability to continue operating correctly even when some of its components fail. It is the mechanism that produces both availability and reliability under failure conditions.

---

### 2.2 Redundancy: Active-Passive, Active-Active

Redundancy means maintaining extra capacity that can take over when primary components fail.

#### Active-Passive (Primary-Standby)

```
                ┌──────────────────┐
[Users] ──────► │   PRIMARY NODE   │ ←── heartbeat ──►  [STANDBY NODE]
                │  (serving traffic)│                   (idle, ready to take over)
                └──────────────────┘
```

**How it works:**
- One node (primary) handles all traffic.
- One or more standby nodes receive replicated data and monitor the primary via heartbeat.
- If the primary fails, the standby is promoted and starts accepting traffic.
- DNS or load balancer is updated to point to the new primary.

**Types:**
- **Hot standby:** Standby is always running, data fully synchronized. Failover in seconds.
- **Warm standby:** Standby is running but not fully caught up. Needs minutes to sync before taking over.
- **Cold standby:** Standby is not running. Must boot and restore from backup. Takes many minutes to hours.

**Trade-offs:**

| Pro | Con |
|---|---|
| Simpler to operate (one authoritative node) | Standby capacity is wasted (no traffic in normal operation) |
| No write conflicts | Failover adds latency (DNS propagation, app reconnects) |
| Easy to reason about data state | Single active node is still a throughput ceiling |

---

#### Active-Active (Multi-Master)

```
                ┌──────────────────┐         ┌──────────────────┐
[Users] ──────► │     NODE A       │ ◄──────► │     NODE B       │ ◄── [Users]
                │  (serving traffic)│  sync   │  (serving traffic)│
                └──────────────────┘         └──────────────────┘
```

**How it works:**
- All nodes handle traffic simultaneously.
- Data is kept synchronized between nodes (synchronously or asynchronously).
- If one node fails, the load balancer routes all traffic to the remaining nodes.

**Trade-offs:**

| Pro | Con |
|---|---|
| All capacity is utilized (no idle standby) | Conflict resolution needed for concurrent writes |
| Zero-downtime failover (traffic just redistributes) | Harder to reason about data consistency |
| Higher overall throughput | More complex to implement and operate |

> **Rule of thumb:** Active-passive is simpler and preferred for databases (easier consistency guarantees). Active-active is preferred for stateless application servers (no consistency problem — any server can serve any request).

---

### 2.3 Failover Mechanisms: Hot Standby, Warm Standby, Cold Standby

These represent a spectrum of cost vs. recovery time:

```
                 Recovery Time
  Fast ◄──────────────────────────────────────────► Slow
    │                                                │
    │  HOT STANDBY   │  WARM STANDBY  │  COLD STANDBY
    │
  Cost
  High ◄──────────────────────────────────────────► Low
```

#### Hot Standby (Immediate Failover)

- Standby runs continuously with data fully synchronized in real time.
- Failover in **seconds** (just a DNS update or VIP swap).
- **Cost:** You pay for a full duplicate of your entire infrastructure, running 24/7.
- **Use case:** Critical systems that cannot afford even brief downtime (payment processing, air traffic control).

```
Primary DB (active) ──── synchronous replication ────► Hot Standby DB (running, synced)
                                                              │
                                                   Failover: seconds
```

#### Warm Standby (Minutes to Failover)

- Standby runs but may lag behind the primary (asynchronous replication).
- Needs a few minutes to catch up and become the new primary.
- **Cost:** Moderate — infrastructure running but not necessarily full capacity.
- **Use case:** Most production systems. Good balance of cost and recovery time.

```
Primary DB (active) ──── asynchronous replication ───► Warm Standby (running, slight lag)
                                                               │
                                                    Failover: 2–10 minutes
```

#### Cold Standby (Hours to Failover)

- No standby running. Recovery requires booting a new instance from a snapshot or backup.
- **Cost:** Low — you only pay for backup storage, not running hardware.
- **Use case:** Non-critical systems, DR (Disaster Recovery) for systems that can tolerate hours of downtime.

```
Primary DB (active) ──── periodic backups ────► S3 / Backup Storage
                                                        │
                                             Failover: restore backup → boot → configure
                                             Time: 30 minutes to hours
```

#### Comparison Table

| Mechanism | RTO | RPO | Cost | Use Case |
|---|---|---|---|---|
| Hot Standby | Seconds | Near zero | High | Payments, healthcare, real-time systems |
| Warm Standby | 2–10 min | Seconds to minutes | Medium | Most web apps, APIs |
| Cold Standby | 30 min – hours | Minutes to hours | Low | Dev/QA, non-critical batch systems |

> **RTO** = Recovery Time Objective (how fast you must recover)  
> **RPO** = Recovery Point Objective (how much data loss you can tolerate)

---

## 3. Consistency

In a distributed system with multiple replicas, **consistency** describes the guarantees made about what value a read returns relative to recent writes.

> The fundamental tension: writes go to one node, but reads might go to any node. How synchronized do those nodes need to be?

---

### 3.1 Strong Consistency

Every read receives the **most recent write**, or an error. No stale reads are ever returned.

```
Client writes X=5 to Node A
         │
         ▼
Node A replicates to B and C SYNCHRONOUSLY
         │
         ▼
Acknowledgment sent to client only after ALL nodes confirm
         │
         ▼
Any subsequent read (from any node) returns X=5
```

**How it's achieved:**
- Synchronous replication: the write is not acknowledged until all replicas confirm.
- Distributed consensus protocols: Paxos, Raft (used in etcd, Zookeeper, CockroachDB).
- Two-Phase Commit (2PC) for distributed transactions.

**Trade-offs:**

| Pro | Con |
|---|---|
| Simple reasoning — reads always reflect reality | Higher write latency (must wait for all replicas) |
| No stale reads — never see phantom data | Reduced availability — if any replica is unreachable, write can block or fail |
| Required for financial and inventory systems | Lower throughput due to coordination overhead |

**Use when:** Correctness is critical. Bank account balances, inventory counts, seat bookings.

---

### 3.2 Eventual Consistency

Given **no new writes**, all replicas will **eventually** converge to the same value. In the short term, different nodes may return different (stale) values.

```
Client writes X=5 to Node A
         │
         ▼
Node A acknowledges immediately
         │
         ├──── async replication to B (takes 50ms) ──► B returns X=5 ✓ (after delay)
         └──── async replication to C (takes 200ms) ─► C may return X=3 ✗ (briefly stale)
```

**How it's achieved:**
- Asynchronous replication: writes are acknowledged immediately; replicas catch up in the background.
- Conflict resolution (last-write-wins, CRDTs) handles concurrent writes.

**Trade-offs:**

| Pro | Con |
|---|---|
| Low write latency (no waiting for replicas) | Temporarily stale reads |
| High availability (write succeeds even if replicas are slow) | Application must tolerate inconsistency |
| High throughput | Conflict resolution adds complexity for concurrent writes |

**Use when:** Stale reads are acceptable short-term. DNS, social media feeds, shopping carts, recommendation systems, user profiles.

> DNS is the canonical example of eventual consistency: a DNS update may take minutes to hours to propagate globally, but eventually every resolver sees the new value.

---

### 3.3 Weak Consistency

**No guarantee** is made about when (or whether) a read will return the most recent write. The system makes a best-effort attempt.

```
Client writes X=5
         │
         ▼
No guarantee about when other nodes will reflect X=5
Some reads may never see X=5 (e.g., if connection drops before replication)
```

**Where it appears:**
- CPU caches (cache coherence is not guaranteed across cores without explicit synchronization).
- Real-time multiplayer games (position data is best-effort; latency is more important than perfect accuracy).
- Live video streaming (a few dropped frames don't matter; rebuffering is worse).
- VoIP calls (a dropped packet is not retransmitted; freshness matters more than completeness).

**Use when:** Latency dominates. Losing or skipping some data is acceptable. Real-time systems where old data is worthless (live sensor readings, game state).

---

### 3.4 Causal Consistency

Operations that are **causally related** are seen by all nodes in the correct causal order. Operations with no causal relationship may be seen in different orders by different nodes.

```
Alice posts: "I'm getting married!" (Write W₁)
Bob reads Alice's post               (Read R₁, causally depends on W₁)
Bob replies: "Congratulations!"      (Write W₂, causally depends on R₁)

Causal consistency guarantees:
→ Anyone who reads Bob's reply (W₂) must also see Alice's original post (W₁).

No guarantee:
→ Concurrent unrelated posts may appear in different orders on different nodes.
```

**Implementation:**
- **Vector clocks:** Each message carries a version vector tracking causality.
- **Lamport timestamps:** Logical clocks that establish happens-before ordering.

**Trade-offs:**

| Pro | Con |
|---|---|
| Preserves semantic correctness for related operations | More complex tracking than eventual consistency |
| Better than eventual consistency for collaborative systems | Higher metadata overhead (vector clocks) |
| Less costly than strong consistency | Only partial ordering — unrelated ops still unordered |

**Use when:** Social media timelines, collaborative documents, comment threads — systems where related content must appear in logical order but unrelated content can be looser.

---

### 3.5 Read-Your-Writes Consistency

A client always sees the **effects of their own writes** in subsequent reads, even if other clients may still see stale data.

```
User updates their profile photo (write to primary)
         │
         ▼
User immediately views their profile (read)
         │
         ▼
Read-your-writes guarantee: user sees their new photo
         (even if the replica hasn't received the write yet)

Other users may temporarily still see the old photo (eventual consistency for them)
```

**How it's achieved:**
- Route reads to the primary for a short window after a write.
- Track a per-user "last write timestamp" and route reads to replicas only after they've caught up to that timestamp.
- Use sticky sessions to route the same user to the same server/replica.

**Why it matters for UX:**
Without read-your-writes, users experience jarring behavior: they update something, refresh the page, and see the old value — appearing as if their action had no effect.

**Used by:** Most modern web applications. This is often the first consistency upgrade applied on top of eventual consistency systems.

---

## 4. The CAP Theorem

Proposed by Eric Brewer in 2000, the CAP theorem is one of the foundational principles of distributed system design.

> **In a distributed system, you can only guarantee two of the following three properties simultaneously:**

---

### 4.1 Consistency, Availability, and Partition Tolerance

```
            Consistency
                /\
               /  \
              /    \
             / CA?  \
            /        \
           /──────────\
          /            \
  Availability ────── Partition
                      Tolerance
```

#### Consistency (C)
Every read receives the **most recent write or an error**. All nodes see the same data at the same time. (This is *linearizability* — strong consistency in the CAP sense.)

#### Availability (A)
Every request receives a **response** (not an error), without guarantee it contains the most recent data. The system keeps serving, even if some data may be stale.

#### Partition Tolerance (P)
The system continues to operate despite **network partitions** — failures where messages between nodes are lost or delayed.

---

#### Why You Must Choose P in Practice

Network partitions are not optional failures — they are an inherent property of distributed networks. Networks drop packets, switches fail, data centers lose inter-rack connectivity. Since you cannot eliminate network partitions, **every distributed system must be partition-tolerant**.

This reduces CAP to a **practical binary choice**:

```
When a network partition occurs:
    Option A: Maintain CONSISTENCY (return errors/timeouts rather than stale data)
    Option B: Maintain AVAILABILITY  (return stale data rather than errors)

You cannot do both.
```

---

#### CP — Consistency + Partition Tolerance

The system prioritizes returning **correct data** over always responding. During a partition, the system refuses to respond (returns errors or timeouts) rather than risk returning stale data.

```
[Node A] ─── PARTITION ─── [Node B]

Read request arrives at Node B:
→ Node B cannot confirm data is current (can't reach A)
→ Returns error / times out (no stale data)
```

**When to choose CP:** When correctness is non-negotiable. Financial transactions, inventory systems, ticket booking — situations where serving stale data causes real-world harm.

---

#### AP — Availability + Partition Tolerance

The system prioritizes **always responding** over returning the most recent data. During a partition, nodes serve whatever data they have, even if it might be stale.

```
[Node A] ─── PARTITION ─── [Node B]

Read request arrives at Node B:
→ Node B returns its current (potentially stale) data
→ System keeps running; user gets a response
→ Eventually, when partition heals, nodes reconcile
```

**When to choose AP:** When uptime matters more than perfect correctness. Social feeds, product recommendations, user profiles, DNS — situations where briefly stale data is acceptable.

---

### 4.2 CP Systems vs. AP Systems: Real-World Examples

#### CP Systems

| System | Why CP |
|---|---|
| **HBase** | Prioritizes consistency; may return errors during partitions |
| **Zookeeper** | Coordination service; returns errors rather than stale metadata |
| **etcd** | Used for Kubernetes config; must be authoritative — uses Raft consensus |
| **CockroachDB** | Distributed SQL with strong consistency via Raft |
| **Traditional RDBMS** | Single node: no partition tolerance question; add replication → must choose |

#### AP Systems

| System | Why AP |
|---|---|
| **Cassandra** | Tunable consistency; defaults to AP for high availability |
| **DynamoDB** | Eventual consistency by default; strongly consistent reads optional |
| **CouchDB** | Designed for AP with eventual consistency and conflict resolution |
| **DNS** | Availability over correctness; stale records acceptable during propagation |
| **Memcached / Redis Cluster** | Cache availability preferred; stale cache is acceptable |

#### Real-World Decision Examples

**Ticket Booking (CP required):**
Two users try to book the last seat on a flight. If Node A books it and Node B (partitioned) doesn't know, both users get confirmed. You'd have two people with the same seat. System must be CP — refuse Node B's booking until partition heals.

**Social Media Feed (AP acceptable):**
A user updates their profile name. Due to a partition, users in Europe still see the old name for 30 seconds. This is acceptable — seeing a slightly stale profile name is vastly preferable to the social media platform returning errors.

> **The right question is not "CP or AP?" but: "What is the cost of incorrect data vs. the cost of an error response?"**

---

## 5. PACELC Theorem: Latency Extension of CAP

Proposed by Daniel Abadi in 2012, PACELC extends CAP with a critical observation:

> **CAP only describes behavior during failures (partitions). But even in normal operation (no partition), distributed systems face a fundamental trade-off between latency and consistency.**

```
PACELC:

If Partition (P) → choose between Availability (A) or Consistency (C)
Else (E, normal operation) → choose between Latency (L) or Consistency (C)
```

**Why this matters:** Partitions are rare. Normal operation is the constant condition. CAP says nothing about the latency vs. consistency trade-off that engineers deal with every day.

The two trade-offs are **independent** — a system makes one choice during partitions and a separate choice during normal operation.

---

### PACELC Matrix

```
                    During Partition
                    ┌────────────────────────────────┐
                    │    Availability  │ Consistency │
                    ├─────────────────┼─────────────┤
During Normal  Low  │    PA/EL        │   PC/EL     │
Operation  Latency  │  (most common)  │    (rare)   │
           ─────────┼─────────────────┼─────────────┤
           Strong   │    PA/EC        │   PC/EC     │
           Consist. │  (uncommon)     │  (financial)│
                    └────────────────────────────────┘
```

---

### 5.1 PA/EL Systems

**Partition → Availability | Else → Low Latency**

During a partition: keep serving (choose availability).
During normal operation: respond fast (choose low latency, accept eventual consistency).

This means **both** dimensions favor performance over correctness. These systems are highly available and fast but may serve stale data in both partition and non-partition scenarios.

**Examples:**

| System | Why PA/EL |
|---|---|
| **Cassandra** | Tuned for high availability + low latency; eventual consistency by default |
| **DynamoDB** (default) | Responds from nearest replica without waiting for quorum |
| **Riak** | Favors availability; conflict resolution via vector clocks |
| **ScyllaDB** | High availability and latency sensitive; eventual consistency model |

**Ideal use cases:**
- Social media feeds (slightly stale posts are fine)
- IoT sensor data ingestion (high write volume, stale reads acceptable)
- Shopping cart (last-write-wins is acceptable)
- DNS (propagation delay is inherent and expected)

---

### 5.2 PC/EC Systems

**Partition → Consistency | Else → Consistency**

During a partition: refuse to serve inconsistent data (choose consistency over availability).
During normal operation: still prioritize correctness over speed (choose consistency over low latency).

This means **both** dimensions favor correctness. Writes are only acknowledged after confirmation from a quorum or all replicas. These systems are **the most correct but the slowest**.

**Examples:**

| System | Why PC/EC |
|---|---|
| **Google Spanner** | Global strong consistency via TrueTime + Paxos; built for financial-grade data |
| **HBase** | Consistency over availability; CP under CAP, EC under PACELC |
| **CockroachDB** | Distributed SQL; strong consistency via Raft protocol |
| **etcd** | Consensus store; correctness is paramount (Kubernetes depends on it) |

**Ideal use cases:**
- Financial transactions (a banking system must never show an incorrect balance)
- Inventory management (overselling a product is a real business problem)
- Medical records (incorrect data can endanger lives)
- Flight seat reservations (two people with the same seat is unacceptable)

---

### 5.3 PA/EC Systems

**Partition → Availability | Else → Consistency**

During a partition: keep serving (availability).
During normal operation (no partition): prioritize consistency over latency.

This is the "best-of-both-worlds" attempt. The system tolerates brief inconsistency only during actual failures, but otherwise maintains strong consistency despite the latency cost.

```
Normal operation (no partition):
  Write → wait for quorum → acknowledge
  → Consistent but has latency

During partition:
  Write → acknowledge immediately (some replicas unreachable)
  → Available but potentially inconsistent temporarily
```

**Examples:**

| System | Why PA/EC |
|---|---|
| **MongoDB** (with `majority` write concern) | Strong consistency during normal ops; falls back to availability during partition |
| **Cosmos DB** (Bounded Staleness) | Normal: consistent within bounded window; partition: available |
| **E-commerce platforms** | Amazon-style: consistent inventory under normal load, available during outages |

**Ideal use cases:**
- E-commerce inventory (consistent when healthy, must stay up during outages)
- Reservation systems that can handle brief over-booking windows during failures
- Systems where downtime is worse than a brief window of inconsistency

**Why it's rare:**
PA/EC is the hardest to implement correctly. You need one system to switch behavior based on whether it's currently in a partition. Most teams choose either PA/EL (just be fast) or PC/EC (just be correct) rather than managing two different consistency modes.

---

### PACELC System Comparison Table

| System | CAP | PACELC | Why |
|---|---|---|---|
| Cassandra | AP | PA/EL | Always fast, always available, eventual consistency |
| DynamoDB (default) | AP | PA/EL | Fast global reads, eventual consistency |
| DynamoDB (strong reads) | CP | PC/EC | Consistent reads, higher latency |
| Google Spanner | CP | PC/EC | Global transactions, financial-grade consistency |
| HBase | CP | PC/EC | Consistent, may refuse during partitions |
| CockroachDB | CP | PC/EC | Distributed SQL, strong consistency |
| MongoDB (majority write) | AP | PA/EC | Available during partition, consistent otherwise |
| Riak | AP | PA/EL | High availability focus |
| etcd | CP | PC/EC | Consensus store, correctness paramount |
| Redis (Cluster) | AP | PA/EL | Speed-first cache; stale reads acceptable |

---

## 6. Quick Reference Tables

### Consistency Model Comparison

| Model | Guarantee | Latency | Availability | Use Case |
|---|---|---|---|---|
| **Strong** | Always read most recent write | High | Lower | Payments, inventory |
| **Causal** | Causally related ops in order | Medium | Medium | Social feeds, chat |
| **Read-Your-Writes** | You see your own writes | Low | High | User profile pages |
| **Eventual** | Converges given no new writes | Low | High | DNS, recommendations |
| **Weak** | No guarantees | Lowest | Highest | Games, live video |

---

### Failover Mechanism Comparison

| Type | Standby State | RTO | RPO | Cost | Best For |
|---|---|---|---|---|---|
| Hot Standby | Always running, fully synced | Seconds | ~0 | High | Payments, healthcare |
| Warm Standby | Running, slight replication lag | 2–10 min | Seconds | Medium | Most web services |
| Cold Standby | Off; restore from backup | 30+ min | Minutes–hours | Low | Dev, non-critical apps |

---

### CAP vs. PACELC Summary

| Framework | What It Describes | When It Applies |
|---|---|---|
| **CAP** | Trade-off between C and A under partition | Only during network failures |
| **PACELC** | Trade-off between L and C in normal operation + CAP during partition | **Always** |

> **CAP tells you what breaks during a failure. PACELC tells you the price you pay every day.**

---

### SLI → SLO → SLA Chain

| Concept | Who Sets It | Who Sees It | Binding? | Example |
|---|---|---|---|---|
| **SLI** | Engineering (measure) | Internal | No | 99.96% of requests succeed |
| **SLO** | Engineering (target) | Internal | No (but drives priorities) | Target: 99.95% success rate |
| **SLA** | Business + Legal | Customers | Yes (financial penalties) | Promise: 99.9% success rate |

---

### Availability Nines Quick Reference

| Availability | Downtime / Year | Common Usage |
|---|---|---|
| 99% | 3.65 days | Internal tools |
| 99.9% | 8.77 hours | Standard web applications |
| 99.99% | 52.6 minutes | Critical business systems |
| 99.999% | 5.26 minutes | Mission-critical infrastructure |

---

### Key Formulas

```
System Availability (in series):   A_total = A₁ × A₂ × ... × Aₙ
System Availability (in parallel):  A_total = 1 − (1 − A)ⁿ

MTBF-based Availability:   A = MTBF / (MTBF + MTTR)

Error Budget:   Budget = 1 − SLO
                e.g., SLO = 99.9% → Error Budget = 0.1% ≈ 43 min/month
```

---

*References:*
- *[System Design Interview Handbook — Scalability](https://www.systemdesigninterview.com/guides/system-design-interview-handbook/31-scalability)*
- *[donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer)*
- *[Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)*
