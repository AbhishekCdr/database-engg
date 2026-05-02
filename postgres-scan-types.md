# PostgreSQL Scan Types — A Comprehensive Internal Guide

> A deep dive into every scan node the PostgreSQL planner/executor can produce, how each works internally, when the planner picks it, and how to read it in `EXPLAIN`.

---

## Table of Contents

1. [Background: How PostgreSQL Reads Data](#1-background-how-postgresql-reads-data)
2. [Sequential Scan (Seq Scan)](#2-sequential-scan-seq-scan)
3. [Index Scan](#3-index-scan)
4. [Index Only Scan](#4-index-only-scan)
5. [Bitmap Index Scan + Bitmap Heap Scan](#5-bitmap-index-scan--bitmap-heap-scan)
6. [TID Scan](#6-tid-scan)
7. [TID Range Scan](#7-tid-range-scan)
8. [Sample Scan (TABLESAMPLE)](#8-sample-scan-tablesample)
9. [Function Scan](#9-function-scan)
10. [Values Scan](#10-values-scan)
11. [CTE Scan](#11-cte-scan)
12. [Subquery Scan](#12-subquery-scan)
13. [WorkTable Scan (Recursive CTE)](#13-worktable-scan-recursive-cte)
14. [Named Tuplestore Scan](#14-named-tuplestore-scan)
15. [Result Node](#15-result-node)
16. [Foreign Scan (FDW)](#16-foreign-scan-fdw)
17. [Custom Scan](#17-custom-scan)
18. [Parallel Scans](#18-parallel-scans)
19. [Append, Merge Append & Partition Pruning](#19-append-merge-append--partition-pruning)
20. [How the Planner Chooses a Scan](#20-how-the-planner-chooses-a-scan)
21. [Reading EXPLAIN — Field Reference](#21-reading-explain--field-reference)
22. [Quick-Reference Summary Table](#22-quick-reference-summary-table)
23. [Video Notes — Bitmap Index Scan in Postgres](#23-video-notes--bitmap-index-scan-in-postgres)

---

## 1. Background: How PostgreSQL Reads Data

PostgreSQL stores tables as a **heap**: an unordered collection of 8 KB pages (`BLCKSZ`). Each page holds many tuples, each addressed by a **CTID** (`(block_number, offset_within_block)`). Indexes are separate structures (B-tree, Hash, GIN, GiST, SP-GiST, BRIN) that map indexed values to CTIDs.

A **scan node** in the executor's plan tree is responsible for producing a stream of tuples from some source — a base table, an index, a function, a constant set, a remote server, etc. Every query begins with one or more scan nodes at the leaves; joins, aggregations, sorts, etc. consume their output.

Key internal concepts that drive scan choice:

| Concept | Meaning |
|---|---|
| **Buffer Pool (`shared_buffers`)** | In-memory cache of 8 KB pages |
| **Visibility Map (VM)** | One bit per heap page indicating "all-visible" — enables Index Only Scans |
| **MVCC** | Each tuple has `xmin`/`xmax`; visibility check requires the heap page unless VM says all-visible |
| **HOT chains** | In-page tuple chains for non-indexed updates |
| **Cost model** | `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`, `effective_cache_size` |
| **MCV / Histograms** | Stored in `pg_statistic` to estimate selectivity |

The planner enumerates candidate scan paths for each relation, costs them, and the cheapest path wins (subject to GUC overrides like `enable_seqscan=off`).

---

## 2. Sequential Scan (Seq Scan)

### What it does
Reads the heap page-by-page from block 0 to the end (`relpages`), evaluates the filter predicate on every tuple, and emits rows that pass.

### Internal mechanics
- Implemented in `nodeSeqscan.c` using the table-AM API (`table_beginscan` / `table_scan_getnextslot`).
- Uses a **ring buffer** (256 KB by default for sequential reads via `BAS_BULKREAD`) so a large scan does not evict the whole shared buffer cache.
- **Synchronized scans** (`synchronize_seqscans=on`): if another backend is already scanning the same table, this scan starts at that backend's current block. Both finish in roughly one pass instead of two — they share I/O.
- Visibility check happens per tuple via `HeapTupleSatisfiesMVCC`.

### When the planner picks it
- Small tables (cheaper to read 50 pages sequentially than to do random index lookups).
- Predicates expected to match a large fraction of rows (rule of thumb: > 5–10 %).
- No usable index for the predicate.
- `random_page_cost` is high relative to `seq_page_cost` (default 4.0 vs 1.0 — for SSDs, lowering to 1.1 can flip plans toward indexes).

### EXPLAIN example
```
Seq Scan on orders  (cost=0.00..18584.00 rows=12 width=72)
  Filter: (status = 'CANCELLED'::text)
  Rows Removed by Filter: 999988
```

### Pros / Cons
| Pros | Cons |
|---|---|
| Cheap I/O per page (sequential read) | Touches every page even for one row |
| No index maintenance overhead | Bad on large tables with selective predicates |
| Always available | High `Rows Removed by Filter` is a smell |

---

## 3. Index Scan

### What it does
Walks an index in key order, fetching the matching CTID from each index entry, then reads the corresponding heap tuple to apply remaining filters and check visibility.

### Internal mechanics
- Implemented in `nodeIndexscan.c`.
- For each index entry that matches the **scan keys** (operators pushed down to the index AM), the executor calls `index_getnext_tid` → `table_index_fetch_tuple`.
- Two predicate categories:
  - **Index Condition** — evaluable inside the index (`indxquals`).
  - **Filter** — re-checked on the heap tuple (`qpquals`).
- I/O pattern is **random** on the heap (each CTID may live on a different page).
- Returns rows in **index order** — useful for `ORDER BY` to skip a sort.

### When the planner picks it
- Highly selective predicates (small fraction of rows match).
- Queries that benefit from index ordering (`ORDER BY indexed_col LIMIT n`).
- Small `LIMIT` values where stopping early matters.

### EXPLAIN example
```
Index Scan using orders_customer_id_idx on orders
  (cost=0.42..8.45 rows=1 width=72)
  Index Cond: (customer_id = 42)
  Filter: (status = 'PAID'::text)
```

### Cost model intuition
`cost ≈ index_descent + matching_index_entries * cpu_index_tuple_cost + matching_rows * random_page_cost` (heap fetches dominate).

---

## 4. Index Only Scan

### What it does
Like an Index Scan, but **does not visit the heap** when the index entry alone can answer the query and the page is marked all-visible.

### Internal mechanics
- Requires that **all referenced columns** are in the index (covered query).
- Uses the **Visibility Map**:
  - VM bit set → tuple is visible to every snapshot → return data straight from the index.
  - VM bit unset → must do a "heap fetch" to verify visibility (defeats the optimization).
- B-tree `INCLUDE` columns (Postgres 11+) let you add non-key payload columns purely for covering: `CREATE INDEX ... ON t (a) INCLUDE (b, c);`.
- Vacuum maintains the VM; tables with frequent updates may have low all-visible coverage.

### When the planner picks it
- Read-mostly tables that are vacuumed regularly.
- Aggregates over indexed columns (`SELECT count(*) FROM t WHERE indexed_col > x`).
- Covering queries on hot tables.

### EXPLAIN example
```
Index Only Scan using orders_status_amount_idx on orders
  (cost=0.42..120.40 rows=4000 width=12)
  Index Cond: (status = 'PAID'::text)
  Heap Fetches: 12
```
> `Heap Fetches: 0` is the dream — every row served from the index.

---

## 5. Bitmap Index Scan + Bitmap Heap Scan

### What they do
A two-phase strategy:
1. **Bitmap Index Scan** walks one (or several) indexes and builds an in-memory **bitmap of CTIDs** to read.
2. **Bitmap Heap Scan** reads those heap pages **in physical (block) order**, turning random I/O into mostly sequential I/O.

### Internal mechanics
- Implemented in `nodeBitmapIndexscan.c` and `nodeBitmapHeapscan.c`.
- Bitmap is page-granular when it's small, then can be **lossy** (only "this page contains a match") when the bitmap exceeds `work_mem`. Lossy pages require a re-check of every tuple on the page.
- Multiple bitmaps can be combined with **BitmapAnd** / **BitmapOr** nodes — the planner can use several indexes for one table in one scan.
- Always reports `Recheck Cond` — needed for lossy pages and for index conditions that aren't fully evaluable in the index AM (e.g., GIN approximate matches).

### When the planner picks it
- Selectivity in the "medium" zone — too many rows for an Index Scan's random I/O, too few for a Seq Scan.
- Multiple indexable predicates that can be `AND`/`OR`-combined.
- GIN indexes (full-text search, JSONB containment) almost always feed a bitmap scan.

### EXPLAIN example
```
Bitmap Heap Scan on orders  (cost=120.45..3450.10 rows=15000 width=72)
  Recheck Cond: ((status = 'PAID') AND (created_at > '2026-01-01'))
  Heap Blocks: exact=2350 lossy=0
  ->  BitmapAnd  (cost=120.45..120.45 rows=15000 width=0)
        ->  Bitmap Index Scan on orders_status_idx  ...
        ->  Bitmap Index Scan on orders_created_at_idx  ...
```

### Trade-offs
- Loses index ordering (output is in heap-page order) → cannot satisfy `ORDER BY`.
- Excellent for analytics-style filters; poor when only one row is needed.

---

## 6. TID Scan

### What it does
Retrieves tuples by their physical address (`ctid`) — the cheapest possible lookup: zero index walk, one page read.

### Internal mechanics
- Implemented in `nodeTidscan.c`.
- Triggered by predicates of the form `WHERE ctid = '(0,5)'` or `ctid = ANY(...)`.
- Mostly used internally by features like `SELECT ... FOR UPDATE` re-fetches, `MERGE`, and replication, but is also exposed to users.

### EXPLAIN example
```
Tid Scan on orders  (cost=0.00..4.01 rows=1 width=72)
  TID Cond: (ctid = '(123,4)'::tid)
```

### Caveats
- CTIDs change after `VACUUM FULL`, `CLUSTER`, or non-HOT updates — never persist them.

---

## 7. TID Range Scan

### What it does
Added in **PostgreSQL 14**. Reads a contiguous **range** of CTIDs, e.g. `WHERE ctid >= '(0,0)' AND ctid < '(1000,0)'`.

### Why it exists
Lets tools (and clever applications) read a table in chunks for parallel ETL, sampling, or reindex jobs without acquiring a full table scan and without an index.

### EXPLAIN example
```
Tid Range Scan on big_table  (cost=0.01..120.00 rows=10000 width=64)
  TID Cond: ((ctid >= '(0,0)'::tid) AND (ctid < '(1000,0)'::tid))
```

---

## 8. Sample Scan (TABLESAMPLE)

### What it does
Implements the SQL standard `TABLESAMPLE` clause for statistical sampling.

### Built-in methods
| Method | Behavior |
|---|---|
| `BERNOULLI(p)` | Each row is independently selected with probability `p%`; still scans the whole table |
| `SYSTEM(p)` | Each block is selected with probability `p%`; cheaper but biased toward block-level clustering |
| `SYSTEM_ROWS(n)` | (extension `tsm_system_rows`) Approximately `n` rows, faster |
| `SYSTEM_TIME(ms)` | (extension `tsm_system_time`) Sample for ~`ms` milliseconds |

### Internal mechanics
- Implemented in `nodeSamplescan.c` with a pluggable `TsmRoutine`.
- Optional `REPEATABLE (seed)` makes the sample deterministic.

### EXPLAIN example
```
Sample Scan on orders  (cost=0.00..1860.00 rows=10000 width=72)
  Sampling: bernoulli ('1'::real)
```

---

## 9. Function Scan

### What it does
Treats a set-returning function (SRF) as if it were a table. Used for `generate_series`, `unnest`, PL/pgSQL functions returning `SETOF record`, etc.

### Internal mechanics
- Implemented in `nodeFunctionscan.c`.
- For materializing functions, the executor calls the function once and stores results in a tuplestore, then iterates.
- For `RETURNS SETOF` with `ValuePerCall` mode, rows are pulled one at a time without materialization.
- `WITH ORDINALITY` adds an `int8` row-number column.

### EXPLAIN example
```
Function Scan on generate_series s  (cost=0.00..10.00 rows=1000 width=4)
```

---

## 10. Values Scan

### What it does
Iterates over an inline `VALUES (...), (...)` list as if it were a table.

### Internal mechanics
- Implemented in `nodeValuesscan.c`.
- Each row is built from a constant expression list — evaluated lazily.
- Often appears under `INSERT ... VALUES` and inside join queries that pivot a small constant set.

### EXPLAIN example
```
Values Scan on "*VALUES*"  (cost=0.00..0.04 rows=3 width=36)
```

---

## 11. CTE Scan

### What it does
Reads from a non-inlined Common Table Expression (the `WITH foo AS (...)` you see in `EXPLAIN`).

### Internal mechanics
- Implemented in `nodeCtescan.c`.
- Pre-PostgreSQL 12, **all CTEs were optimization fences** — materialized into a tuplestore once and re-read.
- Postgres 12+ inlines CTEs by default unless they are `MATERIALIZED`, recursive, or referenced multiple times. When materialized, you'll see a `CTE` subplan and `CTE Scan` consumers.
- Useful when you want the planner to compute something exactly once, e.g. an expensive aggregate joined back to the base table.

### EXPLAIN example
```
CTE Scan on top_customers  (cost=0.00..200.00 rows=100 width=44)
```

---

## 12. Subquery Scan

### What it does
A wrapper around a sub-plan whose output cannot be flattened into the outer query — typically because of `LIMIT`, `DISTINCT`, set operations, or aggregation in the subselect.

### Internal mechanics
- Implemented in `nodeSubqueryscan.c`.
- Mostly a passthrough that adds projection / column re-mapping; it doesn't materialize.

### EXPLAIN example
```
Subquery Scan on s  (cost=10.00..20.00 rows=100 width=8)
  ->  Limit  ...
```

---

## 13. WorkTable Scan (Recursive CTE)

### What it does
Used by `WITH RECURSIVE` to read the **working table** that holds rows produced by the previous iteration.

### Internal mechanics
- Implemented in `nodeWorktablescan.c`.
- Sits inside a `Recursive Union` plan node alongside a `CTE Scan` of the seed (non-recursive) branch.
- Each iteration reads its predecessor's output, runs the recursive term, appends new rows to the result, and repeats until no new rows appear.

### EXPLAIN example
```
Recursive Union  (cost=0.00..120.00 rows=200 width=8)
  ->  Seq Scan on departments d   -- non-recursive term
  ->  Hash Join  (...)
        ->  WorkTable Scan on tree  (cost=0.00..0.20 rows=10 width=8)
        ->  ...
```

---

## 14. Named Tuplestore Scan

### What it does
Reads from a transition table inside a trigger (`REFERENCING OLD TABLE AS old / NEW TABLE AS new`).

### Internal mechanics
- Implemented in `nodeNamedtuplestorescan.c`.
- The transition table is a per-statement tuplestore populated by the executor; the trigger function scans it like a regular relation.

---

## 15. Result Node

### What it is
Not strictly a "scan", but it's a leaf executor node worth knowing.

- Returns a single row of constants (e.g., `SELECT 1+1`).
- Wraps a child plan with an extra one-time filter (`Result` with `One-Time Filter: false` short-circuits the plan to zero rows).
- Implements `INSERT ... DEFAULT VALUES`.

### EXPLAIN example
```
Result  (cost=0.00..0.01 rows=1 width=4)
  One-Time Filter: false
```

---

## 16. Foreign Scan (FDW)

### What it does
Reads from a **foreign table** via a Foreign Data Wrapper — `postgres_fdw`, `file_fdw`, `mysql_fdw`, `clickhouse_fdw`, etc.

### Internal mechanics
- Implemented in `nodeForeignscan.c`, but the actual fetch logic lives in the FDW.
- Three FDW callbacks govern execution: `BeginForeignScan`, `IterateForeignScan`, `EndForeignScan`.
- **Pushdown** is the key optimization — the FDW can push WHERE clauses, projections, joins (PG 9.6+), aggregates (PG 10+), `LIMIT`, `ORDER BY`, even DML to the remote side. Whatever is pushed shows up as `Remote SQL` in `EXPLAIN VERBOSE`.
- `use_remote_estimate` toggles whether costs come from a remote `EXPLAIN` or from local heuristics.

### EXPLAIN example
```
Foreign Scan on remote_orders  (cost=100.00..200.00 rows=1000 width=72)
  Remote SQL: SELECT id, total FROM orders WHERE status = 'PAID'
```

---

## 17. Custom Scan

### What it does
An extension hook (`CustomScan` / `CustomScanMethods`) that lets third-party code provide its own executor node.

### Real-world examples
- **Citus** — distributes scans across worker shards.
- **TimescaleDB** — `ChunkAppend`, `DecompressChunk` for hypertables.
- **pg_strom** — GPU-accelerated scans/joins.

The planner sees Custom Scan paths during `set_rel_pathlist_hook` and can choose them in cost competition with built-in paths.

---

## 18. Parallel Scans

PostgreSQL 9.6+ supports **intra-query parallelism**. Each worker is a separate backend process; a `Gather` (or `Gather Merge`) node above the parallel scan collects results.

### Parallel-aware scan variants

| Variant | Notes |
|---|---|
| **Parallel Seq Scan** | Workers cooperatively claim block ranges via shared scan state. Default block-chunking grows as the scan progresses. |
| **Parallel Index Scan** | B-tree only; workers descend the same tree and split the leaf-level walk. |
| **Parallel Index Only Scan** | Same idea, with VM-driven heap-skip. |
| **Parallel Bitmap Heap Scan** | The bitmap is built **serially** by one leader, then scanned in parallel by workers (the bitmap lives in shared memory). |

### GUCs that govern parallelism
- `max_worker_processes` — global worker pool.
- `max_parallel_workers` — cap of pool used for queries.
- `max_parallel_workers_per_gather` — per-Gather cap (default 2).
- `min_parallel_table_scan_size`, `min_parallel_index_scan_size` — tables/indexes smaller than this won't go parallel.
- `parallel_setup_cost`, `parallel_tuple_cost` — costing knobs.

### EXPLAIN example
```
Gather  (cost=1000.00..12000.00 rows=10000 width=72)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on big_orders  (cost=0.00..11000.00 rows=4167 width=72)
        Filter: (status = 'PAID'::text)
```

---

## 19. Append, Merge Append & Partition Pruning

For **partitioned tables** and **table inheritance**, the planner emits one scan per child relation and combines them.

| Node | Behavior |
|---|---|
| **Append** | Concatenates child outputs in arbitrary order. |
| **Merge Append** | Children are individually sorted on the same key; node merges them (preserves `ORDER BY`). |
| **Parallel Append** (PG 11+) | Workers pick child plans dynamically — useful when child scans have very different costs. |

### Partition pruning
- **Plan-time pruning** happens during planning when the predicate references constants.
- **Run-time pruning** (PG 11+) happens at executor start (parameterized queries) or per-loop (nested-loop joins) — `EXPLAIN ANALYZE` reports `Subplans Removed: N`.

### EXPLAIN example
```
Append  (cost=0.00..3500.00 rows=15000 width=72)
  Subplans Removed: 11
  ->  Seq Scan on orders_2026_q1
  ->  Seq Scan on orders_2026_q2
```

---

## 20. How the Planner Chooses a Scan

### Path generation
For each base relation, `set_rel_pathlist` builds candidate `Path` structs:
1. `create_seqscan_path` → always added.
2. For each usable index → `create_index_path` (Index / Index Only) and `create_bitmap_heap_path`.
3. `create_tidscan_path` if `ctid` predicates exist.
4. Parallel variants if size thresholds allow.
5. Foreign / Custom paths via hooks.

`add_path` keeps only paths that are not dominated on **(cost, sort order, parallel-safety, parameterization)**.

### Cost formula highlights

```
Seq Scan:
    startup_cost = 0
    total_cost   = relpages * seq_page_cost
                 + reltuples * (cpu_tuple_cost + qual_cost)

Index Scan (random heap I/O dominates):
    total_cost  ≈ index_descent_cost
                + selectivity * reltuples * (cpu_tuple_cost + cpu_index_tuple_cost)
                + selectivity * reltuples * random_page_cost     -- heap fetches

Bitmap Heap Scan:
    + cost of underlying bitmap index scans
    + heap I/O modeled as a mix of seq + random based on selectivity
```

`effective_cache_size` tells the planner how much of the heap is likely cached, lowering the effective random page cost.

### Selectivity estimation
- `pg_statistic` stores **MCVs** (most-common values), **histogram bounds**, **null fraction**, **n_distinct**, and per-column correlation.
- Multi-column correlations come from **extended statistics** (`CREATE STATISTICS` — `ndistinct`, `dependencies`, `mcv`).
- `ANALYZE` (and autovacuum) populate these.

### Levers when the planner picks the wrong scan
| Symptom | Lever |
|---|---|
| Picks Seq Scan when index would win | `ANALYZE`; lower `random_page_cost`; raise `effective_cache_size`; add covering index |
| Picks Index Scan that fetches huge heap | Encourage Bitmap with multi-column predicates; check selectivity stats |
| Won't go parallel | Raise `max_parallel_workers_per_gather`; lower `min_parallel_table_scan_size`; check `parallel_safe` functions |
| Bad row estimates | `CREATE STATISTICS`; raise `default_statistics_target`; rewrite predicates |

GUCs `enable_seqscan`, `enable_indexscan`, `enable_indexonlyscan`, `enable_bitmapscan`, `enable_tidscan` etc. don't actually disable a path — they add a huge penalty (`disable_cost = 1.0e10`). Useful for diagnosis, not production.

---

## 21. Reading EXPLAIN — Field Reference

```
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, WAL) <query>;
```

| Field | Meaning |
|---|---|
| `cost=startup..total` | Planner's abstract units (not ms). Startup = cost before first row. |
| `rows` | **Estimated** row count. With `ANALYZE`, also shows actual. |
| `width` | Average row size in bytes. |
| `actual time` | Wall-clock per loop, in ms. |
| `loops` | Times this node was executed (e.g. inner side of a nested loop). |
| `Buffers: shared hit/read/dirtied/written` | Page-level I/O accounting. `read` = from OS / disk; `hit` = from buffer cache. |
| `Heap Fetches` | Index Only Scan tuples that needed a heap visit (VM bit unset). |
| `Heap Blocks: exact / lossy` | Bitmap Heap Scan precision; lossy ⇒ rechecks every tuple on those pages. |
| `Rows Removed by Filter` | Tuples read but discarded — high values suggest a missing index. |
| `Workers Planned/Launched` | Parallel execution status. |

### Common diagnostic patterns
- **`actual rows` ≫ estimate** → bad statistics or correlated columns; consider `CREATE STATISTICS`.
- **High `Buffers: shared read`** → cold cache or scan touching too many pages.
- **`Heap Fetches` > 0** on Index Only Scan → run `VACUUM` to refresh the visibility map.
- **`Heap Blocks: lossy=...`** → `work_mem` is too small for the bitmap; increase it.

---

## 22. Quick-Reference Summary Table

| Scan Node | Source | Output Order | I/O Pattern | Best For |
|---|---|---|---|---|
| Seq Scan | Heap | Heap order | Sequential | Small tables / non-selective predicates |
| Index Scan | Index → Heap | Index order | Random heap | Highly selective predicates, ORDER BY + LIMIT |
| Index Only Scan | Index (+ VM) | Index order | Mostly index pages | Covered queries on vacuumed tables |
| Bitmap Index + Heap Scan | Index(es) → bitmap → Heap | Heap-block order | Mostly sequential heap | Mid-selectivity, multi-index AND/OR, GIN |
| TID Scan | Heap by CTID | N/A | One page | Internal re-fetches, debugging |
| TID Range Scan | Heap by CTID range | Heap order | Sequential | Chunked ETL, parallel readers (PG 14+) |
| Sample Scan | Heap | Sample order | Varies by method | Statistical sampling |
| Function Scan | SRF | Function order | None | `generate_series`, `unnest`, PL functions |
| Values Scan | Inline VALUES | Listed order | None | Small constant relations |
| CTE Scan | Materialized CTE tuplestore | Producer order | None | Optimization fences, multi-reference CTEs |
| Subquery Scan | Sub-plan | Sub-plan order | None | Non-flattenable subselects |
| WorkTable Scan | Recursive CTE working table | Iteration order | None | `WITH RECURSIVE` |
| Named Tuplestore Scan | Trigger transition table | Trigger order | None | Statement-level triggers |
| Result | Constants / one-time filter | N/A | None | `SELECT 1`, contradiction short-circuit |
| Foreign Scan | Remote via FDW | Remote-defined | Network | External data, federated queries |
| Custom Scan | Extension | Extension-defined | Extension | Citus, TimescaleDB, pg_strom |
| Parallel * | Same as base, in workers | Possibly unordered | Parallel I/O | Large analytical scans |
| Append / Merge Append | Multiple children | Concatenated / merge-sorted | Per child | Partitioned tables, inheritance |

---

## 23. Video Notes — Bitmap Index Scan in Postgres

> Distilled notes from a walkthrough video on how Postgres picks between Sequential, Index, and Bitmap Index Scans, with a focus on **why the Bitmap Index Scan exists** as a middle-ground strategy.

### Key Concepts

#### Sequential Scan (Full Table Scan)
- Reads **every row** in the table from disk into memory.
- Efficient when the query needs to touch a **large portion** of the table — there is no point doing index lookups if you'll read most pages anyway.
- Avoids random I/O entirely; everything is sequential reads, which the OS/storage handles efficiently.

#### Index Scan
- Uses an index (e.g., a B-tree on the primary key) to **locate exactly the rows that match**.
- Best for **small, targeted queries** where the predicate is highly selective.
- **Downside:** each matched index entry triggers a **random heap fetch** to read the actual row. When many rows match, the cost of all those random page reads quickly outweighs the cost of just scanning the table sequentially.

#### Bitmap Index Scan (Postgres' middle-ground strategy)
- A two-phase plan unique in spirit to Postgres:
  1. **Phase 1 — Build the bitmap.** Walk the index and, instead of fetching rows immediately, mark the **heap pages** (and within them, the tuples) that contain matches. The result is an in-memory bitmap of "interesting" pages.
  2. **Phase 2 — Read the heap in physical order.** Visit each marked page **once, in block-number order**, and re-check the conditions on the tuples there.
- Random I/O is converted into mostly **sequential I/O** because the heap is read in page order rather than in index order.
- **Combines multiple indexes.** For a query like `WHERE grade > 95 AND id < 10000`, Postgres can:
  - Build a bitmap from the `grade` index,
  - Build a bitmap from the `id` index,
  - **AND** them together (`BitmapAnd`),
  - Then read only the heap pages that survive the intersection.
- Conditions are **rechecked** after fetching, both because (a) the bitmap may be lossy and (b) the indexes may have approximated the predicate.

### Strategy Insights — How Postgres Chooses

| Number of matching rows | Scan Postgres tends to pick | Reason |
|---|---|---|
| **Very few** (highly selective) | **Index Scan** | A handful of random heap fetches is cheap; index ordering may also help `LIMIT`/`ORDER BY`. |
| **Many** (low selectivity) | **Sequential Scan** | If you're reading most of the table anyway, skip the index overhead and stream the heap. |
| **Moderate** (in between) | **Bitmap Index Scan** | Index narrows down the page set, then heap is read in physical order — best of both worlds. |

### Performance Trade-offs at a Glance

- **Sequential Scan** — no random I/O, but reads everything (wasted work on selective predicates).
- **Index Scan** — precise, but each row triggers a random page read; degrades quickly as match-count grows.
- **Bitmap Scan** — precise *and* batches page access; pays the cost of building the bitmap up front, then amortizes by reading heap pages once and in order.

### Practical Use Cases for Bitmap Scans

- Queries with **multiple WHERE conditions** that each have an index — Postgres can `BitmapAnd` / `BitmapOr` them.
- **GIN-indexed** queries (full-text search, JSONB containment, trigram search) — the GIN AM almost always feeds a bitmap scan.
- Analytics-style filters that match thousands to millions of rows out of a much larger table — too many for an Index Scan, too few for a Seq Scan.

### Decision Flow — Postgres' Internal Reasoning

```
                ┌──────────────────────────────┐
                │ Predicate on a relation      │
                └──────────────┬───────────────┘
                               │
              estimate selectivity from pg_statistic
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   very selective         moderate                low selectivity
   (few rows)             (many but not all)      (most of table)
        │                      │                      │
        ▼                      ▼                      ▼
   ┌──────────┐         ┌──────────────┐         ┌──────────┐
   │  Index   │         │   Bitmap     │         │   Seq    │
   │  Scan    │         │   Index +    │         │   Scan   │
   │          │         │   Heap Scan  │         │          │
   └──────────┘         └──────────────┘         └──────────┘
   random heap          build bitmap →           stream heap
   fetches per row      heap read in             pages in order
                        block order
```

### Takeaway

> Bitmap Index Scans are Postgres' clever optimization: they **batch work to reduce random access**. Use them when a query returns a moderate number of rows or combines multiple indexable conditions — Postgres will merge the bitmaps, walk the heap once in physical order, and recheck the predicate on the way through.

---

## Further Reading

- PostgreSQL source: `src/backend/executor/node*scan.c`
- PostgreSQL docs: *Chapter 14 — Performance Tips*, *Chapter 70 — Index Access Method Interface*
- `EXPLAIN` reference: <https://www.postgresql.org/docs/current/sql-explain.html>
- *PostgreSQL Internals* by Egor Rogov (free book, Postgres Pro)
- Bruce Momjian's "Inside PostgreSQL Optimizer" talks
