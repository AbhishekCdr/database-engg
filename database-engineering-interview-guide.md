# Database Engineering — Interview Concepts in Depth

> A focused guide to the database-engineering topics that come up in system-design and backend interviews. **This document deliberately does not repeat material covered in `postgres-scan-types.md` (planner, scans, EXPLAIN) or `networking-concepts.md`.** Read all three together for full coverage.

---

## Table of Contents

1. [ACID Properties](#1-acid-properties)
2. [BASE & The Trade-off Spectrum](#2-base--the-trade-off-spectrum)
3. [CAP Theorem & PACELC](#3-cap-theorem--pacelc)
4. [Transaction Isolation Levels & Read Phenomena](#4-transaction-isolation-levels--read-phenomena)
5. [Concurrency Control: 2PL vs MVCC](#5-concurrency-control-2pl-vs-mvcc)
6. [Locking, Latches & Deadlocks](#6-locking-latches--deadlocks)
7. [Write-Ahead Logging (WAL)](#7-write-ahead-logging-wal)
8. [Index Data Structures](#8-index-data-structures)
9. [Storage Engines: Row Store vs Column Store, Heap vs Clustered](#9-storage-engines-row-store-vs-column-store-heap-vs-clustered)
10. [Join Algorithms](#10-join-algorithms)
11. [Query Optimization (Beyond Scans)](#11-query-optimization-beyond-scans)
12. [Normalization & Denormalization](#12-normalization--denormalization)
13. [Sharding & Partitioning](#13-sharding--partitioning)
14. [Replication](#14-replication)
15. [Consistency Models](#15-consistency-models)
16. [Distributed Transactions: 2PC, 3PC, Saga, TCC](#16-distributed-transactions-2pc-3pc-saga-tcc)
17. [Quorums, Read Repair, Anti-entropy](#17-quorums-read-repair-anti-entropy)
18. [Materialized Views](#18-materialized-views)
19. [Caching Strategies](#19-caching-strategies)
20. [Connection Pooling](#20-connection-pooling)
21. [Change Data Capture (CDC)](#21-change-data-capture-cdc)
22. [OLTP vs OLAP, Data Warehouses, Star/Snowflake Schemas](#22-oltp-vs-olap-data-warehouses-starsnowflake-schemas)
23. [SQL vs NoSQL: Families & When to Pick Each](#23-sql-vs-nosql-families--when-to-pick-each)
24. [Triggers, Stored Procedures, UDFs](#24-triggers-stored-procedures-udfs)
25. [Backups & Point-in-Time Recovery](#25-backups--point-in-time-recovery)
26. [Schema Migrations & Zero-Downtime Changes](#26-schema-migrations--zero-downtime-changes)
27. [Hot Topics: Bloom Filters, HyperLogLog, Vector Clocks](#27-hot-topics-bloom-filters-hyperloglog-vector-clocks)
28. [Common Interview Questions & Cheat Sheet](#28-common-interview-questions--cheat-sheet)

---

## 1. ACID Properties

The four classic guarantees for transactions.

### Atomicity
"All or nothing." A transaction either commits **fully** or has **no effect**. The DB rolls back partial work on failure.
- **How it's implemented:** Undo logs (write old values before mutating) or **shadow paging**. PostgreSQL uses MVCC + WAL: an aborted transaction's tuples are simply never marked visible.

### Consistency
The transaction takes the DB from one valid state to another. "Valid" means **all declared constraints hold** — PKs unique, FKs intact, CHECKs satisfied, triggers' invariants honored.
- It's the **application + DB constraint system's** job, not magic.
- ⚠️ Don't confuse with the "C" in CAP — different concept entirely.

### Isolation
Concurrent transactions appear to run **as if serial**. Different isolation levels relax this differently (see §4).

### Durability
Once committed, the change **survives crashes**. Implementation: **WAL flushed to disk (`fsync`) before COMMIT returns** (see §7).

### Example
```
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```
- **Atomicity** — both updates land or neither does.
- **Consistency** — `CHECK (balance >= 0)` rejects overdrafts; total money preserved.
- **Isolation** — a concurrent reader sees either both old balances or both new ones, never one of each (depending on isolation level).
- **Durability** — once `COMMIT` returns, a power loss can't undo the transfer.

---

## 2. BASE & The Trade-off Spectrum

Many distributed systems trade ACID for **BASE**:
- **B**asically **A**vailable
- **S**oft state
- **E**ventual consistency

In an interview, frame it as: ACID-style strong guarantees cost **availability and latency** during partitions. BASE picks **availability** and exposes inconsistency to the application. Real systems pick per-operation: e.g., Amazon shopping cart writes use eventual consistency, payments use strong consistency.

---

## 3. CAP Theorem & PACELC

**CAP** (Brewer): in the presence of a network **P**artition, a distributed system can guarantee at most **one** of **C**onsistency or **A**vailability.

| Choice | Behavior during partition |
|---|---|
| **CP** | Reject writes/reads to keep data consistent (HBase, MongoDB w/ majority reads, etcd, ZooKeeper) |
| **AP** | Serve every request, may return stale/conflicting data (Cassandra, DynamoDB default, Riak) |
| **CA** | Doesn't exist for real distributed systems — single-node only |

**PACELC** (Daniel Abadi, more useful): *If* there's a **P**artition, choose **A** or **C**; **E**lse (no partition), choose **L**atency or **C**onsistency.

| System | PACELC |
|---|---|
| Cassandra | PA / EL |
| DynamoDB | PA / EL (tunable) |
| Spanner | PC / EC |
| MongoDB (majority reads) | PC / EC |
| MySQL with async replication | PA / EL |

**Interview tip:** CAP is necessary background but PACELC is what you actually reason about — most of the time the network is fine, and the DB still chooses between low-latency replication and strong consistency.

---

## 4. Transaction Isolation Levels & Read Phenomena

### Phenomena

| Phenomenon | What happens |
|---|---|
| **Dirty Read** | Read uncommitted data from another tx (could be rolled back) |
| **Non-repeatable Read** | Same row read twice in one tx returns different values |
| **Phantom Read** | Same query (range) returns different *rows* (someone inserted) |
| **Lost Update** | Two txs read-modify-write same row; one's update overwritten |
| **Write Skew** | Both txs read overlapping data, write disjoint data, breaking an invariant |

### ANSI SQL levels

| Level | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | prevented | possible | possible |
| REPEATABLE READ | prevented | prevented | possible (per ANSI; PG prevents) |
| SERIALIZABLE | prevented | prevented | prevented |

### Real implementations differ
- **PostgreSQL:** has 3 effective levels — Read Committed (default), Repeatable Read (snapshot isolation, prevents phantoms too), Serializable (SSI, also prevents write skew).
- **MySQL/InnoDB:** REPEATABLE READ is default and uses next-key locks → also prevents phantoms.
- **Oracle:** Read Committed default, plus Snapshot ("Serializable" but actually SI).
- **SQL Server:** has Read Committed Snapshot (RCSI) too.

### Write skew example (snapshot isolation is not enough)
Constraint: at least one doctor must be on call.
```
T1: SELECT count(*) FROM doctors WHERE on_call;   -- sees 2
T1: UPDATE doctors SET on_call=false WHERE id=1;
T2: SELECT count(*) FROM doctors WHERE on_call;   -- also sees 2 (snapshot)
T2: UPDATE doctors SET on_call=false WHERE id=2;
T1: COMMIT;
T2: COMMIT;   -- now zero on call. Invariant broken.
```
Only **Serializable** (or explicit `SELECT ... FOR UPDATE`) prevents this.

---

## 5. Concurrency Control: 2PL vs MVCC

### Two-Phase Locking (2PL)
Every read takes a shared lock, every write an exclusive lock. **Phase 1 (growing)** acquires locks; **Phase 2 (shrinking)** only releases. Strict 2PL holds all locks until COMMIT/ROLLBACK.
- Used by older DBs and parts of MySQL.
- Pros: simple, true serializability.
- Cons: readers block writers and vice versa → throughput collapses under contention.

### MVCC — Multi-Version Concurrency Control
Each write creates a **new version** of the row instead of overwriting. Readers see a consistent **snapshot** as of their start time without taking locks.

#### How MVCC works (Postgres flavor)
- Every tuple has hidden columns: `xmin` (creating tx ID), `xmax` (deleting tx ID).
- A snapshot = `(xmin_horizon, xmax_horizon, in-progress-list)`.
- A tuple is visible to a snapshot iff its `xmin` committed before the snapshot AND its `xmax` is null/aborted/after the snapshot.
- **UPDATE = INSERT new + mark old's `xmax`**. Old row stays until VACUUM removes it.
- **VACUUM** reclaims dead tuples; **autovacuum** does it on a schedule.

#### MVCC in MySQL (InnoDB)
- Versions stored in the **undo log**, not in the table.
- A `SELECT` walks back through undo records to reconstruct the row as of the snapshot.

#### MVCC trade-offs
- ✅ Readers never block writers, writers never block readers.
- ✅ Excellent for read-heavy workloads.
- ❌ Bloat from dead tuples; need vacuuming.
- ❌ Long-running transactions hold back the "horizon" → vacuum can't reclaim → table bloats.

---

## 6. Locking, Latches & Deadlocks

### Lock granularity
| Level | Pros | Cons |
|---|---|---|
| Database | Simplest | Kills concurrency |
| Table | Easy | Low concurrency |
| Page | Decent | Hot pages contend |
| Row | High concurrency | Most overhead, escalation needed |

### Lock modes (typical)
- **Shared (S)** — read lock; others can read.
- **Exclusive (X)** — write lock; no one else.
- **Intent (IS/IX)** — declared at higher levels (table) before fine-grained locks at row level.
- **Update (U)** — pre-promotion mode to avoid deadlocks.
- Plus advisory locks (app-defined) and predicate/range locks.

### Latches vs locks
- **Locks** are *logical* (transaction-scoped, deadlock-detected).
- **Latches** are *physical* (mutex on a buffer page, microseconds, no deadlock detection).

### Deadlock
Two tx each hold a lock the other wants.
```
T1: locks row A → wants row B
T2: locks row B → wants row A          → deadlock
```
- DBs run a **wait-for graph** detector and abort the cheaper victim with `ERROR: deadlock detected`.
- Mitigations: **always lock in the same order**, use `SELECT ... FOR UPDATE NOWAIT / SKIP LOCKED`, keep transactions short.

### `SELECT ... FOR UPDATE` patterns
```sql
-- Worker queue, skip rows another worker has
SELECT id FROM jobs WHERE status='pending'
ORDER BY id LIMIT 10
FOR UPDATE SKIP LOCKED;
```

---

## 7. Write-Ahead Logging (WAL)

### The rule
**Before modifying a data page on disk, write the change to a sequential log first and `fsync` it.**

### Why
- Random page writes are slow; sequential log writes are fast.
- On crash, replay the log to rebuild any data-page changes that hadn't been flushed.
- Decouples durability (log flushed) from data-page writes (lazy via background `bgwriter` / `checkpointer`).

### Anatomy of a WAL record (Postgres)
- LSN (Log Sequence Number — monotonic byte offset).
- Transaction ID.
- Resource manager (heap, btree, ...).
- Old/new tuple images (for redo/undo).

### Recovery flow
1. On startup, find last **checkpoint** in pg_control.
2. **Redo** every WAL record after that LSN against the data files.
3. Roll back any transactions that hadn't committed (using their WAL UNDO info or via aborted-tx tracking).

### Group commit
Multiple committing tx's `fsync` together → amortizes disk-write cost.

### WAL also enables
- **Streaming replication** — ship the log to a replica that replays it.
- **Point-in-time recovery (PITR)** — restore base backup, then replay WAL up to a target time/LSN.
- **Logical replication / CDC** — decode WAL records into row-level events.

---

## 8. Index Data Structures

> Postgres-specific scan *behavior* lives in `postgres-scan-types.md`. This section covers the **structures themselves** — useful for any DB.

### B-Tree / B+Tree (the workhorse)
- Self-balancing, O(log n) lookup, range scan, ordered traversal.
- B+Tree variant: only **leaves** hold data/pointers; internal nodes are routing only; **leaves linked** for range scans.
- Used by virtually every relational DB (MySQL InnoDB, Postgres, SQL Server) and many key-value stores.
- Page-oriented (4KB–32KB nodes), high fan-out → small height even for huge tables.

```
                    [50 | 100]
                   /     |     \
              [10|30] [60|80] [120|150]
              leaves linked: 10→30→60→80→120→150
```

**When it shines:** equality, range (`BETWEEN`, `<`, `>`), prefix on text, `ORDER BY`.

### Hash Index
- O(1) average equality lookup.
- No range queries.
- Used in MySQL Memory engine, Postgres `USING HASH`, in-memory caches.

### LSM Tree (Log-Structured Merge)
Powers RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, Bigtable.

```
Writes → MemTable (in-memory sorted map)
       → flush to immutable SSTable on disk
       → background compaction merges SSTables
Reads  → check MemTable, then SSTables newest-to-oldest
       → Bloom filter per SSTable to skip
```
- **Write-optimized:** all writes are sequential appends.
- **Reads:** may touch multiple SSTables (mitigated by Bloom filters + tiered/leveled compaction).
- **Trade-off:** write amplification (compaction rewrites data many times); space amplification.

### Hash + LSM hybrid: Bitcask
Single in-memory hash table → on-disk append log.

### GIN — Generalized Inverted Index
- Inverted list per token: `token → [doc1, doc2, ...]`.
- Used for full-text search (`tsvector`), JSONB containment, arrays, trigrams.
- **Slow writes** (each inserted doc touches many posting lists), **fast multi-token queries**.

### GiST — Generalized Search Tree
- Framework for building balanced trees over arbitrary types: geometric (PostGIS), ranges, full-text.
- Supports `KNN` searches (`<->` distance) — vital for spatial queries.

### SP-GiST — Space-Partitioned GiST
- Unbalanced tree for partitioned data (quad-trees, kd-trees, suffix tries).

### BRIN — Block Range Index
- Stores per-block-range summary (min/max).
- Tiny (KBs vs GBs of B-tree).
- Works only when data is **physically correlated** with the indexed column (e.g., monotonically increasing timestamps).

### R-Tree
- Spatial bounding-box indexes; for "all rectangles intersecting this region."

### Bitmap Index (DW-style)
- One bit per row per distinct value.
- Tiny for low-cardinality columns; AND/OR via bitwise ops.
- Used by columnar / analytics DBs (Druid, ClickHouse).

### Inverted Index
- Same idea as GIN but generalized — used by Lucene/Elasticsearch.
- Skip lists, posting list compression (delta + varint), positions for phrase search.

### Choosing an index
| Workload | Pick |
|---|---|
| OLTP equality + range | B-tree |
| Equality only, in-memory | Hash |
| Write-heavy log/timeseries | LSM |
| Full-text / JSON | GIN / Inverted |
| Spatial | GiST / R-tree |
| Append-only timeseries | BRIN |

---

## 9. Storage Engines: Row Store vs Column Store, Heap vs Clustered

### Row Store
- Tuples stored together (all columns of one row in one place).
- Great for OLTP — fetch one whole row by PK is one I/O.
- Postgres heap, MySQL InnoDB, SQL Server (default), Oracle.

### Column Store
- All values of one column stored contiguously.
- Excellent compression (similar values together — RLE, dict, delta, bit-packing).
- Reads only the columns the query references → massive I/O reduction for analytics.
- Vectorized execution operates on chunks of column values in CPU registers.
- Examples: ClickHouse, BigQuery, Redshift, Snowflake, DuckDB, Parquet.

```
Row store:
[id1,name1,age1] [id2,name2,age2] [id3,name3,age3]

Column store:
id:  [id1, id2, id3]
name:[name1, name2, name3]
age: [age1, age2, age3]
```

### Heap-organized (most relational DBs)
- Rows stored in arbitrary insertion order.
- Indexes hold **(value → row-pointer)**.
- A row's physical location is unstable; non-HOT updates move it.

### Clustered / Index-organized (MySQL InnoDB, SQL Server clustered index)
- The **table itself** is stored as a B+tree keyed by the PK.
- PK lookup is one tree walk — no separate heap fetch.
- Secondary indexes store **PK value**, not a physical pointer → one extra B-tree walk per secondary lookup ("double lookup").
- Range scans on the PK are sequential, which is fast.

### HTAP (Hybrid)
- Both at once: **TiDB**, **SingleStore**, **SQL Server columnstore on rowstore**. Lets one DB serve both OLTP writes and OLAP reads.

---

## 10. Join Algorithms

The three textbook joins are the answer when an interviewer asks "how does a SQL join actually work."

### Nested Loop Join
```
for each row r in outer:
    for each row s in inner:
        if r.key == s.key: emit
```
- O(N×M); cheap if inner side has an index on the join key (lookup per outer row).
- Best when one side is **tiny** or the inner has an index.

### Hash Join
```
build phase: hash table on inner.key  (in memory; spill to disk if needed)
probe phase: for each outer row, probe the hash table
```
- O(N+M).
- Best for **equi-joins** on large unsorted datasets.
- Memory-hungry; if inner side > `work_mem`, spills to disk in batches (Grace hash join).

### Sort-Merge Join
```
sort outer by key
sort inner by key
walk both in lockstep, merging
```
- O(N log N + M log M); free if both inputs already ordered (e.g., by an index).
- Best when both sides are **already sorted** or sortable cheaply.
- Supports range joins, not just equality.

### How the planner picks
- Hash join wins for big × big equi-joins.
- Nested loop wins for small × indexed-big.
- Merge join wins when an `ORDER BY` is needed anyway.

### Bonus: Semi/Anti/Outer
- **Semi-join**: `EXISTS` / `IN` — emit outer row once if any match.
- **Anti-join**: `NOT EXISTS` — emit outer row if no match.
- **Outer join**: pad missing side with NULLs.

---

## 11. Query Optimization (Beyond Scans)

### Predicate pushdown
Push WHERE clauses as close to the scan as possible — apply filters before joins to shrink intermediates.

### Projection pushdown
Only read the columns the query needs (huge win in column stores).

### Join reordering
For N-way joins, the optimizer enumerates orders ("left-deep tree", "bushy tree") to minimize intermediate result sizes. Cardinality estimates drive this; bad estimates → bad plans.

### Subquery flattening
`SELECT ... FROM (SELECT ...)` is rewritten as one query when semantically equivalent.

### Constant folding & expression simplification
`WHERE 1=1 AND active=true` → `WHERE active=true`.

### Predicate inference
`a = b AND b = 5` → also infers `a = 5`, can use index on `a`.

### Common interview "why is my query slow?" checklist
1. `EXPLAIN ANALYZE` — what's the plan? Estimates vs actual rows?
2. Is the predicate **sargable**? `WHERE LOWER(email)='x'` defeats an index unless you index `LOWER(email)`.
3. Stats stale? Run `ANALYZE`.
4. Right index? Composite-index column order matters: `(a,b)` index helps `WHERE a=? AND b=?` and `WHERE a=?`, but not `WHERE b=?` alone.
5. Type mismatch? `WHERE int_col = '5'` may force a cast that breaks index usage.
6. Index-defeating wildcards: `LIKE '%foo%'` cannot use a B-tree (use trigram/GIN).
7. N+1 query in app code — should be a single JOIN.

---

## 12. Normalization & Denormalization

### Normal forms
| Form | Rule |
|---|---|
| **1NF** | Atomic values, no repeating groups |
| **2NF** | 1NF + every non-key attribute depends on the whole PK (no partial dependencies) |
| **3NF** | 2NF + no transitive dependencies (`A → B → C`) |
| **BCNF** | Stronger 3NF — every determinant is a candidate key |
| **4NF / 5NF** | Eliminate multi-valued / join dependencies |

### Why normalize
- Eliminates update anomalies (update one fact in one place).
- Smaller storage; cleaner constraints.

### Why denormalize
- Reads are too slow (too many joins).
- Trade write complexity for read speed.
- Common in analytics (star schemas) and in DBs without efficient joins (most NoSQL).

### Practical guidance
> Normalize until it hurts; denormalize until it works.

Use **materialized views**, **summary tables**, or **CDC-driven cache**s for read-heavy projections rather than denormalizing the source of truth.

---

## 13. Sharding & Partitioning

### Partitioning (single-DB)
Split one logical table into multiple physical ones inside one DB:
- **Range** — by date ranges (`2026_q1`, `2026_q2`).
- **List** — by enum (`region IN ('us','eu')`).
- **Hash** — by hash of key (even spread).
- **Composite** — range-then-hash, etc.

**Benefits:** prune partitions at planning, smaller indexes, parallel maintenance, drop-old-data via `DROP PARTITION`.

### Sharding (distributed)
Split data across multiple DB instances. Each shard is a complete DB.

#### Sharding strategies
| Strategy | Mechanic | Pro | Con |
|---|---|---|---|
| **Hash** | `shard = hash(key) mod N` | Even spread | Resharding moves most data; range queries fan out |
| **Range** | `shard = lookup(key)` | Range queries efficient | Hotspots on monotonic keys |
| **Directory / Lookup** | service maps key → shard | Flexible | Lookup service is SPOF |
| **Geo** | by user region | Locality, compliance | Cross-region joins hard |
| **Consistent Hashing** | hash ring | Adding/removing shards moves only `1/N` of keys | More complex |

### Resharding pain
Naive `mod N` means changing `N` invalidates almost every key's mapping. **Consistent hashing** (Cassandra, DynamoDB) and **virtual buckets** (Redis Cluster's 16384 slots) make rebalancing incremental.

### Cross-shard queries
- **Scatter-gather** — query every shard, merge results.
- Costly; design schema so common queries hit one shard.

### Choosing a shard key
- High cardinality (avoid hotspots).
- Common in your most frequent queries.
- Doesn't change after creation (changing a shard key = move the row).

---

## 14. Replication

### Topologies
- **Single-leader (primary–replica)** — writes to leader, reads from anywhere; common in Postgres, MySQL.
- **Multi-leader** — writes to multiple leaders; needs conflict resolution (timestamps, vector clocks, CRDTs).
- **Leaderless (Dynamo-style)** — any node accepts writes; quorums + read repair (Cassandra, Riak, DynamoDB).

### Sync vs Async
| Mode | Behavior | Trade-off |
|---|---|---|
| **Synchronous** | Leader waits for replica ACK before COMMIT | Zero data loss on failover; latency hit; replica down stalls writes |
| **Asynchronous** | Leader commits, ships log later | Fast; possible data loss on failover |
| **Semi-sync** | Wait for ≥1 replica's ACK (at network only, not apply) | Common middle ground |

### How it works (Postgres streaming replication)
- WAL streamed from primary to standby over TCP.
- Standby replays WAL into its data files.
- Hot standbys serve read-only queries (with replication lag).

### Read scaling
Read replicas absorb SELECT load. **Replication lag** means stale reads — use **read-your-writes** policies (route a user's reads to primary for a few seconds after writes) when needed.

### Failover
- **Automatic:** orchestrators (Patroni, Orchestrator) detect leader failure, promote a standby, repoint clients.
- **Split-brain risk:** two leaders accept writes simultaneously after a network blip → use fencing / STONITH / quorum.

---

## 15. Consistency Models

From strongest (most expensive) to weakest:

| Model | Guarantee |
|---|---|
| **Linearizability** | Each op appears to happen instantaneously at some point between invocation and response. Single global order respecting real-time. |
| **Sequential** | Some global order consistent with each client's program order (real-time gaps allowed). |
| **Causal** | If op A causally precedes op B (B saw A), every observer sees A before B. |
| **Read-your-writes** | A client never sees a state older than its own writes. |
| **Monotonic reads** | Within a session, reads don't go backward in time. |
| **Eventual** | If no new writes, all replicas eventually converge. |

### Common combos
- **Strong consistency** = linearizability (Spanner, etcd, single-leader Postgres on primary reads).
- **Strong eventual consistency** = eventual + CRDT convergence (Riak, automerge).

---

## 16. Distributed Transactions: 2PC, 3PC, Saga, TCC

### Two-Phase Commit (2PC)
```
Coordinator                     Participants
   │  ── PREPARE ──────────►       │  vote YES (locks held) / NO
   │  ◄── votes ───────────        │
   │  if all YES → COMMIT          │  apply, release locks
   │  else        → ABORT          │  roll back
```
- **Problem:** if coordinator crashes after PREPARE but before COMMIT, participants are stuck holding locks ("blocking protocol").

### Three-Phase Commit (3PC)
Adds a **PRE-COMMIT** phase + timeouts → non-blocking under crash-stop assumptions, but vulnerable to network partitions. Rarely used in practice.

### Saga
Long-lived business transaction broken into **local transactions**, each with a **compensating action**.
```
Reserve hotel → Reserve flight → Charge card
If charge fails: cancel flight, cancel hotel.
```
- Two flavors: **choreography** (events drive next step) and **orchestration** (central conductor).
- No global locks → high availability; but isolation is the app's problem.

### TCC — Try / Confirm / Cancel
- **Try:** reserve resources (e.g., debit a held balance).
- **Confirm:** finalize (apply hold).
- **Cancel:** release hold.
- Used in payments, booking systems.

### Outbox Pattern
For "publish event after DB commit" problems, write the event into an `outbox` table in the **same** DB transaction as the business write; a separate process reads the outbox and publishes to the broker. Avoids dual-write inconsistency.

---

## 17. Quorums, Read Repair, Anti-entropy

### Quorum formula
With `N` replicas, write quorum `W`, read quorum `R`:
- **Strong consistency** when `W + R > N` (read set overlaps every write set).
- Tunable per request (Cassandra: `ONE`, `QUORUM`, `ALL`, `LOCAL_QUORUM`, etc.).

### Examples
- `N=3, W=2, R=2` — strong consistency, tolerates 1 down node.
- `N=3, W=1, R=1` — fastest, weakest.
- `N=3, W=3, R=1` — fast reads, slow writes.

### Read repair
On a read, if replicas disagree, the coordinator pushes the latest value to lagging replicas synchronously (blocking) or asynchronously.

### Hinted handoff
A coordinator that can't reach a replica stores a "hint" and replays it when the replica returns.

### Anti-entropy / Merkle trees
Periodic background process compares replica trees of hashes to find divergent ranges and reconcile.

---

## 18. Materialized Views

A **regular view** is just a stored query — re-evaluated each time.
A **materialized view** stores the result. It must be **refreshed** when underlying data changes.

### Refresh strategies
- **Full refresh:** recompute the whole thing.
- **Incremental refresh:** apply deltas (Postgres has `REFRESH MATERIALIZED VIEW CONCURRENTLY`; Oracle has fast refresh on commit).
- **Streaming MVs:** auto-maintained by the engine on every write (ksqlDB, RisingWave, Materialize, Snowflake dynamic tables).

### When to use
- Expensive aggregations queried frequently.
- Pre-joined denormalized projections.
- A staleness budget you can afford.

---

## 19. Caching Strategies

### Patterns

| Pattern | Read | Write |
|---|---|---|
| **Cache-aside** (lazy) | App: cache miss → load from DB → put in cache | App: write DB, then **invalidate** or **update** cache |
| **Read-through** | Cache loads from DB on miss (transparent to app) | App writes to DB |
| **Write-through** | Same as read-through | App writes to cache → cache writes synchronously to DB |
| **Write-back** (write-behind) | — | App writes to cache; cache flushes to DB asynchronously (durability risk!) |
| **Write-around** | — | App writes directly to DB, bypassing cache; cache populated only on subsequent reads |

### Eviction policies
- **LRU** — Least Recently Used (most common; doubly-linked list + hashmap).
- **LFU** — Least Frequently Used.
- **FIFO** — first in first out.
- **TTL** — time-based.
- **ARC** — Adaptive Replacement Cache (LRU + LFU hybrid).
- **Random / 2-Random** — surprisingly competitive, no metadata needed.

### Pitfalls
- **Cache stampede / thundering herd:** many requests miss the same hot key; all hit DB. Fix with **request coalescing** (single-flight), **probabilistic early refresh**, or a **mutex / lease**.
- **Stale data:** cache invalidation is famously hard. Use TTLs + invalidation events (CDC).
- **Cache penetration:** queries for keys that don't exist always miss → bypass with negative caching or **Bloom filter** in front.
- **Hot keys:** one key gets 90 % of traffic → shard the value, replicate it across nodes.

### Eventual-consistency window
With write-through to DB then publish-to-cache, a brief window exists where DB has new value but cache hasn't been updated. Choose between **invalidate-on-write** (cache miss next read) and **update-on-write** (race window, but no extra DB read).

---

## 20. Connection Pooling

### Why
Opening a TCP + TLS + auth handshake to a DB is **expensive** (10s of ms) compared to running a query (<1 ms). Pools reuse open sessions.

### Where the pool lives
- **In-process** (driver-level): HikariCP (Java), `pg` for Node, SQLAlchemy pool.
- **External proxy:** **PgBouncer**, **Pgpool-II**, **RDS Proxy**, **ProxySQL**.

### Pool modes (PgBouncer)
| Mode | Behavior | Compatibility |
|---|---|---|
| **Session** | Client owns the server session for its lifetime | Full feature support |
| **Transaction** | Pool reassigns server connection per transaction | No prepared statements / session GUCs / advisory locks across calls |
| **Statement** | Pool reassigns per query | Almost no transactions; rarely used |

### Sizing
- Don't max out: each Postgres backend = one OS process + ~10MB RAM.
- Rule of thumb: pool size ≈ `2 × CPU_cores + effective_spindle_count` per app instance.
- More connections doesn't mean more throughput — past a point, contention dominates.

### Common bug
App pool > DB `max_connections`, app crashes mid-deploy → DB refuses new connections (`too many clients already`). Always cap pool below DB limit *across all app instances combined*.

---

## 21. Change Data Capture (CDC)

Stream every row-level change in the DB as events.

### How
- **Trigger-based:** triggers write to an audit table; a job ships rows. Slow, intrusive.
- **Log-based:** read the WAL / binlog directly. **Debezium** for Kafka, Postgres logical replication, MySQL `binlog_format=ROW`. Low overhead, near-real-time.

### Use cases
- Replicate to a search index (Elasticsearch).
- Update caches consistently (avoid dual-write).
- Stream to a data warehouse / lake.
- Materialized view pipelines.

### Schema for events
```json
{
  "op": "u",                       // c=create, u=update, d=delete
  "ts_ms": 1714723200000,
  "before": {"id": 1, "name": "Old"},
  "after":  {"id": 1, "name": "New"},
  "source": {"db":"shop","table":"users","lsn":"0/1A2B3C"}
}
```

### Outbox + CDC
The most reliable "write to DB and publish event" pattern: write to outbox in same TX, CDC ships the outbox row to Kafka.

---

## 22. OLTP vs OLAP, Data Warehouses, Star/Snowflake Schemas

### OLTP — Online Transaction Processing
- Many small reads/writes; latency-critical.
- Highly normalized; row-store; ACID.
- Examples: bank, e-commerce checkout.

### OLAP — Online Analytical Processing
- Few, huge analytical reads; bulk loads.
- Denormalized; column-store; vectorized; ACID often relaxed.
- Examples: dashboards, BI, ML feature stores.

### Star schema
- Central **fact table** (e.g., `sales`) with measure columns and FK to dimensions.
- **Dimension tables** describe each FK (date, product, store, customer).
- Wide, denormalized dimensions → joins are simple star-shaped.

```
        dim_date
            │
dim_product── fact_sales ──dim_customer
            │
        dim_store
```

### Snowflake schema
- Same as star but dimensions are normalized into sub-tables (e.g., `dim_product → dim_category`).
- Less storage, more joins.

### Modern lakehouse
- Files (Parquet/ORC) on object storage + table format (**Iceberg**, **Delta**, **Hudi**) + query engine (Spark, Trino, Snowflake, Databricks). Brings ACID + time travel + schema evolution to data lakes.

---

## 23. SQL vs NoSQL: Families & When to Pick Each

| Family | Examples | Data Model | Best For |
|---|---|---|---|
| **Relational** | Postgres, MySQL, SQL Server, Oracle | Tables, joins, ACID | Anything where data has relationships and you need consistency |
| **Document** | MongoDB, Couchbase, DynamoDB (single-table) | JSON-ish docs | Schemaless / nested data, rapid iteration |
| **Key-Value** | Redis, DynamoDB, RocksDB, LevelDB | Opaque blob by key | Caches, session stores, queues |
| **Wide-Column** | Cassandra, ScyllaDB, HBase, Bigtable | Sparse 2D map (row, col) → val | Massive write throughput, time-series, log data |
| **Graph** | Neo4j, JanusGraph, Amazon Neptune | Nodes + edges + properties | Relationships are the workload (social, fraud, knowledge graphs) |
| **Time-Series** | InfluxDB, TimescaleDB, Prometheus | Timestamped points + tags | Metrics, IoT |
| **Search** | Elasticsearch, OpenSearch, Solr | Inverted index | Full-text search, log analytics |
| **Vector** | pgvector, Pinecone, Weaviate, Milvus | High-dim float vectors | Embedding similarity for AI/ML |
| **NewSQL / Distributed SQL** | CockroachDB, YugabyteDB, Spanner, TiDB | Relational + horizontally scalable | When you need SQL **and** Cassandra-style scale |

### Interview heuristic
> If the data has clear relations and you'll outgrow one server in a year or two, start with Postgres. Re-evaluate if you actually hit limits.

---

## 24. Triggers, Stored Procedures, UDFs

### Trigger
DB-side function that fires on INSERT/UPDATE/DELETE/TRUNCATE — `BEFORE`, `AFTER`, or `INSTEAD OF`. Per-row or per-statement.
- ✅ Enforce invariants the app might forget.
- ❌ Easy to make queries mysteriously slow; debugging is painful.

### Stored procedure / UDF
Code that runs inside the DB (PL/pgSQL, T-SQL, PL/SQL).
- ✅ Reduces network round-trips for batch logic.
- ✅ Centralizes complex queries.
- ❌ Couples logic to a specific DB; harder to version-control.
- ❌ Profiling and observability worse than app code.

Modern preference is to keep most logic in the app, using DB only for set-based operations and integrity.

---

## 25. Backups & Point-in-Time Recovery

### Backup types
- **Full** — entire DB.
- **Incremental** — only blocks changed since last backup.
- **Differential** — changes since last full.
- **Logical** — `pg_dump` / `mysqldump`; portable, slow on huge DBs.
- **Physical** — file-system-level snapshot of data dir; fast restore, version-locked.

### Point-in-Time Recovery (PITR)
1. Take a base backup.
2. Continuously archive WAL.
3. To restore at time T: restore base backup → replay WAL up to T.

### 3-2-1 rule
3 copies, on 2 different media, with 1 offsite.

### Test restores
> A backup you've never restored is not a backup. Run a restore drill quarterly.

---

## 26. Schema Migrations & Zero-Downtime Changes

### The hard ones
- **Adding a NOT NULL column with default** — older Postgres versions rewrote the whole table (slow, locks). Postgres 11+ optimizes constant defaults.
- **Renaming a column / table** — breaks deployed app versions during a rolling deploy.
- **Changing a column type** — table rewrite + locks.
- **Adding an index on a huge table** — `CREATE INDEX CONCURRENTLY` (Postgres) avoids a write lock but takes longer.
- **Dropping a column** — fine, but only after no app version reads it.

### Expand–Contract pattern
The reliable recipe for breaking changes:
1. **Expand:** add the new shape (column, table, FK) **alongside** the old.
2. Backfill data.
3. Deploy app version that **double-writes** old + new.
4. Switch reads to new.
5. **Contract:** stop writing old; drop old.

Each step is independently deployable and reversible.

### Online schema change tools
- **gh-ost** (MySQL) — copies data into a new table while replaying binlog of changes.
- **pt-online-schema-change** — older tool with triggers.
- Postgres: `CREATE INDEX CONCURRENTLY`, `ALTER TABLE ... ADD COLUMN ... DEFAULT ... NOT NULL` (PG 11+), `pg_repack`.

---

## 27. Hot Topics: Bloom Filters, HyperLogLog, Vector Clocks

### Bloom filter
- Probabilistic set membership: "definitely not in set" or "probably in set."
- Bit array + k hash functions; configurable false positive rate.
- Used by LSM-tree DBs to skip SSTable lookups, by CDNs to avoid origin hits, by Bitcoin/SPV.

```
add(x): set bits at h1(x), h2(x), ..., hk(x)
test(x): all those bits set? → maybe; any 0? → definitely no
```

### Counting Bloom filter
Replace bits with small counters → supports **delete**.

### HyperLogLog
- Probabilistic **cardinality estimation** ("how many distinct users today?").
- ~1.5 KB to estimate billions with ~2 % error.
- Built on bit-pattern trick: longest run of leading zeros in a hash → log2(cardinality) estimate; HLL averages many such estimators.

### Count-Min Sketch
- Probabilistic **frequency** estimation. "How many times have we seen X?" with bounded over-count.

### Vector clocks / Lamport timestamps
- Track causality across distributed events without a global clock.
- **Lamport timestamp:** scalar; on send, `t = t+1`; on receive, `t = max(t, t_msg)+1`. Total order, not causality.
- **Vector clock:** map of `node → counter`; partial order capturing **happens-before**. Used by Dynamo, Riak, version vectors in CRDTs.

### CRDTs — Conflict-free Replicated Data Types
- Data structures (counters, sets, maps, sequences) where independent updates **always merge** without conflict.
- Used by Riak, Redis CRDB, Automerge, collaborative editors (Figma).

---

## 28. Common Interview Questions & Cheat Sheet

### Quick-fire answers

**Q: Difference between `WHERE` and `HAVING`?**
A: `WHERE` filters rows before grouping; `HAVING` filters groups after `GROUP BY`.

**Q: `UNION` vs `UNION ALL`?**
A: `UNION` removes duplicates (extra sort/hash); `UNION ALL` keeps them and is faster.

**Q: `DELETE` vs `TRUNCATE` vs `DROP`?**
A: `DELETE` is per-row, transactional, fires triggers, slow on big tables. `TRUNCATE` deallocates pages, fast, minimal logging, can't be filtered. `DROP` removes the table entirely.

**Q: Clustered vs non-clustered index?**
A: Clustered = table is *stored as* the index (one per table). Non-clustered = separate structure pointing back to the row.

**Q: Composite index column order?**
A: Most-selective and most-frequently-filtered columns first; equality predicates before range; an index on `(a,b,c)` helps `a`, `a+b`, `a+b+c` — not `b` alone.

**Q: When does an index NOT help?**
A: Very small table, low-selectivity predicate, predicate not sargable, function applied to indexed column, type mismatch, leading wildcard `LIKE '%x'`.

**Q: Optimistic vs pessimistic concurrency?**
A: Pessimistic = lock first, do work, commit (DB row locks). Optimistic = read with version, do work, on commit check version unchanged (else retry). Optimistic wins for low-contention workloads.

**Q: How does Postgres handle UPDATE under MVCC?**
A: Inserts a new tuple version; marks old's `xmax`. Old becomes dead; vacuum reclaims later.

**Q: How would you design a URL shortener?** *(classic system design)*
A: Hash → short code → DB. Sharded KV (DynamoDB / Cassandra) for the mapping. Cache hot codes in Redis. Counter or random-collision-retry for code generation. Read-heavy → many replicas.

**Q: How would you design Twitter timeline?**
A: Fanout-on-write (push to followers' inboxes) for most users; fanout-on-read (pull from followees) for celebrities. Mix called "hybrid fanout."

**Q: What happens if you `SELECT *` in production?**
A: Reads all columns even if you need two — defeats covering indexes, more I/O, more network, breaks if schema adds wide columns. Always list columns explicitly.

**Q: Why is `OFFSET 1000000` slow?**
A: Engine still walks and discards a million rows. Use **keyset / seek pagination**: `WHERE id > last_seen_id ORDER BY id LIMIT 20`.

### Cheat-sheet: Pick the right tool

| Symptom | Likely answer |
|---|---|
| "We need transactions across rows" | Relational |
| "Schema changes weekly" | Document |
| "100k writes/sec time-series" | Wide-column / time-series DB |
| "Find friends-of-friends in <100ms" | Graph DB |
| "Search by partial text + facets" | Elasticsearch |
| "Find similar embeddings for RAG" | Vector DB |
| "Web sessions, ephemeral" | Redis / Memcached |
| "Multi-region, strong consistency" | Spanner / CockroachDB |
| "Read-heavy + tolerable staleness" | Read replicas + cache |

---

## Recommended Further Reading

- ***Designing Data-Intensive Applications*** — Martin Kleppmann (the canonical reference)
- ***Database Internals*** — Alex Petrov (storage engines, distributed protocols)
- ***Readings in Database Systems*** ("the Red Book") — Stonebraker & Hellerstein, free online
- ***The Internals of PostgreSQL*** — Suzuki Hironobu (free online)
- ***SQL Performance Explained*** — Markus Winand (`use-the-index-luke.com`)
- Jepsen analyses (`jepsen.io`) — empirical correctness reports of distributed DBs
- Aphyr's "Hermitage" — isolation-level test suite & write-ups
