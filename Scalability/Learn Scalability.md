# Scalability — Complete System Design Notes

## Table of Contents

1. [What is Scalability?](#1-what-is-scalability)
2. [Vertical vs Horizontal Scaling](#2-vertical-vs-horizontal-scaling)
3. [Stateless Services and Session Management](#3-stateless-services-and-session-management)
4. [Database Scaling](#4-database-scaling)
   - 4.1 [Vertical vs Horizontal](#41-vertical-vs-horizontal)
   - 4.2 [Read Replicas — Master-Slave Replication](#42-read-replicas--master-slave-replication)
   - 4.3 [Multi-Master Replication and Conflict Resolution](#43-multi-master-replication-and-conflict-resolution)
   - 4.4 [Database Sharding](#44-database-sharding)
   - 4.5 [Federation (Functional Partitioning)](#45-federation-functional-partitioning)
   - 4.6 [Connection Pooling and Query Caching](#46-connection-pooling-and-query-caching)
5. [Auto Scaling](#5-auto-scaling)
   - 5.1 [Policies](#51-policies)
   - 5.2 [Metrics](#52-metrics)
   - 5.3 [Cooldown Periods](#53-cooldown-periods)
   - 5.4 [Predictive Scaling](#54-predictive-scaling)
6. [Bottlenecks](#6-bottlenecks)
   - 6.1 [CPU, Memory, I/O, Network](#61-cpu-memory-io-network)
   - 6.2 [Single Points of Failure (SPOF)](#62-single-points-of-failure-spof)
   - 6.3 [Hot Spots and Data Skew](#63-hot-spots-and-data-skew)
   - 6.4 [Connection Limits and Thread Pool Exhaustion](#64-connection-limits-and-thread-pool-exhaustion)
   - 6.5 [Thundering Herd Problem](#65-thundering-herd-problem)
7. [Scaling Evolution: 0 to Millions of Users](#7-scaling-evolution-0-to-millions-of-users)
8. [Quick Reference Tables](#8-quick-reference-tables)

---

## 1. What is Scalability?

Scalability is a system's ability to **handle increasing load without degrading performance**.

> A system that responds in 100ms with 1,000 users but takes 10 seconds with 100,000 users is **not scalable**.

A scalable system maintains its performance characteristics as load grows — either by adding resources or distributing work more efficiently.

**Performance vs Scalability (key distinction):**
- **Performance problem** → system is slow even for a single user.
- **Scalability problem** → system is fast for one user but slow under heavy load.

---

## 2. Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)
Making your **existing machine more powerful** — upgrading RAM, CPU, or storage. The application code does not change.

### Horizontal Scaling (Scale Out)
Adding **more machines** instead of upgrading existing ones. Requests are distributed across multiple servers behind a load balancer.

### Comparison Table

| Aspect | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| What changes | Bigger machine | More machines |
| Application changes | None | Often significant (stateless design, sharding) |
| Cost curve | Exponential (2× power ≈ 3–4× price) | Linear (2× machines ≈ 2× price) |
| Ceiling | Limited by largest available machine | No theoretical limit |
| Downtime | Usually requires restart | Can add machines with zero downtime |
| Complexity | Low | Higher (distributed system challenges) |

### When to Use Which

Most production systems use **both** at different layers:

- **Application servers** → easiest to scale horizontally. Make them stateless, add a load balancer.
- **Database** → vertically scale first (more RAM for caching), then add read replicas, then shard.

> **Practical rule:** Start with vertical scaling for your DB and horizontal scaling for your app tier. Shard only when read replicas are not enough and write throughput is the bottleneck.

---

   ## 2.1 Decision Framework (When to Use Which)


<img width="1126" height="928" alt="2 1 Decision Framework" src="https://github.com/user-attachments/assets/5c71c8d4-2c3e-48ed-a5cb-185780aebd8f" />

---

## 3. Stateless Services and Session Management

### What is a Stateless Service?

A service is **stateless** when any instance can handle any request without relying on data from a previous request. Every request carries all the information the server needs.

This is the **single most enabling property for horizontal scaling**.

### The Problem: Server-Local State

When a user logs in and the server stores their session in local memory:
- Requests must always go to that specific server (**sticky sessions**).
- Sticky sessions prevent free load distribution.
- Session is lost if that server goes down.

### The Fix: Externalize State

| State Type | Externalize To |
|---|---|
| User sessions | Redis (low latency + built-in TTL) |
| File uploads | Object storage (S3) |
| Shared in-memory cache | Distributed cache (Redis, Memcached) |
| Scheduled tasks | Centralized scheduler (not cron on each server) |

> **Rule:** If removing a server should not lose any data or disrupt any user, the service is stateless.

---

## 4. Database Scaling

### Progression Overview

```
Step 1: Optimize (indexes, query tuning, connection pooling) — FREE
Step 2: Vertical scale (more RAM, faster CPU)
Step 3: Add caching layer (Redis) — often reduces DB load by 80%+
Step 4: Add read replicas (offload reads)
Step 5: Shard (split data across multiple DBs) — nuclear option
```

---

### 4.1 Vertical vs Horizontal

Same concept as application tier scaling, but applied to the database:

- **Vertical:** Bigger machine → more RAM means more data fits in buffer pool → fewer disk reads.
- **Horizontal:** Multiple DB nodes — read replicas, sharding, federation.

Exhaust vertical scaling before committing to horizontal DB scaling (sharding adds significant engineering complexity).

---

### 4.2 Read Replicas — Master-Slave Replication

<img width="568" height="258" alt="4 2 Read Replicas — Master-Slave Replication" src="https://github.com/user-attachments/assets/d17420e3-4db2-449d-8cc4-d30bf6429bee" />

**How it works:**
- The **master** handles all writes and replicates them to one or more **slaves**.
- Slaves serve **read-only** queries.
- If master goes offline, system continues in read-only mode until a slave is promoted.

**When to use:** Read-heavy workloads (social feeds, product catalogs, etc.)

**Trade-offs:**

| Pro | Con |
|---|---|
| Offloads read traffic | Replication lag (stale reads) |
| More read capacity | Promotion logic needed if master fails |
| Redundancy | More hardware and complexity |

**Handling replication lag:**
After a write, temporarily route that user's reads to the master for a short window (read-your-writes consistency). Once lag catches up, go back to replicas.

---

### 4.3 Multi-Master Replication and Conflict Resolution

<img width="386" height="158" alt="4 3 Multi-Master Replication and Conflict Resolution" src="https://github.com/user-attachments/assets/6fa32047-1041-4be9-b951-32e08a6a3b8c" />

**How it works:**
- Both masters accept reads and writes.
- Masters coordinate with each other on writes.
- If either master goes down, the other continues serving reads and writes.

**Conflict Resolution Strategies:**

When two masters receive conflicting writes simultaneously:

1. **Last-write-wins (LWW):** The write with the latest timestamp wins. Simple but can lose data.
2. **Application-level resolution:** The app detects conflicts and resolves them with custom logic.
3. **CRDTs (Conflict-free Replicated Data Types):** Data structures that merge automatically (used in distributed systems like Riak).
4. **Operational Transformation:** Used in collaborative editing (like Google Docs).

**Trade-offs:**

| Pro | Con |
|---|---|
| High availability (no single write master) | Conflict resolution complexity |
| Writes continue during partial failure | May violate ACID (loosely consistent) |
| Geographic write distribution | Increased write latency due to sync |

---

### 4.4 Database Sharding

Sharding distributes data across multiple databases, each managing a **subset** of the data.

<img width="436" height="255" alt="4 4 Database Sharding" src="https://github.com/user-attachments/assets/ab25c811-0338-4721-83c5-47f6a799e5bd" />

**Benefits:**
- Less read/write traffic per node.
- Smaller indexes → faster queries.
- Writes scale horizontally (no single master bottleneck).
- Fault isolation (one shard down ≠ full outage).

**Trade-offs:**

| Pro | Con |
|---|---|
| Scales writes and storage | Complex application logic (cross-shard queries) |
| Fault isolation per shard | Data distribution can become uneven (hot shards) |
| Independent shard scaling | Resharding is painful |

---

#### 4.4.1 Consistent Hashing

A technique to distribute data across nodes while **minimizing reshuffling** when nodes are added or removed.

**Standard hashing problem:** If you use `hash(key) % N` servers, adding/removing a server changes the mapping for nearly all keys → massive data migration.

**Consistent hashing solution:**

```
         0°
    ┌────────────┐
    │            │
270°│   Ring     │90°
    │            │
    └────────────┘
         180°

Nodes placed at positions on ring:
  Node A @ 10°
  Node B @ 130°
  Node C @ 250°

Key k → hash(k) → angle → clockwise to next node
```

- Nodes and keys are mapped to positions on a **virtual ring**.
- A key is assigned to the **first node clockwise** from its position.
- Adding/removing a node only affects **neighboring keys** — typically only `K/N` keys are remapped (K = keys, N = nodes).

**Virtual nodes (vnodes):**
Each physical node is mapped to **multiple virtual positions** on the ring. This ensures more even distribution even when node capacities differ.

**Used by:** Cassandra, DynamoDB, Memcached clusters, Redis Cluster.

---

#### 4.4.2 Resharding

Resharding is required when:
- A single shard can no longer handle its load.
- Data distribution becomes uneven due to growth.
- You're adding or removing shards.

**Strategies:**

1. **Double sharding:** Create a new schema with 2× the shards. Migrate data gradually, dual-write during migration.
2. **Consistent hashing with virtual nodes:** Minimizes the data that needs to move when adding shards.
3. **Offline migration:** Shut down writes, migrate, restart. Simple but causes downtime.
4. **Online migration:** Dual-write to old and new shards; gradually shift reads; decommission old shards. Zero downtime but complex.

> **Key insight:** Consistent hashing was specifically designed to reduce the cost of resharding. With simple modulo hashing, adding a shard requires remapping ~all data. With consistent hashing, only `1/N` of data moves when adding the N-th node.

---

### 4.5 Federation (Functional Partitioning)

Instead of one monolithic database, split databases **by function/domain**.

<img width="331" height="210" alt="4 5 Federation" src="https://github.com/user-attachments/assets/17189f2d-b918-4a96-846c-5db128409e74" />

**Benefits:**
- Less read/write traffic per DB → less replication lag.
- Smaller databases → more fits in memory → more cache hits.
- No single central master → parallel writes → higher throughput.
- Teams can independently own their domain's database.

**Trade-offs:**

| Pro | Con |
|---|---|
| Domain isolation | Cross-domain joins are complex (application-level or service links) |
| Independent scaling per function | More hardware and infrastructure |
| Reduced blast radius of failures | App must route queries to correct DB |

**Limitations:** Not effective if your schema has huge shared tables or requires frequent cross-domain joins.

---

### 4.6 Connection Pooling and Query Caching

#### Connection Pooling

Every DB query requires a connection. Creating/destroying connections is expensive (TCP handshake, auth, etc.).

**Connection pooling** maintains a **reusable pool** of open connections:

<img width="493" height="190" alt="4 6 Connection Pooling and Query Caching" src="https://github.com/user-attachments/assets/e578960b-1fc9-4fdf-893a-335c52d84293" />

- Application requests a connection from the pool (fast).
- After use, connection is returned to the pool (not closed).
- Pool manages lifecycle; DB sees fewer concurrent connections.

**Tools:** PgBouncer (PostgreSQL), ProxySQL (MySQL), HikariCP (Java), connection pooling built into ORMs.

**Modes (PgBouncer):**
- **Session mode:** One connection per client session. Safest.
- **Transaction mode:** Connection returned to pool after each transaction. Recommended for most apps.
- **Statement mode:** Connection returned after each statement. Most efficient but breaks some SQL features.

#### Query Caching

Cache the result of expensive queries so they don't hit the database on every request.

<img width="505" height="212" alt="4 6 2 Query Caching" src="https://github.com/user-attachments/assets/58156213-2e28-4ad9-bb1b-e7c67e39da57" />

**Levels of caching:**
1. **Application-level (Redis/Memcached):** Cache full query results or domain objects.
2. **Database query cache:** Most DBs have built-in query caches (MySQL's was removed in 8.0; generally unreliable).
3. **ORM-level caching:** Many ORMs support result caching.

**Cache invalidation challenges:**
- If underlying data changes, stale cache entries can be served.
- Hard to invalidate complex multi-table query caches.
- Use TTL as a safety net; use explicit invalidation on writes.

---

## 5. Auto Scaling

Auto scaling automatically **adds or removes server instances** based on real-time metrics, eliminating the need to manually provision servers before a traffic spike.

<img width="546" height="333" alt="5  Auto Scaling" src="https://github.com/user-attachments/assets/9670ee96-4683-4788-a4f4-b8ffd6c5f545" />

---

### 5.1 Policies

| Policy Type | How It Works | Best For |
|---|---|---|
| **Target tracking** | Maintain a metric at a target value (e.g., keep CPU at 60%) | Steady, predictable traffic |
| **Step scaling** | Add/remove specific amounts at specific thresholds (e.g., if CPU > 90% add 5 instances) | Aggressive response to sudden spikes |
| **Scheduled scaling** | Scale at predetermined times (e.g., every Monday 9 AM) | Known traffic patterns |
| **Predictive scaling** | ML-based forecasting from historical patterns | Recurring traffic with consistent patterns |

**Example step scaling config:**
```
CPU < 40%   → remove 2 instances
CPU 40–70%  → no action (stable zone)
CPU 70–90%  → add 2 instances
CPU > 90%   → add 5 instances immediately
```

---

### 5.2 Metrics

Common scaling signals, mapped to workload type:

| Metric | Workload Type | Notes |
|---|---|---|
| CPU utilization | CPU-bound (image processing, computation) | Most common |
| Memory utilization | Memory-heavy workloads | Less common in auto-scaling |
| Request count per instance | HTTP services | Good for load-based scaling |
| Request latency (p99) | Latency-sensitive services | Scale before users notice slowness |
| Queue depth | Async workers, message consumers | Best for queue-based scaling |
| Custom metrics | Any application-specific signal | Via CloudWatch, Prometheus, etc. |

> **Key insight:** CPU-bound apps scale well on CPU. I/O-bound apps may have low CPU even when overloaded — use request latency or queue depth instead.

---

### 5.3 Cooldown Periods

Cooldown prevents **rapid oscillating changes** after a scaling event.

**Problem without cooldown:**
```
High CPU → add 10 instances
New instances starting up → CPU appears low → remove 8 instances
CPU spikes again → add more → infinite oscillation
```

**With cooldown (typically 3–5 minutes):**
After scaling out, no further scaling action is taken until instances are warmed up and handling traffic.

**Types:**
- **Scale-out cooldown:** Time after adding instances before allowing another scale-out.
- **Scale-in cooldown:** Time after removing instances before allowing another scale-in. Usually longer to avoid removing too aggressively.

**Instance startup time is the hidden cost of auto-scaling:**
- New instances need to: boot OS → initialize app → warm up caches → establish DB connections.
- This takes 2–5 minutes during which the instance isn't helping.

**Mitigations:**
- Use pre-built AMIs or container images with everything pre-installed.
- Implement **readiness probes** that only route traffic after the instance is fully ready.
- Use **pre-warming**: launch instances before the expected spike.

---

### 5.4 Predictive Scaling

Goes beyond reactive scaling — uses ML to learn traffic patterns and scale **proactively**.

<img width="647" height="328" alt="5 4 Predictive Scaling" src="https://github.com/user-attachments/assets/0576f12c-a919-4494-8874-c625f56f194e" />

**Supported by:**
- AWS Auto Scaling (Predictive Scaling)
- Google Cloud Autoscaler
- Kubernetes Horizontal Pod Autoscaler (with custom metrics)

**When predictive scaling is valuable:**
- Regular business hours patterns (9–5 traffic peaks)
- Weekly events (Monday morning, weekend drops)
- Scheduled batch jobs that create load at fixed times

> **Combine both:** Use predictive scaling for known patterns + reactive scaling as a safety net for unexpected spikes.

---

## 6. Bottlenecks

A **bottleneck** is any component whose capacity limits the throughput of the entire system.

> Adding servers does nothing if the bottleneck is your database. Adding read replicas does nothing if the bottleneck is a single lock in your application code.

**Rule: Measure before you guess.** Instrument your system with monitoring and let data tell you where the bottleneck is.

---

### 6.1 CPU, Memory, I/O, Network

Every server has four fundamental resources. Any one of them can become the limiting constraint.

#### CPU Bottleneck

**Symptoms:** CPU running at or near 100%, rising response times correlated with CPU usage, request queuing.

**Common causes:** Image processing, video transcoding, encryption, data compression, complex computations.

**Fixes:**
- Optimize algorithms (reduce time complexity).
- Cache computed results (avoid recomputing).
- Scale horizontally (spread computation across more cores).
- Offload to background workers.

#### Memory Bottleneck

**Symptoms:** OS starts swapping to disk (orders of magnitude slower than RAM), GC pauses becoming frequent and long, processes killed by OOM (Out of Memory) killer.

**Common causes:** Caching too much in-process, loading entire datasets into memory, memory leaks.

**Fixes:**
- Profile memory allocation to find the source.
- Reduce in-process cache sizes.
- Fix memory leaks.
- Vertically scale to more RAM.

#### I/O Bottleneck (Disk / Network)

**Symptoms:** High disk wait times (`iowait`), slow queries despite low CPU, timeouts on network calls.

**Common causes:**
- **Disk I/O:** Missing database indexes, insufficient RAM for DB buffer pool, spinning HDDs.
- **Network I/O:** Many outbound calls, large payloads, chatty microservices.

**Fixes:**
- Use SSDs.
- Add database indexes.
- Add caching layer.
- Use connection pooling.
- Compress payloads.
- Batch outbound calls.

#### Network Bottleneck (Bandwidth Saturation)

**Symptoms:** Bandwidth near 100% of link capacity, latency spikes even on simple operations.

**Common causes:** Serving large files (video, images) without CDN, excessive inter-service traffic, data replication consuming bandwidth.

**Fixes:**
- CDN for static content.
- Compress payloads (gzip, Brotli).
- Batch and aggregate inter-service calls.
- Increase network link capacity.

#### Summary Table

| Resource | Symptoms | Common Causes | Fixes |
|---|---|---|---|
| CPU | High utilization, request queuing | Heavy computation, inefficient algorithms | Optimize, cache results, scale out |
| Memory | Swapping, OOM kills, GC pauses | Large caches, memory leaks, full dataset loading | Profile, fix leaks, scale up |
| Disk I/O | High iowait, slow queries | Missing indexes, insufficient RAM for buffer pool | SSD, indexing, caching |
| Network | Bandwidth saturation, high latency | Large payload transfer, chatty services | CDN, compression, batching |

---

### 6.2 Single Points of Failure (SPOF)

A **single point of failure (SPOF)** is any component whose failure brings down the entire system.

> SPOFs do not just limit growth — they create catastrophic failure modes where a single hardware malfunction takes down everything.

**How to identify SPOFs:**
Trace the path of a user request through your system. At each step ask: *"If this component fails, what happens?"* If the answer is "everything stops" → you've found a SPOF.

**Common SPOFs and Fixes:**

| Component | SPOF Risk | Fix |
|---|---|---|
| Application server | Single instance | 2+ instances behind load balancer |
| Load balancer | Single LB | Active-passive or active-active LB pair |
| Database | Single primary | Hot standby + automatic failover |
| DNS | Single provider | Multiple DNS providers |
| Cache | Single Redis node | Redis Sentinel or Redis Cluster |
| Message broker | Single broker | Clustered Kafka/RabbitMQ |

**Active-Passive vs Active-Active:**

<img width="609" height="365" alt="6 2 Single Points of Failure (SPOF)" src="https://github.com/user-attachments/assets/0e6792fe-c832-4966-b04f-40a5b1473555" />

> **Not every SPOF needs to be eliminated immediately.** Match redundancy investment to business criticality and your availability SLA.

---

### 6.3 Hot Spots and Data Skew

A **hot spot** is a server, partition, or cache key that receives **disproportionately more traffic** than its peers.

**Data skew** = data or traffic is unevenly distributed across partitions, causing some partitions to be overwhelmed while others sit idle.

#### Where Hot Spots Appear

**Sharded databases:**
Shard by user ID → one celebrity user generates 1,000× more traffic → their shard is overloaded.

**Caches:**
A viral piece of content hammers one cache key → one cache node is overwhelmed.

**Message queues:**
One Kafka partition receives far more messages than others → consumer lag builds on that partition.

**Application servers:**
Sticky sessions route all of a popular user's requests to one server.

#### Fixes by Layer

| Layer | Fix |
|---|---|
| Cache hot keys | Replicate hot keys across multiple cache nodes; use local in-process cache for top N hot items |
| Hot shard in DB | Secondary read cache for hot records; split hot shard into smaller pieces |
| Sharding strategy | Use consistent hashing with virtual nodes for more even distribution |
| Sticky sessions | Externalize sessions (Redis); remove stickiness |

> Some systems (e.g., DynamoDB, Cassandra) detect hot spots automatically and redistribute traffic in real time.

---

### 6.4 Connection Limits and Thread Pool Exhaustion

Every server has a **finite number of connections** it can maintain simultaneously.

- Database server: e.g., 500 concurrent connections.
- Web server: e.g., 10,000 concurrent TCP connections.

When limits are hit, new requests queue up or get rejected.

#### Thread Pool Exhaustion

Many frameworks process requests using a **pool of worker threads**. If all threads are blocked waiting on slow I/O operations, new requests queue up. If the queue fills → requests are dropped.

```
Thread Pool (100 threads max):
  Thread 1 ──► waiting on DB query (3s) ← stuck
  Thread 2 ──► waiting on downstream service (5s) ← stuck
  ...
  Thread 100 ──► stuck
  
  New request arrives → queue full → 503 Service Unavailable
```

**Insidious property:** CPU and memory look fine; the server appears idle, but accepts no new work.

#### Fixes

| Fix | Description |
|---|---|
| Connection pooling | Share a smaller number of DB connections across many app requests |
| Async I/O | Use non-blocking frameworks (Node.js, Netty, asyncio) — handle thousands of connections per thread |
| Strict timeouts | Ensure stuck connections are closed rather than holding slots forever |
| Circuit breakers | Fail fast when downstream is slow; don't let all threads block on one failing service |
| Bulkheads | Isolate connection pools per downstream service — one slow dependency can't exhaust all connections |

---

### 6.5 Thundering Herd Problem

A **thundering herd** happens when a large number of processes/requests are all waiting for the same event, and when that event occurs, they **all activate simultaneously** and overwhelm the system.

#### Common Scenarios

**Cache stampede (cache expiration):**

<img width="612" height="348" alt="6 5 Thundering Herd Problem" src="https://github.com/user-attachments/assets/e5ec0a27-9500-4b8b-a8b9-3d5fe1ff900a" />

**Service recovery:**
A failed service comes back online. All queued requests flood it at once before it can stabilize.

**Lock release:**
Many threads waiting on a lock all wake up simultaneously when the lock is released. Only one can acquire it; the rest burn CPU competing.

**Auto-scaling:**
Multiple new instances are marked healthy simultaneously → sudden traffic shift.

#### Fixes

| Problem | Fix |
|---|---|
| Cache stampede | **Mutex/lock-based cache rebuild:** Only one request rebuilds the cache; others wait or serve stale data |
| Cache stampede | **Probabilistic early expiration:** Start refreshing cache before it expires (reduce thundering simultaneous expiry) |
| Service recovery | **Gradual traffic ramp-up:** Use a traffic ramp policy when bringing a service back online |
| Retry storms | **Jittered retries:** Add random jitter to retry intervals so clients don't all retry at the same millisecond |
| Health check rush | **Gradual health check promotion:** New instances absorb traffic incrementally |

---

## 7. Scaling Evolution: 0 to Millions of Users

Systems scale in stages. Solve each bottleneck as it becomes real — don't pre-build for problems you don't have yet.

### Stage 1: Single Server (0–1,000 users)

```
[Client] ──► [Single Server: App + DB + Web]
```

One machine runs everything. Optimize for development speed, not scale. Set up automated backups.

---

### Stage 2: Separate DB (1,000–10,000 users)

```
[Client] ──► [App Server] ──► [DB Server]
```

Move the database to its own machine. App and DB scale independently. Add connection pooling.

---

### Stage 3: Load Balancer + Multiple App Servers (10,000–100,000 users)

```
[Client] ──► [Load Balancer] ──► [App Server 1]
                              ──► [App Server 2]  ──► [DB Server]
                              ──► [App Server N]
                                       ▲
                                  [Redis: Sessions]
```

- Make application **stateless** (sessions → Redis).
- Auto-scale the app tier.
- Add health checks.

---

### Stage 4: Caching + CDN (100,000–500,000 users)

```
[Client] ──► [CDN: Static assets] 
[Client] ──► [LB] ──► [App Servers] ──► [Redis Cache] ──► [DB]
```

- Redis caches frequent DB queries.
- CDN serves static assets.
- DB load drops dramatically (often 80%+ reduction).

---

### Stage 5: Read Replicas (500,000–2 million users)

```
[App] ──── writes ──►  [Primary DB]
[App] ──── reads ───►  [Read Replica 1]
                  ───►  [Read Replica 2]
```

- Route read queries to replicas.
- Implement read-your-writes consistency.

---

### Stage 6: Message Queues + Async Processing (2–10 million users)

```
[App] ──► [Message Queue (Kafka/RabbitMQ)] ──► [Workers]
   │                                               │
   ▼                                        (email, notifications,
Respond fast                                 analytics, etc.)
(only sync work)
```

- Move non-critical operations off the request path.
- Workers process independently and scale separately.

---

### Stage 7: Sharding + Multi-Region (10–100+ million users)

```
Region A:                          Region B:
[App] ──► [Shard 0] (users A–M)    [App] ──► [Shard 2] (users A–M replica)
[App] ──► [Shard 1] (users N–Z)    [App] ──► [Shard 3] (users N–Z replica)
                    ▲                                    ▲
                    └──────── Cross-region replication ──┘
```

- Shard by user ID or another partition key.
- Deploy in multiple geographic regions.
- GSLB (Global Server Load Balancing) routes users to nearest region.

---

## 8. Quick Reference Tables

### Scaling Checklist

| ✅ Step | Description |
|---|---|
| Optimize first | Indexes, query tuning — free wins |
| Stateless app servers | Sessions → Redis, files → S3 |
| Load balancer | 2+ app servers from day one |
| Cache layer | Redis between app and DB |
| CDN | Static assets off your servers |
| Read replicas | Offload reads from primary |
| Async processing | Non-critical work off the request path |
| Sharding | Last resort for write scalability |

### Common Bottleneck Indicators

| Symptom | Likely Bottleneck |
|---|---|
| High CPU, queuing requests | CPU-bound computation |
| Swapping/OOM kills | Memory exhaustion |
| High iowait, slow queries | Disk I/O, missing indexes |
| Bandwidth near 100% | Network saturation |
| Server looks idle but drops requests | Thread pool / connection limit exhaustion |
| DB overwhelmed after cache miss burst | Cache stampede (thundering herd) |
| One shard much hotter than others | Hot spot / data skew |
| Single component takes entire system down | Single point of failure |

### Database Technique Selection Guide

| Technique | Use When |
|---|---|
| **Indexing** | Slow queries on filtered/sorted columns |
| **Connection pooling** | Many short-lived connections to DB |
| **Read replicas** | Read-heavy workload (>70% reads) |
| **Caching (Redis)** | Same data read frequently, rarely changes |
| **Federation** | Clear domain boundaries, few cross-domain joins |
| **Sharding** | Write throughput bottleneck, huge datasets |
| **Multi-master** | Need write availability across regions |

---

> **The most expensive scaling mistake:** Optimizing for a problem you don't have yet. Engineers who implement sharding, CQRS, and event sourcing on day one for an app with 200 users spend months building infrastructure for a million-user scale and never validate the product.
>
> **Scale when you need to, not when you can.**

---

*References:*
- *[System Design Interview Handbook — Scalability](https://www.systemdesigninterview.com/guides/system-design-interview-handbook/31-scalability)*
- *[donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer)*
