# WarpQL — project overview

**WarpQL: A GPU-Native Columnar Query Engine for Analytical Workloads**

A column-store query engine whose filter, join, aggregation and sort operators are hand-written CUDA kernels, with batched INSERT, UPDATE and DELETE applied on the GPU under a serial-equivalence guarantee. Evaluated against a multi-core CPU engine on Star Schema Benchmark data, with every query result checked against DuckDB and every write batch checked against a serial oracle.

## Why this problem suits a GPU

Analytical queries scan millions of rows, filter, join to small dimension tables, and aggregate. Per row the work is a few comparisons and a hash lookup, so speed is limited by **memory bandwidth** and **random-access latency**, not arithmetic. GPUs have much higher memory bandwidth than CPUs and hide latency with thousands of resident threads. The catch: if data must be copied over PCIe for every query, the copy can cost more than the computation. We measure both cases (data already on the GPU, and data copied per query).

## Architecture

```
CLI / benchmark driver
        |
Plan builder (CPU)             -- an operator tree per SSB query; no SQL parser
Write batch builder (CPU)      -- INSERT/UPDATE/DELETE statements with sequence numbers
        |
Executor + device memory pool  -- runs queries and write batches one after another on one stream
        |                \
GPU engine (our kernels)  CPU engine (C++/OpenMP, same algorithms)
        |
GPU global memory: columns (with capacity), validity bitmaps, primary-key index,
                   hash tables, intermediates, aggregates
        ^
Storage layer (CPU): dbgen .tbl -> DuckDB -> binary column files
```

Correctness: every query result is compared exactly with DuckDB; every write batch is compared with a serial CPU oracle (and with DuckDB applying the same SQL statements). Benchmarks write one CSV format; Python scripts plot it.

## Data model

- Structure-of-arrays: each column is its own array, so a warp reading 32 consecutive rows of one column makes one coalesced access.
- Strings are dictionary-encoded with **sorted** dictionaries, so range predicates on strings become integer range checks.
- Decimals are scaled integers; sums use 64-bit accumulators, so results are exact.
- Dates are `yyyymmdd` int32, so year filters become integer ranges.
- Every table has a **validity bitmap** (1 bit per row slot); every scan ANDs with it, so deleted rows are skipped.
- `lineorder` columns are allocated with spare **capacity** for inserts; a GPU **primary-key index** maps `(lo_orderkey, lo_linenumber)` to a row id.

## Kernels we will write

| ID | Kernel | Idea | Key concepts |
| --- | --- | --- | --- |
| K1 | filter | one thread per row, branch-free predicate, warp results packed with `__ballot_sync` into a bitmap | coalescing, constant memory, warps |
| K2 | compaction | count per block, scan of block counts, scatter at computed offsets | warp/block/device scan, `__popc`, shuffles |
| K3 | gather | `out[j] = col[sel[j]]` | late materialisation |
| K4 | hash build | open addressing, 64-bit slot (key, value) inserted with one `atomicCAS` | atomics, lock-free insert |
| K5 | hash probe | walk slots until match or empty | random access, L2 hit rate |
| K7 | aggregation | global atomics vs shared-memory privatised tables vs warp-aggregated atomics; general hash version | atomic contention |
| K8 | radix sort | LSD: per-block histograms, scan, stable scatter | histograms, scan, scatter |
| K9 | top-k | sort-then-take first; block bitonic + merge later | bitonic sort, shared memory |
| K10 | reduction | warp shuffle, then shared memory | reduction trees |
| K11 | fused tile kernels | filter + probe + aggregate one tile of rows in registers per block, skipping invalid rows | kernel fusion, register pressure, occupancy |

### Write-path kernels (details in `docs/write-path-design.md`)

| ID | Kernel | Idea | Key concepts |
| --- | --- | --- | --- |
| W0 | primary-key index build/probe | 64-bit key claimed with `atomicCAS`, row id in a parallel array | atomics, phased writes across kernel boundaries |
| W1 | sort statements | radix sort by (key, seq) so each key's statements are contiguous and ordered | stable sort, composite keys |
| W2 | segment heads | adjacent-difference + scan to find each key's segment | segmented scan |
| W3 | probe | does the key have a live row, and where? | random access |
| W4 | replay | one thread per key replays its statements in seq order; writes statuses and the final action | serial equivalence, divergence on hot keys |
| W5 | insert slots | scan over insert flags gives new row positions; capacity check | scan, all-or-nothing batch |
| W6 | apply | write columns in place, append inserts, flip validity bits, update index | one writer per key, no locks |
| W7 | atomic fast path | `atomicMax` on (seq, index) for SET updates, `atomicAdd` for deltas, `atomicCAS`+`atomicMin` for inserts | contention vs sorting cost |

(K6, a radix-partitioned join, is a stretch goal only.)

## The four SSB query flights

| Flight | Queries | Dimension joins | Work |
| --- | --- | --- | --- |
| 1 | Q1.1–Q1.3 | date | filters on date, discount, quantity; one SUM |
| 2 | Q2.1–Q2.3 | part, supplier, date | group by year, brand; order by them |
| 3 | Q3.1–Q3.4 | customer, supplier, date | group by nation/city, year; order by year, revenue desc |
| 4 | Q4.1–Q4.3 | customer, supplier, part, date | profit = revenue − supply cost, grouped |

Query text: https://clickhouse.com/docs/get-started/sample-datasets/star-schema (also in the SSB paper).

## Concurrency answer (the professor's question)

Writes are batched; each statement has a sequence number. The GPU result is defined to equal applying statements one by one in sequence order, so two writes to the same row resolve in sequence order (latest SET wins; deltas add up), and of two inserts with the same key the earlier succeeds and the later is rejected. Queries never overlap a batch, so each sees a consistent snapshot. Full design: `docs/write-path-design.md`.

## Main experiments

1. GPU vs CPU per query, with and without host→device copy, across scale factors.
2. Filter cost vs selectivity.
3. Join probe cost vs build size (shared memory → L2 → DRAM).
4. Aggregation strategy vs number of groups and skew (where each strategy wins).
5. Our radix sort vs CUB vs CPU.
6. Fused tile kernels vs operator-at-a-time: runtime, bytes moved, occupancy.
7. Write throughput (statements/s) vs batch size, operation mix and key skew; segmented replay vs atomic fast path vs OpenMP CPU.
8. Query slowdown from deleted rows (validity checks) as the deleted fraction grows.

## Relation to prior work

The fused tile-based model follows Crystal: Shanbhag, Madden, Yu, "A Study of the Fundamental Performance Characteristics of GPUs and CPUs for Database Analytics", SIGMOD 2020 (https://arxiv.org/abs/2003.01178). Our contribution is an independent implementation plus comparisons inside one engine on our hardware. We do not claim novelty.

## Scope

Must have: K1–K5, K7 (global + privatised), K8, K9, K10; W0–W6 with the full conflict test matrix; all 13 SSB queries on CPU and GPU, also after writes; DuckDB and serial-oracle checks; ≥ 4 measured optimisations.
Should have: W7 atomic fast path and its comparison; K11 for flight 1 (and 2); warp-aggregated atomics; shared-memory dimension tables; dense-key joins.
Stretch: multi-statement transactions, dimension-table writes, vacuum, radix-partitioned join, decoupled look-back scan, TPC-H Q6, A100 runs at large scale.
