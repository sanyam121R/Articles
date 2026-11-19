---
title: "Master Database Sharding from First Principles"
date: "2024-08-20"
description: "Comprehensive guide to database sharding — from fundamentals to production challenges, consistent hashing, and real-world case studies from Instagram, Uber, Amazon, and Netflix."
tags: ["sharding", "database", "system-design", "distributed-systems", "architecture", "scaling"]
readTime: "45 min read"
link: ""
---

# Master Sharding from First Principles
### Staff Engineer Level — System Design Interview Reference

> Taught using the Feynman Technique: analogy first, definition second, trade-off always.

---

## Table of Contents

- [Part 1 — Prerequisites: Databases & Scaling](#part-1--prerequisites-databases--scaling)
- [Part 2 — Replication Before Sharding](#part-2--replication-before-sharding)
- [Part 3 — Understanding Sharding](#part-3--understanding-sharding)
- [Part 4 — Sharding Strategies](#part-4--sharding-strategies)
- [Part 5 — Shard Key Selection](#part-5--shard-key-selection)
- [Part 6 — Production Challenges](#part-6--production-challenges)
- [Part 7 — Advanced Concepts](#part-7--advanced-concepts)
- [Part 8 — System Design Interviews](#part-8--system-design-interviews)
- [Part 9 — Real Production Case Studies](#part-9--real-production-case-studies)
- [Part 10 — Mastery Section](#part-10--mastery-section)

---

# Part 1 — Prerequisites: Databases & Scaling

## What Is a Database?

**Analogy first:** A database is a super-organized filing cabinet. Instead of randomly throwing papers around, you have labeled folders, an index at the front telling you exactly where everything lives, and strict rules about how files are added, changed, or removed.

**Technical definition:** A database is a system that persistently stores structured data, provides efficient retrieval mechanisms, and enforces consistency guarantees (ACID) across reads and writes.

---

## How Databases Store Data Internally

Most relational databases store data in **pages** — fixed-size chunks of disk, typically 4KB, 8KB, or 16KB. Think of a page like a physical page in a book. The database always reads and writes in full pages, never individual bytes.

When you have a table with millions of rows, those rows are spread across thousands of pages on disk. Finding a specific row means:

1. Knowing which page it's on
2. Loading that page from disk into memory (the **buffer pool**)
3. Scanning that page to find your row

**Why this matters:** Disk access is roughly 100,000× slower than RAM access. Everything in database design is ultimately about minimizing disk I/O.

---

## Tables, Rows, Columns, Primary Keys

```
users table (stored across pages on disk):
+----+--------+------------------+---------------------+
| id | name   | email            | created_at          |
+----+--------+------------------+---------------------+
| 1  | Alice  | alice@ex.com     | 2024-01-01 09:00:00 |
| 2  | Bob    | bob@ex.com       | 2024-01-02 11:30:00 |
| 3  | Carol  | carol@ex.com     | 2024-01-03 14:45:00 |
+----+--------+------------------+---------------------+
```

A **Primary Key** uniquely identifies each row. It must be unique, non-null, and ideally immutable. Databases automatically build an index on the primary key, making lookup by PK extremely fast.

---

## B-Trees — Why Every Database Uses Them

**Why not a hash map?**

A hash map gives O(1) lookup but:
- Doesn't support range queries (`WHERE id BETWEEN 100 AND 200`)
- Doesn't maintain sorted order
- Works poorly on disk (random I/O, not sequential)

A **B-Tree** is a self-balancing tree designed specifically for disk-based storage:

```
B-Tree Structure (simplified):

                    [50 | 100]
                   /     |      \
           [20|30]    [70|80]   [110|120]
          /  |  \    /  |  \    /   |   \
        [leaf][leaf][leaf][leaf][leaf][leaf]

Each leaf node holds actual row data (or pointers to it).
Each internal node holds keys + child pointers.
All leaf nodes are at the same depth → balanced.
```

**Key properties:**
- Supports range queries efficiently (traverse in sorted order)
- Each node stores many keys → tree stays shallow (depth 3–4 even for millions of rows)
- Depth of 3–4 means: **only 3–4 disk reads to find ANY row**, regardless of table size
- Sorted order maintained automatically on inserts/deletes

---

## Clustered vs. Non-Clustered Indexes

**Clustered Index:** The actual row data IS the leaf node of the B-Tree. Data is physically sorted by the clustered key. Each table has exactly one clustered index (because data can only be sorted one way on disk).

- In MySQL InnoDB: the Primary Key is always the clustered index
- In PostgreSQL: no clustered index by default (heap storage), but you can `CLUSTER` a table

**Non-Clustered (Secondary) Index:** A separate B-Tree sorted by the indexed column, where leaf nodes contain the indexed value + a pointer back to the PK (not the actual row).

```
Finding row by email using a secondary index:

Step 1: Traverse secondary index B-Tree (email-sorted)
        alice@ex.com → PK=1

Step 2: Use PK=1 to traverse primary (clustered) B-Tree
        PK=1 → {Alice, alice@ex.com, 2024-01-01}

This "double lookup" (or bookmark lookup) is why secondary indexes
can be slower than primary key lookups.
```

**Why this matters for sharding:** Indexes only work within a single database node. There is no global index across shards — this becomes a critical constraint later.

---

## Query Execution Basics

When you run `SELECT * FROM users WHERE email = 'alice@ex.com'`:

1. **Parser:** Tokenizes and parses SQL string into an Abstract Syntax Tree
2. **Planner/Optimizer:** Evaluates multiple execution plans using table statistics. Should it use the email index? Do a full table scan?
3. **Executor:** Actually runs the chosen plan, fetching pages from disk/buffer pool
4. **Return results:** Streams rows back to client

The query optimizer makes cost-based decisions using statistics (row counts, column cardinality, index selectivity). This is why `EXPLAIN ANALYZE` is invaluable — it shows you which plan was chosen and why.

---

## Scaling Fundamentals

### Vertical Scaling (Scale Up)

**Analogy:** You have one delivery truck. You buy a bigger truck.

Add more RAM, faster CPUs, faster NVMe SSDs, more CPU cores. No code changes required.

**Hard limits:**
- The largest AWS instance (u-24tb1.metal) has 24TB RAM and costs ~$200/hr. You can't go beyond this.
- Cost grows non-linearly: a 512GB RAM server costs far more than 2× a 256GB server.
- Single point of failure: one server dies → full outage.
- Upgrades require downtime.

### Horizontal Scaling (Scale Out)

**Analogy:** Instead of one bigger truck, you buy more trucks.

Add more commodity servers, distribute data and load across them.

**Benefits:** Theoretically unlimited scale, commodity hardware, incremental capacity addition, no single point of failure.

**Costs:** Dramatically increased complexity, network becomes a bottleneck, distributed failure modes, data consistency becomes hard.

---

## The Four Bottlenecks

Understanding these is non-negotiable. They tell you **what problem sharding actually solves**.

### 1. CPU Bottleneck

**Symptoms:** CPU at 90–100%, simple queries slow during peak, query queue building.

**Causes:** Complex joins, heavy aggregations, many concurrent connections, sorting large result sets.

**Solutions before sharding:** Read replicas (offload read-heavy queries), query optimization, caching, denormalization to avoid expensive joins.

### 2. Memory Bottleneck

**Symptoms:** High disk I/O rates, buffer pool hit ratio dropping below 95%, query latency spikes.

The **buffer pool** (in-memory cache of disk pages) is your primary defense against disk I/O. When your working set (frequently accessed data) exceeds the buffer pool, you constantly go to disk.

```
Example:
  Total data:   1 TB
  Buffer pool:  64 GB
  Fits in RAM:  6.4% of data

If your access is NOT concentrated on that 6.4%,
you will constantly read from disk.
```

**Solutions:** Add RAM, add read replicas (each has its own buffer pool), add caching layer (Redis).

### 3. Disk Bottleneck

**Symptoms:** High I/O wait in `top`, disk queue depth growing, write latency spiking.

Modern NVMe SSD: ~3GB/s sequential, ~500K IOPS random. Sounds fast, but:
- One user `INSERT` can cause 5–10 disk writes (WAL write, B-Tree page updates, potentially multiple secondary index updates)
- At 10,000 writes/sec → 50,000–100,000 disk write operations/sec → saturates fast

**Solutions:** Faster disks, write batching, partitioning write load, **sharding**.

### 4. Network Bottleneck

**Symptoms:** High NIC utilization, replication lag growing, packet drops.

At 10 Gbps, you can transfer ~1 GB/s. A high-write primary streaming WAL to multiple replicas can easily saturate this. Also: connection overhead (PostgreSQL spawns a process per connection — expensive).

**Solutions:** Connection poolers (PgBouncer, ProxySQL), compression of replication stream, **sharding** (each shard has its own replication stream).

---

## Part 1 Key Takeaways

- Databases store data in B-Trees on disk pages; 3–4 disk reads to find any row
- Clustered index = data IS the index; secondary indexes require a double lookup
- Vertical scaling hits real physical and cost limits
- The four bottlenecks (CPU, Memory, Disk, Network) hit at different scales — identify which one you're hitting before choosing a solution
- Understanding these bottlenecks shows you *exactly* what sharding solves and what it doesn't

---

# Part 2 — Replication Before Sharding

## What Is Replication?

**Analogy:** You have one master chef who cooks everything and writes all recipes. You train assistant chefs who can serve customers from the same cookbook. Customers can ask any assistant chef for a dish, but only the master chef can add or change recipes.

**Technical definition:** Replication is the process of keeping multiple copies of a database synchronized, so that changes on the primary are continuously applied to one or more replicas.

---

## Primary-Replica Architecture

```
                    ┌─────────────────────┐
    Writes ────────►│     PRIMARY DB      │
                    │  (Leader / Master)  │
                    └────────┬────────────┘
                             │ Replication Stream (WAL / binlog)
                    ┌────────┴────────┐
                    │                 │
              ┌─────▼──────┐   ┌─────▼──────┐
    Reads ───►│  REPLICA 1 │   │  REPLICA 2 │◄── Reads / Analytics
              └────────────┘   └────────────┘
```

**How it works (PostgreSQL streaming replication):**

1. Write transaction commits to primary
2. Primary writes change to the **WAL** (Write-Ahead Log) — an append-only log of every change
3. WAL bytes are streamed to replicas over TCP
4. Each replica's recovery process applies the WAL to its own data files
5. Replica now has the same data (with a small delay)

**Why WAL first?** WAL is always written before data pages are modified. This means if the server crashes, you can replay the WAL to recover to a consistent state. Replication is WAL shipping.

---

## Replication Lag — The Critical Subtlety

Replication is **asynchronous by default**. The primary does NOT wait for replicas to acknowledge a write before returning success to the client.

```
Timeline:
  T=0ms:   User writes "post_id=99" to primary
  T=0ms:   Primary WAL written, "commit OK" returned to user
  T=5ms:   WAL streamed to replica
  T=10ms:  Replica applies WAL change
  T=10ms:  Replica now has post_id=99

Between T=0ms and T=10ms:
  User loads their profile → reads from replica → post_id=99 NOT FOUND
  → Appears to user as if their post disappeared
```

**Healthy systems:** Replication lag of 5–100ms. **Under write load:** seconds to minutes.

**Consistency issues this creates:**

- **Read-Your-Own-Writes:** Write to primary, immediately read from replica → don't see your own write
- **Monotonic Reads:** Read from Replica-1 (at time T=100), then Replica-2 (at T=90) → data appears to go backward in time
- **Causal Consistency:** "You can see comments on a post that you haven't seen yet"

**Solutions:**

- Route "write then immediately read" to the primary (session consistency)
- Sticky sessions — route a user to the same replica for their session
- Synchronous replication — primary waits for replica to confirm before returning (latency penalty)
- Accept eventual consistency for non-critical reads

---

## Why Replication Alone Eventually Fails

Here is the key insight that motivates sharding:

**Replication copies 100% of the data to 100% of the replicas.**

```
PRIMARY:  [All 50M users] [All 2B orders] [All 500M products]
            ↑ ALL writes go here
REPLICA1: [All 50M users] [All 2B orders] [All 500M products]
REPLICA2: [All 50M users] [All 2B orders] [All 500M products]
```

**Problem 1 — Write throughput ceiling:**
Every single write goes to ONE machine. You can add 100 read replicas and your write capacity doesn't improve by one query per second.

**Problem 2 — Storage ceiling:**
If data is 10TB, every replica needs 10TB. The primary still needs 10TB. Storage costs grow linearly with replicas.

**Problem 3 — Primary CPU ceiling:**
The primary handles writes, WAL generation, replication streaming, AND any read queries that require strong consistency. All of this competes on one set of CPUs.

### Real Company Examples

**Instagram (2011–2012, ~30M users):**
Write throughput from photo uploads, follows, likes, and new user signups was overwhelming the single PostgreSQL primary. Adding more read replicas helped read load, but the primary was the bottleneck for writes and couldn't be replicated away.

**Uber (2015–2016):**
Active trip GPS pings: ~100K concurrent trips × 1 update/4 seconds = 25,000 writes/sec. A single MySQL primary hit disk I/O limits regardless of how many read replicas were attached.

**Netflix (2012–2014):**
Encoding jobs, user activity events, content metadata updates, and watch history writes all competed on the same primary. The write workload exceeded what any single node could handle.

**Amazon:**
With 350M+ product listings and billions of daily events, neither the data size nor the write throughput could be handled by a primary-replica setup.

**The Lesson:** Replication solves read scaling and high availability. It does NOT solve write throughput limits, total dataset size limits, or single-node CPU limits for writes. For those, you need sharding.

---

## Part 2 Key Takeaways

- Replication creates copies of the entire database on separate servers for read scaling and HA
- Replication lag is real — design read-after-write carefully; don't assume reads see latest writes
- All writes still funnel to a single primary — replication provides zero write scalability
- When write throughput or total data size exceeds what any single machine can handle, you must shard

---

# Part 3 — Understanding Sharding

## The Core Analogy

**Library analogy:**
A massive library with 10 million books has one librarian. As the library grows to 100 million books, one librarian can't manage everything. Solution: divide into sections.

- **Librarian A** handles: books by authors A–F
- **Librarian B** handles: books by authors G–M
- **Librarian C** handles: books by authors N–Z

To find a book by Orwell, you go directly to Librarian C. You never need to ask A or B. Each librarian manages only ~33M books instead of 100M. Capacity scales linearly.

**Bank branch analogy:**
A single branch handles all accounts. The city grows → open multiple branches. Account 0001–9999 at Branch 1, 10000–19999 at Branch 2. The routing number tells you which branch.

---

## Technical Definition

**Sharding** (also called horizontal partitioning) is the practice of splitting a dataset across multiple independent database instances (shards), where each shard holds a distinct subset of the total data, and together all shards form the complete dataset.

```
WITHOUT SHARDING:
┌─────────────────────────────────────────┐
│            Single Database              │
│  Users: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 │  ← All writes here
│  Orders: all orders                     │  ← All data here
└─────────────────────────────────────────┘

WITH SHARDING (3 shards, hash-based):
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│    Shard 1     │  │    Shard 2     │  │    Shard 3     │
│ Users: 1, 4, 7 │  │ Users: 2, 5, 8 │  │ Users: 3, 6, 9 │
└────────────────┘  └────────────────┘  └────────────────┘
   Writes ↑              Writes ↑            Writes ↑
   (independent)        (independent)       (independent)
```

---

## What Sharding Solves

| Problem | Sharding's Solution |
|---|---|
| Write throughput ceiling | N shards = N independent write streams; throughput scales linearly |
| Storage ceiling | Data is split across shards; total capacity = N × single-shard capacity |
| Single-node CPU limit | Each shard handles only 1/N of queries and data |
| Buffer pool starvation | Each shard has its own buffer pool; working set per shard is 1/N of total |

---

## What Sharding Introduces (Read This Carefully)

Most engineers learn the benefits. The problems are where senior interviews separate candidates.

**No cross-shard JOIN:** If User table is on Shard-1 and Order table is on Shard-3, you cannot do a native SQL join across them. You must fetch from each shard separately and join in application code (slow, expensive).

**Cross-shard queries require scatter-gather:** "Get all users who signed up last week" has to query ALL shards in parallel, then aggregate results in the application layer.

**Transactions span shards:** Classic ACID transactions assume one database. Spanning a transaction across shards requires distributed transaction protocols (2PC, Saga) — complex and costly.

**Resharding is painful:** Adding more shards means moving massive amounts of data between shards while staying online. This is an operational nightmare.

**No foreign keys across shards:** You cannot have a `FOREIGN KEY` constraint pointing to a row on a different shard. Referential integrity must be enforced in application code.

**Application complexity:** Every query must be routed to the correct shard. Either your application code does this, or you add a proxy layer (which is another system to operate).

**Hot spots:** If data isn't distributed evenly, some shards get disproportionate traffic while others sit idle.

---

## Core Terminology

**Shard Key (Partition Key):**
The column (or columns) used to determine which shard a piece of data lives on. This is the single most important architectural decision when sharding. Changing it later is extraordinarily expensive.

**Logical Shard:**
A virtual partition. You might define 1,000 logical shards but run only 4 physical servers. Multiple logical shards map to one physical server. This makes adding physical servers easier — you move logical shards, not raw data ranges.

**Physical Shard:**
An actual database server instance. Owns one or more logical shards.

```
1000 logical shards → 4 physical servers:

Logical [0–249]   ──► Physical DB-1  (server: db1.internal)
Logical [250–499] ──► Physical DB-2  (server: db2.internal)
Logical [500–749] ──► Physical DB-3  (server: db3.internal)
Logical [750–999] ──► Physical DB-4  (server: db4.internal)

Add a 5th server? Move logical shards [200–249] and [450–499]
to DB-5 — only those records move, not a full resharding.
```

**Routing Layer:**
The component that receives a query, determines which shard(s) to send it to, and aggregates responses. Can be:
- **Application-level:** Sharding logic in your service code
- **Proxy middleware:** Vitess, ProxySQL, ShardingSphere sit between application and DB
- **Client library:** Cassandra and DynamoDB drivers do routing transparently

```
User Request
     │
     ▼
┌─────────────┐
│  Shard      │  "WHERE user_id=1001 → which shard?"
│  Router     │  Computes: hash(1001) % 4 = 3 → Shard 3
└──────┬──────┘
       │
       ▼
  [Physical Shard 3]  ←── query executes here
```

---

## Part 3 Key Takeaways

- Sharding horizontally partitions data across multiple independent database instances
- Each shard holds a subset of data; all shards together form the complete dataset
- It solves: write throughput, storage capacity, and single-node CPU limits
- It introduces: no cross-shard joins, scatter-gather for broad queries, difficult resharding, distributed transaction complexity, application routing complexity
- The shard key is the most critical architectural decision — choose carefully and early

---

# Part 4 — Sharding Strategies

## Strategy 1: Range-Based Sharding

### How It Works

Partition data based on ranges of the shard key value.

```
By user_id:
  user_id 1 – 1,000,000       → Shard 1
  user_id 1,000,001 – 2,000,000 → Shard 2
  user_id 2,000,001 – 3,000,000 → Shard 3

By date (for order data):
  orders Jan 2024             → Shard 1
  orders Feb 2024             → Shard 2
  orders Mar 2024             → Shard 3
```

**Routing:** Look at the shard key value, find which range it falls in, route to that shard.

### Advantages

- **Range queries are efficient:** "Get users with ID 500K–750K" queries only Shard 1
- **Sequential scans stay local:** Paginating through sorted data is fast
- **Intuitive and simple:** Easy to reason about, easy to implement
- **Natural for time-series:** Each time period is self-contained

### Disadvantages

**The Hotspot Problem (critical):**
New data always goes to the "latest" shard. For monotonically increasing keys:

```
Time = Now, new user signups:
  Shard 1: user_id 1–1M       → 100% COLD (old users, mostly reads)
  Shard 2: user_id 1M–2M      → WARM (some writes)
  Shard 3: user_id 2M–3M      → 🔥 HOT (ALL new signups write here)

Shard 3 CPU: 85%
Shard 2 CPU: 25%
Shard 1 CPU: 10%
```

This is a fundamental issue with any monotonically increasing shard key.

### Real-World Use

**HBase/Bigtable** uses range-based sharding of row keys. Their docs explicitly warn: monotonically increasing row keys (like timestamps) create "hot tablets." HBase engineers spend significant effort on row key design to avoid this. Apache Cassandra provides range-based token ring as an option but defaults to hash-based.

---

## Strategy 2: Hash-Based Sharding

### How It Works

Apply a hash function to the shard key, then use modulo (or bit-masking) to determine the shard.

```
shard_number = hash(shard_key) % num_shards

Examples with 4 shards:
  user_id=1001: hash(1001) % 4 = 2  → Shard 2
  user_id=1002: hash(1002) % 4 = 1  → Shard 1
  user_id=1003: hash(1003) % 4 = 3  → Shard 3
  user_id=1004: hash(1004) % 4 = 0  → Shard 0
  user_id=1005: hash(1005) % 4 = 2  → Shard 2
```

Good hash functions (MD5, MurmurHash3, xxHash) produce uniform distribution. Even if user IDs are sequential (1, 2, 3, 4...), the hash values spread evenly across shards.

### Advantages

- **Uniform distribution:** Writes are spread evenly — no hotspot from new data
- **Deterministic routing:** Same key always → same shard. No lookup needed.
- **Simple implementation:** One hash function + one modulo operation

### Disadvantages

**Resharding is catastrophically expensive:**

```
4 shards → 5 shards:
  user_id=1001: hash(1001) % 4 = 2  (was on Shard 2)
  user_id=1001: hash(1001) % 5 = 1  (now belongs on Shard 1)
  
  → These don't match! user_id=1001's data must move from Shard 2 to Shard 1.

Roughly (N-1)/N = 4/5 = 80% of ALL data must move
when adding just 1 shard to a 4-shard cluster.
```

This is the fundamental problem with naive hash-based sharding. Consistent hashing (Part 7) solves this.

**Range queries are expensive:** "Get users 1000–2000" must query all shards (those IDs hash to all different shards).

**No locality:** Logically related data (e.g., all orders for a user) might end up on different shards if the shard key isn't chosen carefully.

### Real-World Use

Instagram's early sharding used hash-based on user_id. DynamoDB uses hash-based partitioning (they call it the partition key). Cassandra uses consistent hashing (an improved variant).

---

## Strategy 3: Directory-Based Sharding

### How It Works

Maintain a separate **lookup table** (the "directory") that explicitly maps each shard key value to its shard.

```
Directory Service (separate highly-available database):

┌───────────────┬─────────┐
│  tenant_id    │  shard  │
├───────────────┼─────────┤
│  "acme-corp"  │ shard-3 │
│  "globex"     │ shard-1 │
│  "initech"    │ shard-3 │
│  "umbrella"   │ shard-7 │  ← moved to its own shard (whale tenant)
│  "initrode"   │ shard-2 │
└───────────────┴─────────┘

Routing: lookup(tenant_id) → shard → query shard
```

### Advantages

- **Maximum flexibility:** Move any entity to any shard by updating one row in the directory
- **Easy rebalancing:** No data recalculation — just update the mapping
- **Handles irregular distributions:** Manually assign "whale" tenants to their own shards
- **Evolves over time:** Can change sharding strategy for individual keys without a global migration

### Disadvantages

- **Directory is a critical single point of failure:** Must be highly available (run it as a replicated cluster)
- **Extra network hop:** Every query first hits the directory, then hits the shard
- **Directory latency becomes system latency:** Cache directory entries in application memory or Redis
- **Directory must be consistent:** If directory says "shard-3" but data is mid-migration to "shard-5," queries fail
- **Scalability limit:** Directory itself grows with data and can become a bottleneck at extreme scale

### Real-World Use

Flickr used a similar approach for user data routing. LinkedIn used a metadata service for partition mapping. Many multi-tenant SaaS products use directory-based sharding for tenant-to-shard routing.

---

## Strategy 4: Geo-Based Sharding

### How It Works

Assign shards based on the geographic region of the data or user.

```
user.region = 'US'    → US-East Shard   (AWS us-east-1)
user.region = 'EU'    → EU-West Shard   (AWS eu-west-1)
user.region = 'APAC'  → AP-SE Shard     (AWS ap-southeast-1)
user.region = 'LATAM' → SA-East Shard   (AWS sa-east-1)
```

### Advantages

- **Low latency:** User data is physically near the user (5ms ping in-region vs 150ms cross-region)
- **Legal compliance:** GDPR requires EU resident data to remain within EU borders
- **Blast radius reduction:** An EU data center outage doesn't affect US shard
- **Cost optimization:** Region-specific compliance operations (audit logs, encryption) stay contained

### Disadvantages

- **Cross-region queries are slow:** 150–300ms roundtrip for synchronous cross-region calls
- **User relocation complexity:** User moves from US to EU → which shard owns their data? Migration required.
- **Uneven shard sizes:** US has 40% of global users → US shard is 2× bigger than others
- **Complex failure modes:** Region partition means some users can't access data from another region
- **Higher operational cost:** Multi-region infrastructure is significantly more expensive

### Real-World Use

WhatsApp uses regional data centers with routing based on registration region. Cloudflare distributes user data to edge data centers geographically. Banking apps and healthcare platforms are legally required to use geo-sharding to comply with data residency laws.

---

## Strategy Comparison Matrix

```
┌──────────────────┬─────────────┬──────────────┬──────────────┬─────────────────┐
│ Dimension        │ Range-Based │ Hash-Based   │ Dir-Based    │ Geo-Based       │
├──────────────────┼─────────────┼──────────────┼──────────────┼─────────────────┤
│ Write Dist.      │ Uneven 🔴   │ Even ✅       │ Flexible ✅   │ Varies          │
│ Range Queries    │ Fast ✅      │ Slow 🔴       │ Depends      │ Regional only   │
│ Resharding       │ Moderate    │ Hard 🔴       │ Easy ✅       │ Very Hard 🔴     │
│ Complexity       │ Low ✅       │ Low ✅        │ High 🔴       │ Very High 🔴     │
│ Hotspot Risk     │ High 🔴      │ Low ✅        │ Manageable   │ Medium          │
│ Compliance       │ No          │ No           │ No           │ Yes ✅           │
│ Latency          │ Normal      │ Normal       │ +1 hop 🔴    │ Low (local) ✅   │
│ Infrastructure   │ Simple ✅    │ Simple ✅     │ Medium       │ Multi-region 🔴  │
└──────────────────┴─────────────┴──────────────┴──────────────┴─────────────────┘
```

### Decision Guide

| Scenario | Recommended Strategy |
|---|---|
| Time-series data (logs, metrics, events) | Range-based on timestamp + hot-shard mitigation |
| User data with random access patterns | Hash-based on user_id (with consistent hashing) |
| Multi-tenant SaaS needing flexibility | Directory-based with tenant_id |
| EU data residency / global latency requirements | Geo-based |
| Most real production systems | Combination of the above |

---

## Part 4 Key Takeaways

- Range-based: great for range queries, terrible for write hotspots with monotonic keys
- Hash-based: great write distribution, terrible resharding (80% data moves when adding 1 shard)
- Directory-based: maximum flexibility, extra hop and critical SPOF
- Geo-based: compliance and latency, most operationally complex
- Production systems often combine strategies — e.g., hash-based on user_id, with geo-based routing on top for regional isolation

---

# Part 5 — Shard Key Selection

This is the most critical section. A bad shard key is nearly impossible to fix without a painful, risky online migration. Treat this decision with the same gravity as designing your core data model.

---

## What Makes a Good Shard Key

### Rule 1: High Cardinality

The shard key must have many distinct values. If it has only a few possible values, you can never have more shards than values.

```
Bad candidates:
  gender         → 2–3 values → maximum 3 shards
  account_status → 4 values (active/inactive/suspended/deleted)
  is_premium     → 2 values → maximum 2 shards

Good candidates:
  user_id        → billions of values → can have thousands of shards
  order_id       → billions of values
  session_id     → billions of unique values
```

### Rule 2: Even Distribution

Values must be roughly evenly distributed across the key space. Some keys have high cardinality but uneven distribution.

```
country: 195 possible values, but:
  US alone has 40% of global internet users
  → US shard handles 40% of all traffic
  → Other 194 countries share 60%
  → Very uneven
  
Contrast: hash(user_id) produces uniform distribution by design
```

### Rule 3: Alignment with Query Patterns

The most frequent queries should be satisfiable from a single shard. Cross-shard queries are expensive.

```
If 80% of queries are "get all orders for user X":
  → Shard by user_id ✅
  → All of user's orders are on one shard
  → Single shard lookup

If 80% of queries are "get all orders in region X":
  → Shard by region ✅
  → Sharding by user_id would scatter region data across all shards ❌
```

**Ask yourself:** "What is the most common WHERE clause in our most critical queries?"

### Rule 4: Immutability

If the shard key value changes, the record must move to a different shard. This requires: deleting from old shard, inserting to new shard, updating routing table — in a coordinated transaction. This is extraordinarily complex.

**Rule: Choose a shard key that never changes.**

```
Good: user_id (assigned once, never changes)
Good: order_id (assigned at creation, never changes)
Bad:  email (users change emails)
Bad:  username (users change usernames)
Bad:  account_status (changes constantly)
```

### Rule 5: Avoids Monotonic Write Patterns

A shard key that grows monotonically (always increasing) concentrates new writes on the "latest" shard.

```
auto-increment id: 1, 2, 3, 4, 5...
  New records always have the highest ID → always go to the "latest" shard

timestamp: 2024-01-01, 2024-01-02, 2024-01-03...
  New records always have the latest timestamp → always go to the "current" shard
  
Solution: Use hash(auto_increment_id) as effective shard key
→ Sequential IDs produce uniform hashes
```

---

## Analysis of Common Shard Key Candidates

### User ID

**Sequential auto-increment user_id:**
```
user_id: 1, 2, 3, ... 10,000,000
Range shard: new users always go to latest range → hotspot ❌
Hash shard: hash(10000001) % 8 = uniform → good ✅
```

**UUID:**
```
UUID: 550e8400-e29b-41d4-a716-446655440000
hash(uuid) % 8 = 2  → Good uniform distribution ✅
But: no ordering, harder to debug ("which shard is this user on?")
```

**Verdict:** User_id (auto-increment) with hash-based sharding is excellent for social/consumer products.

### Email

```
Routing: hash('alice@gmail.com') % 8 = 3 → Shard 3
```

**Problems:**
- Mutable: users change email → record must move ❌
- Extra lookup: many systems have user_id in events, not email → must look up email first to route
- External dependency: ties your shard key to an external identifier

**Verdict:** Generally bad. Use internally assigned immutable user_id instead.

### Country / Region

```
US → Shard 1  (40% of users)
EU → Shard 2  (25% of users)
APAC → Shard 3  (20% of users)
Other → Shard 4  (15% of users)
```

**Problems:**
- Inherently uneven (US shard is 2× EU shard) ❌
- User can change country ❌
- Very low cardinality if used alone ❌

**Use only for:** Geo-compliance requirements (GDPR), combined with another shard key.

### Timestamp

```
created_at range → shard:
  Jan 2024 → Shard 1  (all reads, cold)
  Feb 2024 → Shard 2  (all reads, cold)
  Mar 2024 → Shard 3  (100% of writes 🔥)
```

**When it's acceptable:** Append-only logs, audit trails, time-series metrics where you query by time ranges AND you plan to archive/retire old shards on a schedule.

**When it's terrible:** Any write-heavy, live system where old shards are cold and current shard is a hot write target.

### Tenant ID (Multi-Tenant SaaS)

```
company_a → Shard 1
company_b → Shard 2
company_c → Shard 2
```

**Good:**
- All queries within a tenant stay on one shard ✅
- Natural tenant isolation ✅
- Tenant-scoped transactions are fully ACID ✅

**Problems:**
- Large "whale" tenants create hot shards ❌
- New tenants all hash to existing shards ❌ (unless using directory-based assignment)

**Solution for whale tenants:**
```
Identify tenants with >10× average data volume or traffic
Give them dedicated shards:

company_a (whale, 500K users) → dedicated Shard 1
company_b (1K users)         → Shard 2 (shared)
company_c (800 users)        → Shard 2 (shared)
company_d (2K users)         → Shard 3 (shared)
```

---

## The Celebrity / Viral Content Problem

This is a famous interview topic and a real production nightmare.

**Setup:** You shard by user_id (hash-based). Justin Bieber has 100 million followers.

```
user_id = JB_ID → hash(JB_ID) % 8 = 3 → Shard 3

When ANY of his 100M followers loads their feed:
  "Get latest posts from user JB_ID"
  → Routes to Shard 3

At peak: 10M followers loading feed simultaneously
  → 10M queries/second all hitting Shard 3
  → Shard 3 CPU: 100% 🔥
  → Shard 3 disk I/O: saturated 🔥
  → Other 7 shards: 10% utilization
```

**This cannot be solved purely through sharding strategy.** The root cause is one entity having 100M relationships.

**Solutions:**

1. **Aggressive caching:** Store Justin Bieber's 20 most recent posts in Redis with a TTL. Shard 3 only serves cache misses.

2. **Fan-out on write:** When JB posts, push to all 100M followers' precomputed feed caches. At read time, no shard is hit for JB's data. (Twitter does this for non-celebrity accounts.)

3. **Dedicated read replicas for hot users:** Detect top 1000 most-followed users, create additional read replicas for those shards specifically.

4. **Application-level hot key detection:** Track per-user-id query counts in Redis. When a user_id exceeds threshold, route reads to a cached copy instead of the shard.

---

## How Bad Shard Keys Destroy Scalability: Case Studies

**Case Study: E-commerce sharding by product_category**

```
Electronics → Shard 1
Books       → Shard 2
Clothing    → Shard 3

Prime Day arrives:
  Electronics: 70% of all traffic → Shard 1 at 200% capacity
  Books, Clothing: mostly idle
  
Cannot fix without full resharding while staying online ← catastrophic
```

**Case Study: Social network sharding by username first letter**

```
A–F → Shard 1
G–M → Shard 2
N–Z → Shard 3

More common names start with common letters (John, James, Jennifer...)
→ Shard 2 consistently overloaded
→ Cannot rebalance without expensive data migration
```

---

## Part 5 Key Takeaways

- Good shard key: high cardinality, even distribution, aligns with primary query patterns, immutable, not monotonic
- Bad shard key: timestamps (hotspot), low cardinality fields (gender, country alone), mutable fields (email)
- Whale tenants and celebrity users require separate handling beyond shard key choice — caching is mandatory
- Changing a shard key in production is one of the most expensive database operations possible — get it right upfront

---

# Part 6 — Production Challenges

## Challenge 1: Hot Shards

### Why They Happen

1. **Bad shard key:** Monotonically increasing keys, low cardinality keys
2. **Viral content / celebrity accounts:** One user's data receives disproportionate traffic
3. **Seasonal spikes:** Black Friday traffic concentrates on certain product categories
4. **Organic skew:** Some regions or user segments grow much faster than others
5. **Promotional events:** A marketing campaign drives 10× traffic to specific shards

### Detection

Monitor **per-shard metrics** independently:

```
Metrics to track per shard:
  - QPS (queries per second)
  - CPU utilization %
  - Disk I/O ops/sec
  - Read/write latency (p50, p95, p99)
  - Connection count
  - Replication lag (if shard has replicas)

Alert conditions:
  - Any shard exceeds 1.5× average QPS → investigation
  - Any shard exceeds 80% CPU for >5 minutes → page
  - Per-shard latency diverges >2× from others → investigation
```

Tools: Prometheus with per-shard labels, Datadog APM traces with shard tag, custom shard metrics emitted by your routing layer.

### Prevention and Mitigation

1. **Choose the right shard key** (prevents 90% of hotspot issues)
2. **Add read replicas to the hot shard** (immediate relief for read-heavy hotspots)
3. **Cache hot data aggressively** (Redis for hot users, popular items, trending content)
4. **Move hot keys to dedicated shards** using directory-based routing
5. **Application-level hot key detection:** count per-key QPS, route above-threshold keys to cache
6. **Consistent hashing with virtual nodes:** better baseline distribution

---

## Challenge 2: Resharding

### Why It Becomes Necessary

- Initial shard count is insufficient for growth
- Shard key was poorly chosen (time to change it)
- One shard has grown much larger than others (split it)
- Underutilized shards should be merged to reduce operational overhead

### The Naive Resharding Problem

With simple hash-based sharding (hash % N):

```
Adding shard 5 to a 4-shard cluster:
  4 shards: user_id=1001 → hash(1001) % 4 = 2 → was on Shard 2
  5 shards: user_id=1001 → hash(1001) % 5 = 1 → should be on Shard 1

  ~80% of all keys are now on the "wrong" shard
  All must be moved → massive migration
```

### Migration Strategies

**Strategy A: Stop-the-World (Simple, Only for Small Data)**

1. Announce maintenance window
2. Stop all writes
3. Run migration job: read from old shards, write to new shards
4. Verify checksums
5. Update routing configuration
6. Resume traffic

*Acceptable only for small datasets (<100GB) with a tolerant SLA. Unacceptable at scale.*

**Strategy B: Online Dual-Write Migration (No Downtime)**

```
Phase 1 — Stand up new infrastructure:
  Provision new shard configuration (e.g., 8 shards instead of 4)

Phase 2 — Dual Write:
  All new writes go to BOTH old shards AND new shards simultaneously
  Route reads to old shards (authoritative)
  Start background job to copy historical data to new shards

Phase 3 — Verify:
  Background migration job completes
  Run checksums: verify new shards have correct, complete data
  Shadow reads: compare results from old vs new shards

Phase 4 — Cut Over Reads (Gradual):
  Route 1% of reads to new shards → monitor error rates and latency
  Increase to 10%, 50%, 100% over hours/days
  Feature flags and canary deployments for safety

Phase 5 — Stop Old Writes:
  Remove dual-write; new shards are now authoritative
  Keep old shards read-only for 1–2 weeks as safety net

Phase 6 — Decommission:
  Old shards can now be safely turned off
```

**Strategy C: Use Consistent Hashing (Proactive)**

With consistent hashing (Part 7), adding one node moves only 1/N of total data (~12.5% for an 8-node cluster), not 80%. Design your system with consistent hashing from the start to make resharding far less painful.

### Risks During Resharding

- **Data inconsistency:** If migration fails mid-way, some records are on old shards, some on new
- **Increased load:** Background migration + production traffic can overwhelm hardware
- **Application complexity:** During migration, a key might be on either shard → routing must handle both
- **Rollback difficulty:** Sometimes rolling back a failed migration is harder than completing it
- **Duration:** At 100MB/sec throughput, migrating 1TB takes ~3 hours → long risk window

---

## Challenge 3: Cross-Shard Queries

This is one of the most persistent operational pains with sharding.

### Aggregations

```sql
-- Single shard (fast):
SELECT COUNT(*) FROM orders WHERE user_id = 1001;
-- → Routes to shard 3, runs query, returns count

-- Cross-shard (expensive scatter-gather):
SELECT COUNT(*) FROM orders WHERE created_at > '2024-01-01';
-- → Must query ALL N shards in parallel
-- → Collect N counts, sum them in application code
-- → N × query latency (serial) or ~1 × latency (parallel) + aggregation overhead
```

### Sorting + Pagination

```sql
-- "Get the 10 most recent orders"
-- Each shard returns its top 10 → gather 40 results (4 shards × 10)
-- Application sorts all 40, returns top 10

-- "Get page 5 of results (rows 41–50)"
-- Each shard must return top 50 → gather 200 results → sort → return rows 41–50
-- Cost grows linearly with page depth: deep pagination is very expensive
```

This is why most sharded systems limit pagination depth (e.g., Twitter's timeline limit) or use cursor-based pagination instead of offset-based.

### Joins

```sql
-- If users and orders are on different shards:
SELECT u.name, o.total
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US';

-- Cannot be done as a native SQL join
-- Option 1: Fetch US user IDs from user shards, then query order shards by user_id
-- Option 2: Denormalize user country into the orders table (co-locate data)
-- Option 3: Push to analytics system and query there
```

### Solutions

**1. Design to avoid cross-shard queries (preferred):**
Co-locate related data on the same shard. If users and orders share the same shard key (user_id), all of a user's data is on one shard.

**2. Denormalization:**
Copy data that's needed together onto the same shard. Store `user_country` in the orders table so order queries don't need to join with the users shard.

**3. Fan-out + Aggregate:**
Query all shards in parallel, aggregate results in the application. Acceptable for infrequent operations; never for hot paths.

**4. Global Secondary Indexes:**
DynamoDB and Cassandra support indexes that span all partitions. Querying the index causes the database to internally fan-out. Convenient but expensive for writes (every write must update the global index).

**5. Data Warehouse / OLAP Layer:**
For complex analytical queries, ETL data from all shards to a single OLAP database (Redshift, BigQuery, Snowflake). Run analytics there. The OLTP shards never handle analytical workloads.

**6. CQRS (Command Query Responsibility Segregation):**
Maintain separate read models — denormalized, possibly non-sharded — specifically optimized for complex query patterns. Write side is sharded for throughput; read side is a different data store optimized for reads.

---

## Challenge 4: Distributed Transactions

### The Problem

Classic ACID transactions assume all data is in one database. With sharding:

```
Transfer $100 from Alice (Shard 1) to Bob (Shard 3):

  Step 1: Debit Alice's account on Shard 1   ← succeeds
  Step 2: Credit Bob's account on Shard 3    ← server crashes

Result: $100 is gone from Alice but never arrived in Bob's account.
Money has vanished from the system. ← catastrophic correctness violation
```

### Solution 1: Two-Phase Commit (2PC)

```
Coordinator node orchestrates:

PHASE 1 — PREPARE:
  Coordinator → Shard 1: "Prepare to debit Alice $100"
  Coordinator → Shard 3: "Prepare to credit Bob $100"
  
  Shard 1: locks Alice's record, writes prepare entry to WAL
  Shard 3: locks Bob's record, writes prepare entry to WAL
  
  Shard 1 → Coordinator: "Ready"
  Shard 3 → Coordinator: "Ready"

PHASE 2 — COMMIT:
  Coordinator → Shard 1: "Commit"
  Coordinator → Shard 3: "Commit"
  Both shards commit and release locks
```

**Why 2PC is rarely used at scale:**
- **Coordinator SPOF:** If coordinator crashes after sending PREPARE but before COMMIT, both shards hold locks indefinitely — system stalled until recovery
- **Lock duration:** Locks are held across two network round trips → high latency impact
- **Scalability:** Every distributed transaction requires O(N) network messages
- **Blocking:** A single slow shard can block the entire transaction

### Solution 2: Saga Pattern

Break a distributed transaction into a sequence of local transactions, each with a compensating transaction that can undo it.

```
Transfer Saga:
  T1: Debit Alice $100 on Shard 1
      Compensating C1: Credit Alice $100 on Shard 1

  T2: Credit Bob $100 on Shard 3
      Compensating C2: Debit Bob $100 on Shard 3

Execution:
  Execute T1 → Success
  Execute T2 → FAILURE

On T2 failure, run compensations:
  Execute C1: Credit Alice $100 → money restored
  System returns to consistent state
```

**Properties:**
- No global locks → high throughput
- Eventual consistency — during execution, system is temporarily inconsistent
- Application must implement and test compensation logic
- Retries must be idempotent

**Widely used in:** Microservices architecture, payment systems, order processing.

### Solution 3: Eventual Consistency

For many use cases, strict immediate consistency across shards isn't required. Accept that:
- After a distributed operation completes, all shards will *eventually* converge to a consistent state
- Short periods of temporary inconsistency are acceptable

**Acceptable for:** Social media feeds (seeing a post 1 second late is fine), product catalog updates, user preference changes, read counts and likes.

**Not acceptable for:** Financial ledgers (no money creation/destruction), inventory (can't oversell), authentication (can't allow duplicate sessions).

Amazon's foundational Dynamo paper argued: "Accept that sometimes you'll see stale data. Design your system to tolerate and recover from temporary inconsistencies rather than paying the cost of enforcing them."

---

## Part 6 Key Takeaways

- Hot shards are detected via per-shard metrics; fixed with caching, read replicas, or manual shard splitting
- Resharding without downtime requires dual-write migration + gradual traffic cutover; plan for weeks of work
- Cross-shard queries are expensive; solve by designing data co-location, denormalization, and OLAP offload
- 2PC is theoretically correct but operationally fragile; Saga is the production-preferred pattern; eventual consistency is sufficient for most cases

---

# Part 7 — Advanced Concepts

## Consistent Hashing

### Why It Exists

Simple hash-based sharding (`hash(key) % N`) requires ~80% data movement when adding a shard. This makes scaling painful and risky.

Consistent hashing was invented to solve this: **adding or removing a node moves only ~K/N keys** (where K = total keys, N = number of nodes), instead of almost all keys.

### The Ring Architecture

Imagine wrapping the hash space (0 to 2³² – 1) into a circle (ring).

```
              0 (same as 2^32)
         350      10
      330    Ring    30
    310               50
   290                  70
  270 ─────────────── 90
  250                  110
   230               130
    210             150
      190         170
            180

Servers placed at positions on the ring:
  Server A: position 50
  Server B: position 150
  Server C: position 250
  Server D: position 350

Routing rule: hash(key) → position on ring
             → traverse clockwise → first server you hit = your shard

Example:
  hash('user:1001') = 80  → clockwise from 80 → Server B (150)
  hash('user:2002') = 200 → clockwise from 200 → Server C (250)
  hash('user:3003') = 320 → clockwise from 320 → Server D (350)
```

### Adding a Node

```
Before: A(50)───B(150)───C(250)───D(350)
        Each server owns ~25% of ring

Add Server E at position 100:
After: A(50)─E(100)─B(150)─C(250)─D(350)

Only keys between position 50 and 100 (previously owned by B)
now belong to E. That's ~12.5% of B's keys = ~3% of total keys.
→ Only 3% of total data moves! (vs 80% with simple hash mod)
```

### Virtual Nodes

**Problem:** With few physical servers, they might cluster on one half of the ring, creating uneven distribution.

**Solution:** Each physical server gets multiple "virtual nodes" spread around the ring.

```
Physical servers and their virtual nodes:

Server A: positions [10, 90, 170, 250]  ← spread across ring
Server B: positions [30, 110, 190, 330]
Server C: positions [50, 130, 210, 350]
Server D: positions [70, 150, 230, 370→10]

Each server now owns many small segments of the ring.
Total ownership is much more even.
Cassandra default: 256 virtual nodes per physical node.
```

**Adding a physical node:** It claims virtual nodes from across the ring, taking a small slice from each existing server. Rebalancing is smooth and parallel.

---

## How Major Distributed Databases Implement Sharding

### Apache Cassandra

- **Partitioning:** Consistent hashing with virtual tokens
- **Routing:** Any node can serve any request (no router needed); each node knows the full ring topology
- **Replication:** Each partition is replicated to N nodes (configurable replication factor), next N clockwise nodes on ring
- **Consistency:** Tunable per query: `ONE`, `QUORUM`, `ALL`. Write to QUORUM, read from QUORUM → strong consistency without single point of failure
- **No joins:** Cassandra provides no JOIN capability. You denormalize data for each query pattern.
- **Resharding:** Add node → Cassandra automatically streams data to the new node from existing neighbors

### DynamoDB (AWS)

- **Partitioning:** Hash-based on partition key. AWS manages partition assignment internally.
- **Auto-scaling:** Partitions automatically split when a partition exceeds 10GB or 3,000 WCU
- **Fully managed:** You never see or manage shards directly
- **Global secondary indexes:** Maintained automatically; internally fan out writes to an index partition
- **Hot partition detection:** DynamoDB emits per-partition metrics. Design partition keys to avoid hot partitions (use high-cardinality keys, add randomized suffix if needed).
- **Transactions:** Supports ACID transactions across up to 25 items using an internal protocol

### MongoDB

- **Routing:** `mongos` query router processes sit between application and shards
- **Metadata:** Config servers (replicated) store chunk-to-shard mapping
- **Partitioning:** Range-based or hash-based on shard key
- **Chunks:** Data is divided into chunks (64MB default). Balancer daemon moves chunks between shards to equalize
- **Chunk migration:** Source shard sends chunk data to destination while capturing ongoing changes; config server atomically updates mapping; clients redirect
- **Resharding:** MongoDB 5.0+ supports online resharding (changing shard key without taking down the cluster)

### CockroachDB

- **Architecture:** Distributed SQL, PostgreSQL wire-protocol compatible
- **Partitioning:** Automatic range-based sharding called "ranges" (64–512MB each)
- **Consensus:** Each range is managed by a Raft group (typically 3 replicas)
- **ACID across ranges:** Supports full ACID transactions across any number of ranges using a distributed transaction protocol with parallel commits (reduces cross-range transaction latency vs 2PC)
- **Geo-partitioning:** Pin ranges to specific regions for GDPR compliance
- **Resharding:** Fully automatic — ranges split when too large, merge when too small, rebalance across nodes

### Vitess (MySQL Sharding Middleware)

- **What it is:** A middleware layer that makes MySQL horizontally scalable
- **Used by:** YouTube (since 2011), GitHub, Slack, PlanetScale (built on Vitess)
- **VSchema:** Defines which columns are shard keys and how tables are sharded
- **VTGate:** The routing layer — receives queries, plans them across shards
- **Online schema changes:** Vitess uses gh-ost for non-blocking DDL across all shards — critical for MySQL which doesn't support online DDL well
- **Trade-off:** Applications get nearly normal MySQL semantics; operationally more complex than native MySQL but far simpler than building sharding from scratch

### YugabyteDB

- **Architecture:** Distributed SQL + document API, PostgreSQL-compatible
- **Storage:** DocDB, a log-structured storage engine using Raft per shard
- **Auto-sharding:** Automatic tablet (shard) creation and rebalancing
- **ACID:** Full PostgreSQL-compatible ACID transactions across shards
- **Best of both worlds:** SQL semantics + horizontal scalability + automatic rebalancing
- **Use case:** When you want PostgreSQL but need true horizontal write scaling

---

## Part 7 Key Takeaways

- Consistent hashing solves the resharding problem: adding 1 node moves ~1/N data instead of ~80%
- Virtual nodes spread load more evenly and enable smoother rebalancing
- Cassandra: eventual consistency, no joins, best for high-throughput write workloads
- DynamoDB: fully managed, auto-scaling, good for serverless/cloud-native
- CockroachDB and YugabyteDB: true distributed ACID SQL — close to PostgreSQL semantics
- Vitess: add sharding to existing MySQL with minimal application changes — proven at YouTube scale

---

# Part 8 — System Design Interviews

## How to Sequence Sharding in an Interview

**Wrong approach (fail signal):**
"We'll shard by user_id from day one using 8 shards."

This signals cargo-culting. You're applying a complex solution before justifying the need.

**Correct progression:**

```
1. Single DB
   → "Start here, simple, all features work naturally"

2. Add caching (Redis)
   → "Reduces read pressure by ~80% for hot data"

3. Add read replicas
   → "Scale reads horizontally, primary handles only writes"

4. Add message queue (Kafka/SQS)
   → "Decouple write spikes, async processing"

5. JUSTIFY sharding:
   "At X write/sec and Y TB of data, single primary is a bottleneck.
    Let me introduce sharding."
   
6. Introduce sharding with full trade-off discussion
```

## How to Justify Sharding in an Interview

Concrete language:

> "At our scale, we're handling ~50K writes/sec. A single PostgreSQL primary can handle ~10K–50K simple writes/sec under ideal conditions. We're approaching that ceiling, and our data size at 8TB is getting expensive to manage on a single instance with adequate replication. This is where I'd introduce sharding."

## The Trade-Off Discussion Interviewers Want

Don't just say "we'll shard by user_id." Walk through the trade-offs:

> "Sharding by user_id with hash-based distribution gives us:
>
> **Benefits:** Uniform write distribution, all user data co-located on one shard for user-scoped queries, simple routing logic.
>
> **Costs:** Cross-user queries (trending topics, global analytics) require scatter-gather across all shards; analytics must go to a data warehouse; deep pagination is expensive; celebrities with 100M followers create hot shards regardless of distribution.
>
> **Mitigations:** Cache hot user data in Redis, precompute trending metrics asynchronously, offload analytics to Redshift via CDC pipeline, use consistent hashing so adding shards only moves ~12.5% of data instead of 80%."

## Common Interview Mistakes

| Mistake | What It Signals | Better Approach |
|---|---|---|
| Shard from day one without justification | Cargo-culting, no systems thinking | Show the scaling journey that leads to sharding |
| Pick shard key without discussing trade-offs | Shallow understanding | Always name the access patterns, distribution, and hotspot risks |
| Forget cross-shard operations | Missed real complexity | Proactively address scatter-gather and analytics offload |
| Never mention resharding | Treating sharding as static | Acknowledge that shards grow and must be added over time |
| Ignore hot key problem | Missing a famous real problem | Always mention celebrity problem and caching mitigation |
| Miss the routing layer | Architecture is incomplete | Explicitly mention how queries find the right shard |

---

## System Walkthroughs

### URL Shortener

Scale: 100B URLs, 10K writes/sec, 100K reads/sec

Primary access pattern: `SELECT long_url FROM urls WHERE short_code = 'abc123'`

**Shard key:** `hash(short_code)` — every read is a single key lookup

```
Write: hash('abc123') % 8 = 5 → write to Shard 5
Read:  hash('abc123') % 8 = 5 → read from Shard 5
```

Cross-shard concern: "Get all URLs created by user X" → secondary access, offload to separate user-activity table sharded by user_id, or send to analytics warehouse.

### WhatsApp

Scale: 100B messages/day, 2B users, real-time delivery

**Shard key:** `conversation_id` (not user_id)

Why: Primary access pattern is "get messages in conversation X." With conversation-based sharding, all messages in a thread are on one shard. With user-based sharding, two participants' message history would split across two shards.

Cross-shard concern: "Count all unread messages across all conversations" → separate inbox_count service sharded by user_id.

### Twitter/X (the hardest one)

Scale: 500M tweets/day, timelines of 1000+ follows

**Shard key for tweets:** `tweet_id` (hash-based) OR `author_id`

**The timeline problem:**
```
User follows 1000 accounts. Their timeline = latest tweets from all 1000.
Those 1000 accounts are on up to 1000 different shards.

Option A — Fan-out on Read (pull model):
  At timeline load, query shards of all 1000 followed accounts
  Merge-sort results
  → O(1000 shard queries) per timeline load → terrible at scale

Option B — Fan-out on Write (push model):
  When someone tweets, push tweet to all followers' precomputed timeline caches
  At timeline load, read from user's own timeline cache (single key lookup)
  → O(followers) writes per tweet → Katy Perry tweet → 150M Redis writes

Hybrid (Twitter's actual approach):
  Normal users: fan-out on write → push to followers' caches
  Celebrity accounts (>1M followers): fan-out on read → fetch at timeline load
  This limits fan-out-on-write to accounts with manageable follower counts
```

### Uber

Scale: 15M trips/day, 100K concurrent trips at peak, GPS ping every 4 seconds

**Active trips:** Redis (in-memory, sub-millisecond, short-lived data)

**Completed trips:** Sharded PostgreSQL by `trip_id` (hash-based)

**Driver/rider data:** Sharded by `user_id` (hash-based)

**Nearby drivers:** NOT in sharded SQL → dedicated geospatial service (S2 cells, R-tree index)

**Analytics:** BigQuery — ETL from sharded DBs via CDC (Debezium → Kafka → BigQuery)

---

## Part 8 Key Takeaways

- Always justify sharding after showing the scaling journey (single DB → cache → replicas → sharding)
- Every shard key discussion must include: distribution quality, query alignment, hotspot risk, resharding plan
- Know the famous problems (celebrity/viral, cross-shard pagination, analytics offload) and their solutions
- Twitter's fan-out hybrid is a canonical example interviewers love — know it cold

---

# Part 9 — Real Production Case Studies

## Case Study 1: Instagram

### Initial Architecture (2010)

- Single Django server
- Single PostgreSQL database
- Launched with 1 engineer (Mike Krieger) maintaining the stack

### Scaling Pain Points (2011–2012, 30M users)

- Photo upload writes + follow actions + like events overwhelming single primary
- CPU on primary consistently above 80%
- Disk write I/O saturated during peak hours
- Read replicas helped reads but couldn't address write bottleneck

### Sharding Solution

- **Shard key:** `user_id`
- **Strategy:** 12 logical shards, each a PostgreSQL schema within the same cluster initially, then separate server instances
- **Routing:** Custom Django ORM layer that routed queries based on `user_id % 12`
- **Data co-location:** User photos, followers, and settings all on the same shard as the user record

**Later evolution (post-Facebook acquisition):**
- Thousands of logical shards
- Cassandra for activity feeds (better write throughput, no complex queries needed)
- memcached cluster serving most read traffic
- Sharded MySQL for core user/media metadata

### Lessons Learned

1. **Delay sharding:** They sharded after feeling real pain, not speculatively. This kept early velocity high.
2. `user_id` is an excellent shard key for social products — most queries are user-scoped.
3. Separate your read store (Cassandra/cache) from your write store (sharded PostgreSQL).
4. Cache aggressively — Instagram served millions of photos/day, majority from memcached without hitting DB.

---

## Case Study 2: Uber

### Initial Architecture (2009–2014)

- Monolithic Python application
- Single PostgreSQL database
- Fine at low trip volume

### Scaling Pain Points (2015–2016)

- GPS ping frequency: each active driver sends location every 4 seconds
- At 100K concurrent trips: 25,000 writes/second just for location updates
- Trip completion events, surge pricing calculations, payment events all competing for primary I/O
- Primary hitting disk I/O ceiling at peak hours in major cities

### Sharding Solution

**Active trips:** Moved to Redis — trips are short-lived (average <30 minutes), high-frequency writes, sub-millisecond read needed. Redis perfectly fits the pattern.

**Completed trips and historical data:** Sharded PostgreSQL by `trip_id` (hash-based). These are write-once (trip closes → written → rarely updated), so hotspot risk is low.

**Schemaless (Uber's proprietary sharding layer):**
Uber built an internal sharding system called Schemaless on top of MySQL. It provided:
- Row-key → JSON value storage (flexible schema)
- Sharding transparent to application code
- Built-in sharding expansion without application changes

```
Application → Schemaless Client
                    ↓
              Shard Router (consistent hashing)
            /     |      \
        MySQL-1  MySQL-2  MySQL-3
```

### Lessons Learned

1. Separate your hot path (active trips, real-time GPS) from cold path (historical trips, analytics).
2. Building a custom sharding abstraction layer (Schemaless) gave Uber flexibility to change underlying sharding strategies without application changes.
3. GPS write frequency is brutal — design for 10× your expected write throughput from day one.
4. Geospatial queries ("find nearby drivers") are completely separate from your transactional sharding — need a dedicated geo-index service.

---

## Case Study 3: Amazon Orders

### Scale

- Hundreds of millions of unique customers
- Hundreds of billions in annual orders
- Orders accessed for customer service, returns, recommendations, tax reporting, fraud detection

### Sharding Approach

**Primary shard key:** `customer_id` (hash-based)

Why: The dominant query pattern is "Show this customer's order history." Sharding by customer_id means all of a customer's orders are on one shard → single-shard query, full ACID within one customer's data.

**Secondary access patterns:**
- Merchant's view: "Show me all orders for my products" → separate table sharded by `seller_id`
- Global analytics: → separate Redshift data warehouse, ETL from operational shards

**Fundamental principle — OLTP vs OLAP separation:**

```
OLTP (sharded operational DB):
  Sharded by customer_id
  Query: "Show my order history" → single shard ✅
  Response time: <50ms

OLAP (Redshift data warehouse):
  Non-sharded or column-store sharded differently
  Query: "Revenue by product category in Q3 globally" ✅
  Response time: seconds to minutes (acceptable for analytics)
  
ETL pipeline: CDC from operational shards → Kafka → Redshift
              (data arrives in warehouse within minutes)
```

### Lessons Learned

1. Never try to serve OLTP and OLAP from the same database — they have fundamentally different access patterns, latency requirements, and query shapes.
2. Design your shard key around the primary access pattern — "get orders for customer" is the core query.
3. Secondary access patterns (merchant view, analytics) need their own data stores with appropriate shard keys.
4. At Amazon's scale, EVERY service has its own database (microservices pattern). The "orders" domain isn't one service — it's dozens of services each with their own storage.

---

## Case Study 4: Netflix

### Scale

- 270M+ subscribers globally
- 3M+ content titles (including seasons, episodes)
- Billions of viewing events per day
- Recommendation system consuming all viewing history

### Data Categories and Their Solutions

**Content metadata (titles, descriptions, ratings):**
- Relatively small dataset (~TBs)
- Read-heavy, rarely written
- Cached aggressively in CDN and in-memory
- Cassandra for the key-value store with geographic distribution

**Viewing history (what did user X watch):**
- Massive write throughput (billions of events/day)
- Cassandra sharded by `user_id` — partition key
- All viewing events for a user on one partition → timeline reconstruction is fast
- Compaction strategies tuned for write-heavy time-series data

**Streaming events (bitrate changes, buffering events, quality metrics):**
- Extremely high volume
- Not queried in real-time by end users
- Written to Kafka streams → consumed by Flink for real-time analysis → eventually lands in S3/Iceberg for historical analysis

**Personalization data (recommendations, taste profiles):**
- Not served from operational DB at all
- ML models trained offline, predictions materialized to Redis/Cassandra
- At request time: simple key lookup `get_recommendations(user_id)` → Redis

### Lessons Learned

1. Not all data lives in the same storage system — Netflix uses: Cassandra, MySQL, Redis, Elasticsearch, Kafka, S3, Spark, and dozens of other stores.
2. Streaming data (viewing events) and serving data (recommendations) have completely different shapes and should be separate.
3. Precompute wherever possible — computing recommendations at query time for 270M users is impossible. Precompute daily/hourly and store results for fast serving.
4. Geographic distribution is mandatory at Netflix's scale — each region has its own Cassandra clusters.

---

# Part 10 — Mastery Section

## Complete Mental Model of Sharding

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
           THE SHARDING MENTAL MODEL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRIGGER:
Single DB cannot handle:
├── Write throughput (>50K writes/sec sustained)
├── Data size (>5–10TB on a single node)
└── CPU (primary CPU consistently >80%)

PREREQUISITE CHECKS:
□ Is caching (Redis) already deployed?
□ Are read replicas deployed?
□ Have we exhausted vertical scaling?
□ Is a managed distributed DB (DynamoDB, Spanner) better?

DECISION: Shard

STEP 1 — Choose Shard Key:
├── Identify primary query pattern (most frequent WHERE clause)
├── Verify high cardinality
├── Verify even distribution
├── Verify immutability
└── Verify non-monotonic (or plan hot-shard mitigation)

STEP 2 — Choose Strategy:
├── Range: time-series, range queries needed, accept hotspot risk
├── Hash: uniform writes, no range queries, must use consistent hashing
├── Directory: multi-tenant SaaS, whale tenants, need flexibility
└── Geo: compliance, regional latency, accept cross-region complexity

STEP 3 — Handle Cross-Shard Concerns:
├── Co-locate related data (most important)
├── Denormalize to avoid cross-shard joins
├── Fan-out + aggregate for unavoidable cross-shard reads
├── Analytics → data warehouse (never from shards)
└── Celebrity/viral → caching layer mandatory

STEP 4 — Handle Transactions:
├── Design data so most transactions are single-shard
├── Multi-shard transactions → Saga pattern
└── Accept eventual consistency where appropriate

STEP 5 — Plan for Growth:
├── Use logical shards (1000 logical → 4 physical today)
├── Use consistent hashing for smoother resharding
├── Add per-shard monitoring from day one
└── Document the resharding runbook before you need it

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## The One-Page Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║                    DATABASE SHARDING                             ║
║                    MASTER REFERENCE                              ║
╠══════════════════════════════════════════════════════════════════╣
║ WHAT:  Split data across N independent DB instances              ║
║ WHY:   Scale writes, storage, CPU beyond single-node limits      ║
║ WHEN:  After caching + read replicas are insufficient            ║
║        Writes >50K/sec, Data >5TB, CPU consistently >80%        ║
╠══════════════════════════════════════════════════════════════════╣
║ SHARD KEY RULES                                                  ║
║   ✅ High cardinality (millions+ of unique values)               ║
║   ✅ Even distribution across key space                          ║
║   ✅ Aligns with most frequent query's WHERE clause              ║
║   ✅ Immutable (never changes after assignment)                  ║
║   ❌ Monotonically increasing (timestamps, auto-increment IDs)   ║
║   ❌ Low cardinality (gender, boolean, status)                   ║
║   ❌ Mutable (email, username, phone)                            ║
╠══════════════════════════════════════════════════════════════════╣
║ STRATEGIES                                                       ║
║   Range:     shard by value ranges. Fast range queries.          ║
║              HOTSPOT RISK on monotonic keys.                     ║
║   Hash:      hash(key)%N. Uniform writes. Expensive resharding.  ║
║   Directory: lookup table → shard. Max flexibility. SPOF risk.   ║
║   Geo:       shard by region. Latency + compliance. Complex.     ║
╠══════════════════════════════════════════════════════════════════╣
║ PRODUCTION CHALLENGES                                            ║
║   Hot Shards:    detect via metrics, fix with cache + replicas   ║
║   Resharding:    consistent hashing minimizes data movement       ║
║   Cross-shard:   fan-out, denormalize, send analytics to DW      ║
║   Dist. Txn:     Saga pattern (compensating txns) preferred       ║
║   Celebrity:     cache hot users, fan-out on write              ║
╠══════════════════════════════════════════════════════════════════╣
║ CONSISTENT HASHING                                               ║
║   Ring + virtual nodes. Adding 1 node moves only ~1/N data.      ║
║   Used by: Cassandra, DynamoDB (internally), Riak                ║
╠══════════════════════════════════════════════════════════════════╣
║ DATABASES                                                        ║
║   Cassandra:  consistent hash, eventual consistency, no joins    ║
║   DynamoDB:   hash-based, auto-managed, serverless-friendly      ║
║   MongoDB:    range/hash, mongos router, auto chunk balancing    ║
║   Vitess:     MySQL sharding middleware (YouTube, GitHub)         ║
║   CockroachDB: auto-range sharding, distributed ACID SQL         ║
║   YugabyteDB: distributed PostgreSQL, ACID, auto-rebalance       ║
╠══════════════════════════════════════════════════════════════════╣
║ INTERVIEW FLOW                                                   ║
║   1. Start: single DB → caching → read replicas                  ║
║   2. Justify: cite specific write/storage limits                  ║
║   3. Choose shard key: state access pattern, trade-offs          ║
║   4. Choose strategy: range/hash/directory with rationale        ║
║   5. Address cross-shard: where it happens, how you handle it    ║
║   6. Address resharding: consistent hashing, migration plan       ║
║   7. Address hot keys: caching, dedicated replicas               ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Senior Engineer Interview Summary

A Senior Engineer should demonstrate:

**What you know:**
- Why sharding exists (write throughput ceiling, storage limits)
- How to choose a shard key (cardinality, distribution, query alignment, immutability)
- The four strategies and their primary trade-offs
- What cross-shard queries cost and how to avoid them
- That distributed transactions are hard and Saga is the preferred solution

**How you speak:**
- "Before introducing sharding, I'd confirm we've exhausted caching and read replicas."
- "I'd shard by user_id rather than timestamp because..."
- "The trade-off with hash-based sharding is that range queries require scatter-gather..."
- "Cross-shard analytics should be offloaded to a data warehouse via CDC pipeline."

**Red flags to avoid:**
- Sharding from day one without justification
- Not discussing shard key trade-offs
- Not knowing that cross-shard joins are impossible natively
- Not being able to name what makes a bad shard key

---

## Staff Engineer Interview Summary

A Staff Engineer is expected to additionally demonstrate:

**Architecture-level thinking:**
- "We should use 1000 logical shards mapped to 4 physical servers. This gives us flexibility to add physical capacity by moving logical shards, without changing routing logic."
- "We'll need consistent hashing — naive modulo sharding makes adding capacity too expensive operationally."
- "The Saga pattern for order placement means our inventory service must be idempotent — if the compensation fires and the original succeeded, we shouldn't refund twice."

**Cross-cutting concerns:**
- Understanding how sharding interacts with connection pooling (PgBouncer per shard)
- Understanding that per-shard monitoring requires new alerting infrastructure
- Knowing when NOT to shard (managed distributed DB is better, data fits on one node with proper indexes, write load isn't the bottleneck)

**Production war stories or awareness:**
- Aware of the celebrity/hot-key problem and can propose specific mitigations
- Can describe an online resharding process without downtime, step by step
- Knows the difference between CockroachDB (automatic, ACID) and Cassandra (manual schema design, eventual) and when to recommend each

**Trade-offs as first-class citizens:**
- "The trade-off of fan-out on write is 100M Redis writes for a celebrity tweet. That's an acceptable cost for 2B read requests."
- "We could use 2PC but the coordinator becomes a SPOF. Saga with idempotent compensations is operationally safer."

---

## 50 Advanced Sharding Interview Questions

### Fundamentals

**1. Why can't you just keep adding read replicas instead of sharding?**
Read replicas only scale reads. All writes still funnel to one primary. If write throughput or total data size is the bottleneck, read replicas provide no relief. Replication copies 100% of data to each replica — it doesn't partition the write load.

**2. What's the difference between partitioning and sharding?**
Partitioning is a general term — it can mean dividing a table's data within a single database instance (table partitioning). Sharding specifically refers to horizontal partitioning across multiple independent database instances.

**3. What does "scatter-gather" mean in the context of sharding?**
Scatter: a query that can't be satisfied by one shard is sent to all shards in parallel. Gather: results from all shards are collected by the application layer and merged/aggregated. Expensive in both latency and resource consumption.

**4. Why is the shard key the most critical architectural decision?**
It determines data distribution, query routing, and which queries are single-shard vs cross-shard. Unlike most architectural decisions, changing it in production requires a massive online migration that takes weeks and carries high data integrity risk.

**5. What is a hot shard and how do you detect it?**
A shard receiving disproportionately more traffic than others. Detection: per-shard monitoring of QPS, CPU%, disk I/O, and latency. Alert when any shard exceeds 150% of average across these metrics for sustained periods.

### Strategies

**6. Why is `hash % N` problematic when N changes?**
When N changes, `hash(x) % N₁ ≠ hash(x) % N₂` for most values of x. Approximately `(N-1)/N` of all keys must move to a different shard — about 80% of data when adding 1 shard to a 4-shard cluster.

**7. How does consistent hashing solve the resharding problem?**
By placing both servers and keys on a hash ring, adding a new server only takes ownership of keys that fall between it and its predecessor on the ring — approximately `1/N` of total keys, instead of `(N-1)/N`.

**8. Why does Cassandra use virtual nodes?**
With few physical nodes and direct placement on the ring, node positions can be clustered — creating uneven data distribution. Virtual nodes (256 per physical node by default) spread each server's ring ownership across many small segments, producing much more even distribution.

**9. When would you choose directory-based over hash-based sharding?**
When you need maximum flexibility: reassigning specific tenants to dedicated shards, handling whale customers, or migrating data between shards without global resharding. The directory allows any arbitrary key-to-shard mapping with a single lookup table update.

**10. What are the GDPR implications of geo-based sharding?**
GDPR requires EU resident personal data to remain within EU borders. Geo-sharding ensures EU users' data is stored and processed only in EU-region data centers. You must also prevent accidental cross-region data leakage during routing and handle the edge case of EU users accessing your service from outside EU.

### Shard Key Selection

**11. Can you shard by email? What problems does this create?**
You can, but: (1) email is mutable — if a user changes email, the record must move to a different shard; (2) many queries have user_id, not email, requiring an additional lookup; (3) distribution may be uneven (many users on `@gmail.com`). Use email as a shard key only if it's the sole identifier in your system and is truly immutable.

**12. Why is timestamp a bad shard key for write-heavy systems?**
Timestamp is monotonically increasing. All new records have the latest timestamp, so they all route to the shard handling the current time range. That shard receives 100% of writes while all other shards are cold. Classic write hotspot.

**13. How would you handle whale tenants in a multi-tenant SaaS?**
Identify tenants whose data volume or query traffic exceeds a threshold (e.g., 10× average). Give them dedicated shard(s) via directory-based routing. Standard tenants share shards and are hashed normally. The directory lookup routes each tenant_id to its specific shard, whether shared or dedicated.

**14. Explain the celebrity problem in social networks.**
A user with tens of millions of followers (e.g., @jbiebs on Instagram) has their shard queried every time any of those followers loads a feed. Even with perfect hash-based shard key distribution, this creates a single-shard hot key. Solution: aggressive caching of celebrity posts (Redis TTL), fan-out on write, dedicated read replicas for celebrity shards, and app-level hot key detection.

**15. Sharding Twitter by user_id vs tweet_id — which is better?**
Depends on access patterns. Shard by user_id: all tweets from one user are on one shard (good for "get my tweets" query). Shard by tweet_id: uniform distribution but timeline assembly requires scatter-gather across all author shards. Twitter's actual approach: store tweets by tweet_id, use a separate fan-out service to push tweets to follower timeline caches.

### Cross-Shard Operations

**16. How do you implement deep pagination with sharding?**
For page depth D with page size P: each shard must return D×P results. The application gathers N×D×P results, sorts, and returns rows from D×P to (D+1)×P. Cost grows linearly with page depth. Solutions: cursor-based pagination (continue from a specific tweet_id/timestamp), limit maximum page depth (Twitter shows at most ~3200 tweets in home timeline), or precompute results.

**17. How do you implement a global secondary index in a sharded database?**
Options: (1) Maintain a dedicated index shard that maps secondary key → primary shard (updated synchronously or asynchronously on every write); (2) Use a database that natively supports this (DynamoDB GSI, Cassandra materialized views); (3) Dual-write at application level (write primary data to shard + write to global index table on every mutation).

**18. How should analytics queries be handled in a sharded system?**
Never run analytics on the operational shards. Maintain a dedicated OLAP system (BigQuery, Redshift, Snowflake). Use CDC (Change Data Capture) tools like Debezium to capture changes from all shards, stream through Kafka, and load into the data warehouse. Analytics queries hit the warehouse, not the operational shards.

**19. Fan-out on write vs fan-out on read — explain the trade-off in the context of sharding.**
Fan-out on write: on each new post, push to all follower shards immediately. Timeline reads are O(1) — single shard lookup. Write cost: O(followers) × shard writes per post. Fan-out on read: assemble timeline at read time by querying all followed users' shards. Read cost: O(follows) × shard queries per load. Twitter uses a hybrid: fan-out on write for normal users, fan-out on read for accounts with >1M followers.

**20. What is split-brain and how does it affect sharded systems?**
Split-brain occurs during network partitions when isolated parts of a system believe they're the authoritative source. In a sharded system, if the routing layer believes a shard is unavailable and promotes a replica to primary, but the original primary is still alive (just network-isolated), you can have two primaries accepting conflicting writes. Prevention: majority quorum for primary promotion (Raft/Paxos), fencing tokens, strict lease-based primary election.

### Distributed Transactions

**21. Why is 2PC rarely used in production sharded systems?**
(1) Coordinator is a single point of failure — crash during PREPARE leaves shards holding locks indefinitely; (2) Locks held across network round trips create cascading latency under load; (3) Performance: two network round trips per transaction; (4) Doesn't scale — N shards × 2 phases = O(N) network messages. Most production systems prefer Saga or accept eventual consistency.

**22. Explain the Saga pattern with a concrete e-commerce example.**
Order placement saga: T1: Reserve inventory (compensate: release inventory). T2: Charge credit card (compensate: issue refund). T3: Create order record (compensate: delete order). T4: Send confirmation email (compensate: send cancellation email). Each step executes sequentially; if any step fails, compensations run in reverse order. No global locks; eventual consistency; compensation logic must be idempotent (refunding twice must not charge twice negatively).

**23. What is eventual consistency and when is it acceptable?**
Eventual consistency means that given no new updates, all nodes will converge to the same value over time. Acceptable for: social media feeds, product catalog visibility, user preference changes, like counts. Not acceptable for: financial ledger balances (no money creation/destruction), inventory count during checkout (no overselling), authentication tokens (no duplicate active sessions).

**24. How does DynamoDB handle transactions across partitions?**
DynamoDB Transactions use an internal two-phase protocol coordinated by a transaction coordinator service. Supports up to 25 items across tables/partitions per transaction. Uses conditional expressions (version checks) under the hood — if any item has changed since the transaction began, the entire transaction fails and can be retried. Incurs 2× the write cost (WCU) vs non-transactional writes.

**25. Why is optimistic locking preferred in distributed systems?**
Optimistic: read data with a version number, attempt write only if version hasn't changed (if changed, retry). No locks held during network I/O. Pessimistic: lock record before reading, hold lock through write. In distributed systems, holding locks across network calls means one slow node or network delay stalls all other writers touching that record. Optimistic locking's retry overhead is typically lower than pessimistic locking's contention overhead at web scale.

### Resharding & Operations

**26. Walk through an online resharding process with zero downtime.**
(1) Provision new shards. (2) Enable dual-write: all new writes go to both old and new configuration simultaneously. (3) Background migration job copies historical data to new shards, using CDC to stay current. (4) Verify: run checksums; compare results from old vs new shards in shadow mode. (5) Gradual read cutover: route 1%→10%→50%→100% of reads to new shards, monitoring error rates and latency. (6) Remove dual-write once new shards are fully authoritative. (7) Decommission old shards after a safety window.

**27. How does Vitess handle online schema changes in sharded MySQL?**
Vitess integrates with gh-ost (GitHub's Online Schema Change) for non-blocking DDL. gh-ost creates a shadow table with the new schema, applies existing data, and continuously replays changes from the MySQL binary log onto the shadow table while production traffic continues on the original. Once caught up, it performs an atomic table rename (single metadata lock, <1 second). Vitess orchestrates this process across all shards in parallel.

**28. What happens when a shard runs out of disk space?**
Operational emergency — must be addressed before disk is full (DB crashes). Immediate mitigation: (1) Add storage if cloud (EBS resize, AWS takes minutes). (2) Identify large tables, run archival job to move old data to cold storage. (3) Split the shard — add a virtual node adjacent to the full shard in consistent hashing, taking half its key range. Prevention: alert at 70% disk utilization, 80% critical.

**29. How do you handle a complete shard failure?**
Each shard must have at least one replica running. On shard failure: (1) Routing layer detects unresponsive shard (health check timeout). (2) Promote replica to primary (automated with orchestration tools like Orchestrator for MySQL, Patroni for PostgreSQL). (3) Update routing configuration to point to new primary. (4) Alert on-call; provision replacement replica. If no replica exists: declare incident, restore from backup, accept data loss up to last backup.

**30. What is consistent routing and why does it matter?**
Consistent routing ensures the same key always routes to the same shard for a given routing configuration. This is critical for: cache coherence (cache invalidation assumes a canonical shard), debug reproducibility (can trace which shard a request hit), and correctness (the same key must not simultaneously exist on two different shards with different values during normal operation).

### System Design Application

**31. Design the sharding strategy for a Slack-like product.**
Primary shard key: `workspace_id` (tenant isolation, all queries are workspace-scoped). Large workspaces (e.g., Salesforce with 70K employees) get dedicated shards via directory-based routing. Message storage: Cassandra sharded by workspace+channel (write throughput for message fanout). Metadata (channels, users, permissions): sharded PostgreSQL by workspace_id. Search: Elasticsearch cluster with workspace filter. Analytics: data warehouse via CDC.

**32. How would you shard YouTube's video metadata?**
Primary shard key: `video_id` (hash-based). Most queries: "get metadata for video X" — single key lookup. Watch count: extremely high write frequency; use sharded Redis counters with periodic flush to DB rather than DB writes per view. User's uploaded videos: secondary table sharded by `user_id`. Video recommendations: separate ML-driven service with precomputed results in Redis, keyed by user_id.

**33. How does Airbnb shard listing data?**
Core listing data: sharded by `listing_id` (hash-based). Geographic searches ("find listings in Paris"): separate geospatial search index (Elasticsearch with geo_point fields) — NOT the primary sharded DB. Bookings: sharded by `booking_id` with secondary indexing strategy for listing_id and guest_id lookups. Availability calendar: high-write append-only table, sharded by `listing_id + date_range`, with caching layer.

**34. Design database sharding for a ride-sharing app.**
Active trips: Redis (in-memory, sub-ms, trips last minutes). Completed trips: sharded PostgreSQL by `trip_id` (write-once records, uniform hash distribution). Driver/rider profiles: sharded by `user_id`. Location/proximity queries: dedicated geospatial service (PostGIS, S2 cells, or custom geohash-based store) — not the sharded relational DB. Analytics: BigQuery via CDC pipeline. Payments: separate service, possibly using Stripe with their own sharding.

**35. A shard is receiving 3× the traffic of others. What do you do?**
Immediate (minutes): Add read replicas to the hot shard, configure routing to use them. Cache the hot data (Redis) to absorb read pressure — identify the specific keys causing heat. Short-term (hours): Identify the root cause — is it a specific user, product, or time pattern? If a celebrity/viral key: implement application-level hot-key caching. If write-heavy: consider shard splitting. Medium-term (days): If structural (bad shard key), plan a shard key migration. Add per-shard alerting to catch this earlier next time.

### Advanced Architecture

**36. What is CQRS and how does it relate to sharding?**
Command Query Responsibility Segregation separates write models (commands) from read models (queries). In sharded systems: writes go to a sharded OLTP database optimized for write throughput. For complex read queries that would require expensive cross-shard scatter-gather, maintain separate read models (precomputed, denormalized, possibly non-sharded) built from events published by the write side. Read and write models can evolve independently.

**37. What are chunk migrations in MongoDB?**
MongoDB divides shard data into chunks (~64MB each). The balancer detects uneven chunk distribution across shards and migrates chunks to equalize. During migration: (1) Destination shard receives the chunk data from source. (2) Source captures and streams writes that occur during migration (via oplog) to the destination. (3) Config server atomically updates the chunk's shard assignment. (4) Clients redirect new requests for that chunk range to the destination shard. Can impact performance — tune migration windows to off-peak hours.

**38. How does CockroachDB achieve ACID transactions across shards without traditional 2PC?**
CockroachDB uses Hybrid Logical Clocks (HLC) for causal ordering across nodes and a distributed transaction protocol with "parallel commits." The transaction coordinator writes an atomic commit record to a single range, then in parallel writes intents (uncommitted values) to all other involved ranges. If all intents succeed, the commit record becomes the linearization point. This avoids the two-round-trip 2PC overhead for the common case. Each range uses Raft for internal consensus.

**39. What is the N+1 query problem in sharding and how do you solve it?**
N+1 in sharding: fetch a list of N entity IDs, then make N individual queries to potentially N different shards — a request that should be 2 queries (1 list + 1 batch) becomes N+1. Solution: (1) DataLoader-style batching — collect all entity IDs needed in a request cycle, group by shard, query each shard once with `WHERE id IN (...)` — reduces N queries to at most S queries (S = number of shards). (2) Denormalize data so the list query returns all needed attributes without secondary lookups.

**40. How do you implement distributed rate limiting in a sharded environment?**
Per-user rate limits must be consistent across shards (user can't exceed limit by hitting different shards). Options: (1) Centralized rate limiter in Redis — `INCR user:1001:req_count` with TTL. Fast, accurate, Redis becomes a bottleneck if not clustered. (2) Redis Cluster with consistent hashing — rate limit counter for user X always routes to same Redis slot. (3) Approximate distributed limiting — each shard tracks partial counts, periodically syncs to central store; user can briefly exceed limit during sync lag.

### Deeper Technical

**41. What is connection pooling and why is it critical in sharded systems?**
Without pooling: each query from each application instance opens a new TCP connection to the database. PostgreSQL spawns a process per connection (5–10MB RAM each). With N shards × M application instances, you could have N×M×threads connections → memory exhaustion. Solution: PgBouncer or ProxySQL in transaction pooling mode sits in front of each shard. Each application instance connects to the pooler; the pooler maintains a small pool of real DB connections shared across all clients. Reduces total connection count by 10–100×.

**42. How does sharding interact with database indexes?**
Each shard has its own independent indexes covering only its subset of data. A secondary index on `email` in Shard-1 only covers users on Shard-1. There is no global index across all shards unless you explicitly build one (a global secondary index service or a database that natively supports it like DynamoDB GSI). This is why careful data co-location and denormalization are so important — you need to avoid queries that require joining data from indexes on different shards.

**43. What is data skew vs a hot shard? How do they differ?**
Data skew: uneven data volume across shards (Shard-1 has 2TB, Shard-2 has 500GB). A data skew doesn't necessarily create a hot shard — if the smaller shard contains more frequently accessed data, it could be the hot one. Hot shard: uneven query traffic, regardless of data volume. A shard with 500GB can be hotter than a 2TB shard. Treat them differently: data skew → reshard to rebalance data. Hot shard → caching, read replicas, query routing changes.

**44. How would you handle a migration from sharded PostgreSQL to Cassandra?**
(1) Model data for Cassandra access patterns (Cassandra requires denormalization — one table per query pattern). (2) Enable dual-write: application writes to both PostgreSQL shards and Cassandra simultaneously. (3) Background migration job copies historical data to Cassandra, using CDC from PostgreSQL to stay current. (4) Shadow reads: read from Cassandra, compare with PostgreSQL, log discrepancies without serving Cassandra results to users. (5) Gradually shift read traffic to Cassandra. (6) Once confidence established, remove PostgreSQL writes. High-risk — maintain clear rollback capability throughout.

**45. What is the thundering herd problem in sharding?**
Scenario: a cache entry for hot data expires. Simultaneously, hundreds of requests all see a cache miss and query the underlying shard directly. The shard receives a sudden spike in requests (the "herd") that it wasn't designed to handle. Prevention: (1) Mutex/lock-based cache regeneration — only one request regenerates the cache, others wait; (2) Staggered TTL — add random jitter to TTL to prevent mass simultaneous expiration; (3) Background refresh — refresh cache before it expires based on hit rate; (4) Circuit breaker — if shard is overloaded, fail fast rather than pile on additional requests.

### Interview Judgment Questions

**46. When should you NOT shard?**
When: your total data fits on one server with room to grow (e.g., <1TB); write throughput is below what a well-tuned primary can handle (< ~20K writes/sec for simple operations); read scaling is the actual bottleneck (solve with read replicas); queries require complex joins that would be impossible cross-shard; a managed distributed database (DynamoDB, Spanner) solves the problem without you building sharding infrastructure; the engineering team doesn't have the operational expertise to manage sharded systems safely. Sharding trades simplicity for scale — don't pay that cost until you need it.

**47. How would you convince a team that sharding is necessary right now?**
Present data: "Primary CPU has been above 80% for the last 3 weeks, growing at 5% per week. Write latency p99 has increased from 20ms to 85ms in the last month. Replication lag spikes to 3 seconds during peak. Disk I/O is at 85% of capacity. At our current growth rate, we exceed comfortable single-node limits in 10–12 weeks. Sharding takes 6–8 weeks to implement safely. We need to start now. Here's the proposed shard key, migration plan, and rollback strategy."

**48. What's the difference between application-level and middleware sharding?**
Application-level: sharding logic lives in application code. Each service connects to the correct shard based on the shard key. Simple and explicit; every service must implement routing logic separately; no central abstraction. Middleware: a proxy (Vitess, ProxySQL, Citus, PgBouncer with routing) handles shard routing transparently. Applications use standard database connections; the proxy routes to correct shards. Higher operational complexity (one more system to manage) but simplifies application code, enables cross-service consistency, and allows shard-transparent features like online schema changes.

**49. How do companies like Stripe handle strong consistency in a sharded system?**
Stripe's approach (inferred from public writing): (1) Shard by `customer_id` — all operations for a customer (charges, subscriptions, invoices) are on one shard, enabling full ACID within a customer's data; (2) Cross-customer operations are rare and handled with idempotency keys + Saga; (3) Events and webhooks use an idempotency layer — if a webhook delivery fails, retry with the same idempotency key prevents duplicate processing; (4) Each shard is a fully replicated PostgreSQL cluster for HA; (5) Financial operations produce an immutable audit event log regardless of DB state for reconciliation.

**50. A Staff Engineer asks: "We're considering sharding our user database. Walk me through every trade-off." How do you respond?**
"I'd approach this in order. First: Do we actually need it? What's our write throughput, data size, and CPU on primary? Have we maxed out vertical scaling, caching, and read replicas? If sharding is justified: Shard key — I'd propose user_id because it has billions of unique values, it's immutable, and most queries are user-scoped ('get my profile', 'get my feed'). I'd use hash-based with consistent hashing for uniform distribution and manageable resharding. Trade-offs to address: cross-user queries like 'trending topics' require scatter-gather — we'd precompute trending in a separate service and cache results; analytics must move to a data warehouse; celebrities and viral content create hot keys even with perfect hashing — require Redis caching and per-user hit-count detection. Distributed transactions: we'd design data co-location so most transactions are single-shard; cross-shard transactions use Saga. Resharding plan: start with 64 logical shards on 4 physical servers; adding physical capacity means moving logical shards, not raw data. Operational requirements: per-shard monitoring from day one, PgBouncer in front of each shard, online migration runbook ready before we need it. Finally: should we build this or use CockroachDB/PlanetScale? Given our team's operational maturity, I'd evaluate managed options before committing to custom sharding infrastructure."

---

## What to Learn After Sharding

```
RECOMMENDED LEARNING ROADMAP POST-SHARDING

IMMEDIATE NEXT STEPS:
├── Distributed Systems Theory
│   ├── CAP Theorem (deep — Brewer's paper + follow-up nuances)
│   ├── PACELC Model (extends CAP)
│   ├── Raft consensus (https://raft.github.io)
│   ├── Vector clocks and causal consistency
│   └── CRDT (Conflict-free Replicated Data Types)
│
├── Specific Distributed DBs (internals)
│   ├── Cassandra: SSTable, Memtable, Compaction, Bloom filters
│   ├── DynamoDB: LSM tree, B-tree hybrid, partition management
│   ├── Spanner: TrueTime API, globally consistent timestamps
│   └── CockroachDB: Raft groups, parallel commits, geo-partitioning

MEDIUM TERM:
├── Stream Processing
│   ├── Kafka architecture (partitions ARE shards for streams)
│   ├── Change Data Capture (Debezium, Maxwell)
│   ├── Apache Flink for stream processing
│   └── Event sourcing and CQRS patterns
│
├── Data Warehousing
│   ├── Columnar storage (Parquet, ORC, Arrow)
│   ├── BigQuery / Redshift / Snowflake internals
│   ├── OLAP cube design
│   └── Data lakehouse architecture (Delta Lake, Iceberg)
│
├── Advanced Caching
│   ├── Cache-aside, write-through, write-behind patterns
│   ├── Cache invalidation strategies in distributed systems
│   ├── Redis cluster mode and consistent hashing
│   └── CDN caching and edge computing

LONG TERM:
├── Observability for Distributed Systems
│   ├── Distributed tracing (Jaeger, Zipkin, OpenTelemetry)
│   ├── Per-shard metrics and SLO design
│   └── Chaos engineering (Chaos Monkey, controlled failure injection)
│
└── System Design Practice (sharding-heavy problems)
    ├── Design Google Maps
    ├── Design a Global Payment System  
    ├── Design Facebook Messenger at scale
    ├── Design Netflix's Video Delivery Pipeline
    └── Design a Global Ad Targeting System

KEY PAPERS TO READ:
  • Amazon Dynamo (2007) — foundational distributed KV store
  • Google Bigtable (2006) — range-based sharding at Google scale
  • Google Spanner (2012) — globally consistent distributed SQL
  • Facebook TAO (2013) — social graph storage at 1B+ user scale
  • Cassandra paper (2009) — consistent hashing + eventual consistency
  • Raft (2014) — understandable consensus algorithm
```

---

*This document was generated as a comprehensive reference. Revisit each section multiple times — sharding is understood in layers, and concepts deepen with repeated exposure and practical application.*