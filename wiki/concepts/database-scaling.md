---
title: "Database Scaling: Handling Massive Data and High Read-Write Operations"
category: concept
tags: [database, scaling, sharding, read-replica, indexing, caching, partitioning, cqrs, denormalization]
sources: []
updated: 2026-07-11
---

# Database Scaling

A layered set of techniques for handling massive datasets and high read/write throughput. Apply in order — each layer buys significant headroom before you need the next one.

---

## The scaling ladder (apply in this order)

```
Traffic grows →

1. Vertical Scaling        Easiest. Just use a bigger machine.
        │
        ▼ (hit CPU/memory ceiling)
2. Caching                 Serve reads from memory. Massive impact.
        │
        ▼ (still hitting DB)
3. Connection Pooling      Stop exhausting DB connections.
        │
        ▼ (slow queries)
4. Indexing                Make queries faster, not DB bigger.
        │
        ▼ (write load growing)
5. Read Replicas           Scale reads horizontally.
        │
        ▼ (single table too large)
6. Partitioning            Split one table into chunks, same server.
        │
        ▼ (need separate read model)
7. Denormalization / CQRS  Pre-compute reads, separate write model.
        │
        ▼ (outgrown one server entirely)
8. Sharding                Split data across multiple servers.
        │
        ▼ (write spikes)
9. Write Batching          Buffer and flush writes in bulk.
```

Sharding is powerful but operationally painful. Exhaust the simpler options first.

---

## 1. Vertical Scaling

Add more CPU, RAM, or faster storage (NVMe SSD) to the existing server. Simplest option — no code changes, no architecture changes.

```
Before:              After:
┌──────────┐         ┌──────────────┐
│  DB      │   →     │  DB          │
│  8 core  │         │  64 core     │
│  32GB    │         │  512GB RAM   │
│  HDD     │         │  NVMe SSD    │
└──────────┘         └──────────────┘

✓ Zero complexity
✗ Hard ceiling — you can't scale forever
✗ Expensive at the high end
✗ Single point of failure remains
```

---

## 2. Caching (Redis / Memcached)

Put an in-memory cache in front of the database. Most read-heavy systems see 80-95% of reads served from cache, dramatically reducing DB load.

```
Without cache:                     With cache:
                                              Redis
Client ──► DB (every read)         Client ──► Cache ──► DB (cache miss only)
                                              ▲
                                              └── 95% of reads served here
```

### Cache strategies

**Cache-aside (lazy loading)** — most common:
```
Read request:
  1. Check cache
  2. Cache HIT  → return immediately ✓
  3. Cache MISS → query DB → store in cache → return

Write request:
  1. Write to DB
  2. Invalidate cache entry (not update — avoids stale data)
```

**Write-through** — write to cache and DB simultaneously:
```
Write: Client ──► Cache ──► DB (both updated synchronously)
Read:  Client ──► Cache      (always warm, no misses)

✓ Cache always consistent with DB
✗ Write latency higher (two writes)
```

**Write-behind (write-back)** — write to cache only, flush to DB async:
```
Write: Client ──► Cache ──► (async) ──► DB
                    └── returns immediately

✓ Very fast writes
✗ Risk of data loss if cache crashes before flush
```

### What to cache
```
✓ User sessions / JWT tokens
✓ User profiles (read often, change rarely)
✓ Product catalog, exchange rates
✓ Aggregated counts (likes, views)
✗ Highly personalized data (cache pollution)
✗ Data that must be real-time accurate (account balance during payment)
```

---

## 3. Connection Pooling

Each database connection is expensive (memory, TCP handshake). At scale, you exhaust connections before the DB is even under load.

```
Without pooling:
  1000 app servers × 10 threads = 10,000 DB connections
  PostgreSQL default max: 100 connections → ✗ crash

With pooling (PgBouncer / RDS Proxy):
  1000 app servers → PgBouncer → 100 DB connections (multiplexed)
                        ▲
                        └── queues excess requests, reuses connections
```

