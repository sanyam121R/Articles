---
title: "Rate Limiting with Redis: From First Principles"
slug: "rate-limiting-with-redis"
date: "2025-06-18"
description: "A deep dive into rate limiting concepts from first principles — covering Redis fundamentals, atomicity, race conditions, and practical implementations with code examples."
tags: ["redis", "rate-limiting", "backend", "distributed-systems", "algorithms"]
link: ""
---

This post breaks down every concept behind rate limiting from zero — Redis, its commands, atomicity, race conditions, all of it. Read it top to bottom to build the mental model, then use the code examples as a starting point.

---

## What Redis actually is

Most databases (Postgres, MySQL, MongoDB) store data on disk. Even SSDs take ~100 microseconds per read. Redis stores everything in RAM — your computer's working memory — which takes ~250 nanoseconds. That's roughly 400× faster.

The trade-off: RAM is expensive and clears on restart. Redis accepts this because for things like counters and caches, speed matters more than permanence. For rate limiting, you usually don't need the data to survive a server restart anyway.

**Mental model:** Your hard drive is a library warehouse — huge, holds everything, slow to retrieve. RAM is the desk in front of you — small, instantly accessible, gone when you leave.

---



## Redis's secret weapon: single-threaded

Redis processes exactly one command at a time, in order. While command A executes, commands B and C wait in a queue. This sounds like a limitation, but it's actually the foundation of everything that makes Redis safe for distributed systems.

**It means: two commands can never truly run simultaneously inside Redis.** This is what makes INCR and Lua scripts "atomic" — the single thread is the lock, for free, no extra coordination needed.

---



## Redis data types

Redis isn't just a simple "key → value" store. It supports different value types, each solving a different problem.

**Strings** — the default. A key maps to a text or number value. For rate limiting, you use a string as a plain counter.

**Sorted Sets** — a collection where every member has a numeric score, and Redis automatically keeps members sorted by score. For rate limiting, you use timestamps as scores — then you can instantly ask "how many members have a score in the last 60 seconds?"

**Hashes** — like a JavaScript object. One key holds multiple field-value pairs. For token bucket rate limiting, you store `tokens` and `last_refill` together in one hash.

---



## Every Redis command explained from scratch

`GET key` — reads the value at a key. Returns `null` if the key doesn't exist or has expired.

`SET key value [EX seconds]` — stores a value. The optional `EX` parameter sets an expiry — after that many seconds, Redis automatically deletes the key. This is how counters reset without a cleanup job.

`INCR key` — here's the important one. It atomically: reads the current value → adds 1 → stores it back → returns the new value. All four steps happen as one indivisible Redis operation. Nothing can interleave between them. If the key doesn't exist yet, Redis treats it as 0 and returns 1.

`EXPIRE key seconds` — sets a TTL (Time To Live) on an existing key. Used when you created the key with INCR (which has no EX option) and need to add expiry after the fact.

`TTL key` — returns seconds remaining until expiry. Returns `-1` if no expiry is set, `-2` if the key doesn't exist.

`ZADD key score member` — adds a member to a sorted set with a given score. Redis keeps the set sorted by score automatically. For rate limiting: `score` = timestamp in milliseconds, `member` = unique request ID.

`ZREMRANGEBYSCORE key min max` — removes all members whose score falls between `min` and `max`. Returns how many were removed. For sliding window: call this first to purge timestamps older than your window.

`ZCARD key` — returns the total count of members in a sorted set. After ZREMRANGEBYSCORE cleans old timestamps, ZCARD gives you the exact count in your sliding window.

`HMSET key field value [field value ...]` — sets multiple fields in a hash in one command.

`HMGET key field [field ...]` — reads multiple fields from a hash in one command.

`EVAL script numkeys key [key...] arg [arg...]` — runs a Lua script inside Redis as a single atomic operation. `KEYS[1]`, `KEYS[2]` etc. are your key arguments; `ARGV[1]`, `ARGV[2]` etc. are your value arguments. The entire script blocks all other commands while it runs.

---



## Atomicity — what it actually means

Atomic means: the operation either happens completely or not at all. No other operation can observe a "half-done" state.

Classic example — a bank transfer is two steps: deduct from A, add to B. If the system crashes between steps, $100 vanishes. An atomic transaction ensures both happen or neither does.

For rate limiting, you need: check the count AND increment AND set TTL to be indivisible. Any gap between them is exploitable.

## Race condition — the actual bug

When two operations both read the same shared state, make a decision, and both write back — the writes conflict:

```
Server A: reads count = 99  ←─ both read the same value
Server B: reads count = 99  ←─ at the same moment
Server A: decides "under limit" → writes 100
Server B: decides "under limit" → writes 100  ← overwrites A, count stays 100
Result: 2 requests passed but only 1 was recorded
```

At 50 servers this leaks hundreds of extra requests per second, silently.

## Pipeline — batching for speed

Each Redis command is normally a separate network round trip (send + wait). With 4 commands at 2ms each, that's 8ms of latency. A pipeline bundles all commands into one network request — one send, one batch reply, total 2ms. Commands still execute in order, but pipeline is about **network efficiency, not atomicity**. Other clients can still interleave between your batched commands.

## Lua scripts — atomic multi-step operations

A Lua script runs entirely inside Redis as one atomic operation. Since Redis is single-threaded, no other command from any client can run while your script executes. This is how you chain GET + check + INCR + EXPIRE into one indivisible unit — the Stripe approach. `KEYS[]` hold your key names, `ARGV[]` hold your value arguments (limits, TTLs, etc.).