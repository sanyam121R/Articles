---
title: "Database Pages: The Fundamental Unit of Storage"
date: "2025-03-12"
description: "Understanding how databases store data in pages — the fundamental building block that shapes everything from indexes to sharding."
tags: ["database", "storage", "pages", "buffer-pool", "sharding", "fundamentals"]
link: "https://medium.com/gitconnected/database-pages-the-fundamental-unit-behind-every-query-380dd061b75e"
---

# Database Pages: The Fundamental Unit of Storage

I was learning about **database sharding** from first principles when I stumbled upon one of the most fundamental concepts in database internals—the **Page**.

A **page** is the smallest unit of storage management that a database engine reads from or writes to disk.

Our SSDs and HDDs store files as raw bytes, but database engines don't read one byte at a time—that would be incredibly inefficient. Instead, they divide their data files into fixed-size blocks called **pages**.

- **MySQL (InnoDB)** uses **16 KB** pages.
- **PostgreSQL** uses **8 KB** pages by default.

Imagine going to a library looking for a single quote. The librarian doesn't let you borrow just that sentence—you borrow the **entire page** containing it.

Even if you only need one sentence, you still receive the whole page.

That's exactly how databases interact with storage.

This seemingly simple design decision—**reading and writing pages instead of individual bytes**—becomes the foundation for many advanced database concepts you'll encounter later, including indexes, caching, transactions, replication, and ultimately, sharding.

---

## What's Inside a Database Page?

A page isn't just a collection of rows. It contains several components.

- **Page Header**
  - Stores metadata such as page ID, checksum, free-space pointer, and information about which table the page belongs to.

- **Row Slots**
  - Actual records (for example, `Alice (id=1)` or `Bob (id=2)`) stored as binary-encoded tuples.

- **Free Space**
  - Unused bytes reserved for future inserts and updates.

---

## Why Does a Database Read Entire Pages?

Although operating systems allow reading individual bytes, doing so would be catastrophically inefficient.

There are several reasons:

- SSDs are optimized for **block-level access** (typically around **4 KB** minimum read units).
- Operating systems manage memory using **virtual memory pages** (commonly 4 KB).
- The expensive part of disk access is **I/O latency**, not transferring a few extra kilobytes.

Reading **16 KB** costs only slightly more than reading **1 byte** because most of the time is spent waiting for the I/O operation itself.

So the database effectively says:

> "If I'm paying the cost of accessing the disk anyway, I might as well fetch as much useful data as possible."

Hence, pages.

---

## The Buffer Pool: Pages in RAM

Databases never operate directly on disk for every query.

The workflow looks like this:

```text
Query arrives
      ↓
Database checks the Buffer Pool (RAM)
      ↓
Is Page 5 already cached?

├── YES → Read Carol's row directly from RAM
│          (Fast: nanoseconds)
│
└── NO  → Load Page 5 from SSD into RAM
           Then read the row
           (Slow: microseconds to milliseconds)
```

![Page 5 - 16 KB on disk](@/assets/blogs/Page%205%20-%2016%20KB%20on%20disk)

The **buffer pool** stores recently or frequently accessed pages in memory.

Indexes don't eliminate disk reads—they reduce **how many pages** the database needs to load.

---

## Buffer Pool Terminology Across Databases

| Database | Cache Name |
|-----------|------------|
| MySQL | InnoDB Buffer Pool |
| PostgreSQL | Shared Buffers |
| MongoDB | WiredTiger Cache |
| Cassandra | OS Page Cache |
| Redis | Entire Dataset in Memory |

---

## Why Pages Matter for Scaling

The page abstraction is also the reason scaling databases becomes difficult.

| Problem | Root Cause |
|---------|------------|
| Slow queries without indexes | Too many pages must be loaded |
| RAM is limited | Only a subset of pages can stay in memory |
| Writes become slower | Updating a row also updates index pages, causing multiple I/Os |
| Large tables become huge files | More pages eventually exceed the storage capacity of a single machine |

The final point is exactly **why sharding exists**.

Instead of storing every row on one machine, a table is partitioned across multiple machines using a **shard key** (for example, `user_id`).

Each shard:

- stores only part of the table,
- maintains its own pages,
- has its own indexes,
- and serves only the requests routed to it.

A routing layer determines which shard should receive the query, while the B-tree index on that shard locates the required page.

---

## A Simple Mental Model

> A database is a system that efficiently manages:
>
> - which pages live on disk,
> - which pages stay in RAM,
> - and which page contains the row you're looking for.

Everything else—indexes, caching, transactions, replication, and sharding—is essentially an optimization around one question:

> **How can we minimize expensive disk I/O while finding the right data?**

---

## The Redis Exception

Redis is one of the major exceptions to this model.

Unlike traditional databases, Redis keeps its entire dataset directly in **RAM** rather than loading pages from disk.

Disk is used only for persistence:

- **RDB snapshots**
- **AOF (Append Only File)**

These mechanisms are used for recovery, **not primary data access**.

Since Redis serves data directly from memory, it avoids the disk I/O involved in loading pages, making it dramatically faster for many read-heavy workloads.

---

## Final Thoughts

The concept of **pages** is nearly universal across disk-based databases.

Whether you're using MySQL, PostgreSQL, SQL Server, Oracle, or many NoSQL systems, they all solve the same fundamental problem:

> **Disk is slow. RAM is fast. Keep the hot pages in memory.**

Once you understand pages, concepts like B-trees, indexes, buffer pools, transactions, WAL, replication, and even sharding become much easier to reason about.

Because underneath it all, databases are simply trying to answer one question as efficiently as possible:

> **Which page contains the data I need, and how can I get it with the fewest possible disk reads?**

---

Thanks for reading!

See you in the next blog.