```
App Servers         Connection Pool          Database
┌──────────┐       ┌──────────────┐        ┌──────────┐
│ Server 1 │──────►│              │        │          │
│ Server 2 │──────►│  PgBouncer   │───────►│ Postgres │
│ Server 3 │──────►│  (pool of    │        │ (100 max │
│   ...    │──────►│   100 conns) │        │  conns)  │
│ Server N │──────►│              │        │          │
└──────────┘       └──────────────┘        └──────────┘
```

---

## 4. Indexing

An index is a pre-sorted data structure that lets the DB find rows without scanning the entire table.

```
Without index — full table scan:
SELECT * FROM transactions WHERE user_id = 42;

  DB scans every row: [row1][row2][row3]...[row 50M]  ← O(n)
  50 million rows → slow

With index on user_id:
  DB jumps directly to user 42's rows via B-tree     ← O(log n)
  Milliseconds instead of seconds
```

### Index trade-offs

```
                  Reads          Writes
No index           Slow           Fast (no maintenance)
With index         Fast           Slower (index must be updated)

Rule: index columns you filter or sort on frequently.
Don't over-index — every index slows down INSERT/UPDATE/DELETE.
```

### Types of indexes

```
B-Tree index     → equality and range queries (most common)
                   WHERE user_id = 42
                   WHERE created_at BETWEEN x AND y

Hash index       → equality only, faster than B-tree for exact match
                   WHERE session_token = 'abc123'

Composite index  → multiple columns, order matters
                   INDEX(user_id, created_at)
                   ✓ WHERE user_id = 42 AND created_at > x
                   ✗ WHERE created_at > x (user_id must be first)

Partial index    → index a subset of rows
                   INDEX WHERE status = 'pending'
                   (smaller index, faster for that specific query)
```

---

## 5. Read Replicas

Add read-only copies of the primary database. Route all reads to replicas; all writes to primary.

```
        Writes only            Reads only
           ▼                     ▼
    ┌─────────────┐       ┌─────────────┐
    │   Primary   │──────►│  Replica 1  │
    │  (read +    │──────►│  (read only)│
    │   write)    │──────►│  Replica 2  │
    └─────────────┘       │  (read only)│
                          │  Replica 3  │
                          │  (read only)│
                          └─────────────┘
                          (replicated async from primary)

✓ Scale reads horizontally — add more replicas as needed
✓ Replicas can serve different regions (geo-local reads)
✗ Replication lag — replicas may be slightly behind primary
✗ Writes still bottlenecked on one primary
```

**Replication lag problem:**
```
User updates profile → writes to primary
User immediately reads profile → reads from replica
Replica not yet synced → user sees old data ← stale read

Solution: route reads that immediately follow a write back to primary,
or use "read your own writes" consistency.
```

---

## 6. Partitioning

Split a large table into smaller chunks within the **same database server**. Unlike sharding, there's no distributed system — the DB engine handles it transparently.

```
transactions table (500M rows)
         │
         ▼  partition by month
┌─────────────────────────────────────────┐
│  transactions_2026_01  (40M rows)       │
│  transactions_2026_02  (42M rows)       │
│  transactions_2026_03  (38M rows)       │
│  ...                                    │
└─────────────────────────────────────────┘

Query: WHERE created_at > '2026-06-01'
→ DB only scans transactions_2026_06 and _07
→ ignores all other partitions (partition pruning)
```

### Partition strategies

```
Range partitioning   → by date, ID range
                       Good for time-series data, archiving old partitions

Hash partitioning    → by hash of user_id
                       Even distribution, good for random access

List partitioning    → by region or category
                       WHERE region = 'US' only scans US partition
```

---

## 7. Denormalization and CQRS

### Denormalization

Deliberately duplicate data to avoid expensive joins at read time.

```
Normalized (write-optimized):          Denormalized (read-optimized):
┌──────────────┐  ┌──────────┐        ┌─────────────────────────────────┐
│ orders       │  │ users    │        │ orders_view                     │
│ id           │  │ id       │   →    │ order_id                        │
│ user_id ─────┼─►│ name     │        │ user_name  (copied from users)  │
│ product_id   │  │ email    │        │ product_name (copied from prods) │
└──────────────┘  └──────────┘        │ total                           │
                                      └─────────────────────────────────┘
Reads require JOIN                    Reads are a single table scan
Writes are clean                      Writes must update multiple places
```

### CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries) entirely.

```
                    ┌─────────────────┐
Write path:         │   Write DB      │
  Command ─────────►│  (normalized,   │
  (place order)     │   ACID)         │
                    └────────┬────────┘
                             │ event: OrderPlaced
                             ▼
                    ┌─────────────────┐
                    │  Event Stream   │
                    │  (Kafka)        │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
Read    │ Orders   │  │ Analytics│  │ Search   │
path:   │ View DB  │  │ DB       │  │ Index    │
Query ─►│(fast for │  │(optimized│  │(Elastic- │
        │ UI)      │  │ for agg) │  │ search)  │
        └──────────┘  └──────────┘  └──────────┘

Each read model is pre-computed and optimized for its specific query.
```

---

## 8. Sharding (Horizontal Partitioning)

Split data across **multiple independent database servers**. Each server (shard) owns a subset of the data.

```
Without sharding:                 With sharding (by user_id):
┌──────────────────┐              ┌──────────────┐
│  Single DB       │              │   Shard 1    │  user_id 0-33M
│  All 100M users  │    →         │   33M users  │
└──────────────────┘              ├──────────────┤
                                  │   Shard 2    │  user_id 33M-66M
                                  │   33M users  │
                                  ├──────────────┤
                                  │   Shard 3    │  user_id 66M-100M
                                  │   33M users  │
                                  └──────────────┘
```

### Shard routing

```
Request: GET /users/42
         │
         ▼
  Shard Router (hash(user_id) % num_shards)
         │
         ▼
  hash(42) % 3 = 0  →  Shard 1
```

### Shard key selection — the most critical decision

```
Good shard key:           Bad shard key:
  user_id                   created_at (hotspot — all writes go to today's shard)
  tenant_id                 status (hotspot — most rows are 'active')
  order_id (hashed)         country (uneven — US shard 10× bigger than others)

A bad shard key creates hotspots:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │
│ 80% load │  │ 10% load │  │ 10% load │  ← Shard 1 is the hotspot
└──────────┘  └──────────┘  └──────────┘
```

### Sharding problems

```
Cross-shard queries:
  SELECT * FROM orders WHERE user_id IN (42, 1001, 5M)
  → must query Shard 1 + Shard 2 + Shard 3 and merge results
  → slow and complex

Cross-shard transactions:
  Transfer money from user 42 (Shard 1) to user 1001 (Shard 2)
  → need distributed transaction or Saga pattern

Rebalancing:
  Add a 4th shard → must move data from existing shards
  → expensive, must be done live without downtime
```

---

## 9. Write Batching / Buffering

Instead of writing each event immediately, buffer in memory and flush to DB in bulk.

```
Without batching:
  Event 1 → INSERT
  Event 2 → INSERT       10,000 individual writes
  Event 3 → INSERT       10,000 round trips to DB
  ...

With batching:
  Event 1  ┐
  Event 2  ├─► Buffer ──► INSERT 10,000 rows in one batch
  Event 3  │              (one round trip, much faster)
  ...      ┘
  (flush every 100ms or 1000 events, whichever comes first)

✓ Dramatically reduces write IOPS
✓ DB can optimize bulk inserts
✗ Small window of data loss if server crashes before flush
✗ Slight write latency (buffering window)
```

---

## Summary: Which technique for which problem

```
Problem                          Solution
───────────────────────────────────────────────────────
Slow reads on small dataset    → Indexing
Reads overwhelming DB          → Caching (Redis)
Too many DB connections        → Connection Pooling
Read load > write load         → Read Replicas
Single table too large         → Partitioning
Need separate read model       → Denormalization / CQRS
Outgrown single server         → Sharding
Write spikes / high write IOPS → Write Batching
Everything at once             → All of the above, in order
```

---

## See also

- [[saga-pattern]] — handling distributed transactions when each shard/service has its own DB
- [[microservice-communication]] — how services stay in sync across separate databases
- [[aws-api-gateway]] — caching at the API layer before requests reach the DB
