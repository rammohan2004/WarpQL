# WarpQL — build plan (as fast as possible)

No calendar. Steps run back to back: the next step starts as soon as `/verify-step` passes on the current one. The only fixed date is the course submission deadline, **Sat 7 November 2026** (code zip or public GitHub link on Moodle; presentation during end-sems).

Two hard gates:
- **Gate A (Step 6):** all 13 SSB queries run on the GPU and match DuckDB.
- **Gate B (Step 7):** batched INSERT/UPDATE/DELETE on the GPU matches the serial oracle on every conflict test, and all 13 queries still match DuckDB after writes.

No optimisation work (Steps 9–10) starts before both gates pass.

Each step lists what to build, the concepts, and how we know it is done. Claude Code explains every step before doing it (see `CLAUDE.md`). The write-path design is in `docs/write-path-design.md`.

---

## Step 0 — Environment and hardware

**Build:** WSL2 + CUDA toolkit + build tools + Claude Code working; GPU identified; SSB data generated at SF1; git repo initialised (plan for a public GitHub repo for submission).

**How:** follow `docs/environment-setup-wsl2.md`, then run `/hardware-check`, which records GPU model, compute capability, memory, chosen scale factors and table capacity headroom in `docs/hardware.md`.

**Done when:**
- [ ] `nvidia-smi` and `nvcc --version` work inside WSL
- [ ] A trivial CUDA program compiles and runs (Claude writes it as the first explained step)
- [ ] `docs/hardware.md` filled in, including max scale factor and lineorder capacity headroom
- [ ] SSB SF1 generated, loaded into DuckDB, exported to binary columns
- [ ] Max `lo_orderkey` and `lo_linenumber` checked; primary-key packing into `uint64` confirmed
- [ ] Git repo with first commit

## Step 1 — Project skeleton, interfaces and write-ready storage

**Build:** CMake project (C++ + CUDA + OpenMP + GoogleTest), error-check macro, RAII device buffer, memory pool, GPU timer, shared host/device hash function, `Column` / `Table` / `Batch` / `Operator` interfaces, binary column loader.

Write-ready from day one (see `docs/write-path-design.md` §3):
- columns have `capacity` and `n_rows`;
- each table has a **validity bitmap** (all live after load);
- primary-key packing helper for `(lo_orderkey, lo_linenumber)`.

**Concepts:** RAII and move-only types, `__host__ __device__` functions, CUDA error handling, CUDA events, pool allocation, bitmaps.

**Done when:**
- [ ] `cmake --build` and `ctest` run
- [ ] Loader reads SF1 columns; row counts match DuckDB; validity bitmap all set
- [ ] Unit test for primary-key packing/unpacking

## Step 2 — CPU reference engine + DuckDB harness

**Build:** single-threaded CPU operators (filter honouring validity, hash build/probe, hash aggregation, sort, top-k), all 13 SSB query plans, then OpenMP versions. A harness that runs the SQL in DuckDB and compares results exactly.

**Concepts:** open addressing, load factor, star-schema joins, morsel-style chunking, thread-local aggregation, `std::atomic` CAS.

**Done when:**
- [ ] Single-threaded engine passes 13/13 at SF1
- [ ] OpenMP engine passes 13/13 at SF1 and the max laptop SF
- [ ] One command runs the whole comparison

## Step 3 — Filter, scan, compaction, gather

**Build:** predicate kernel producing a bitmap with `__ballot_sync`, **ANDed with the validity bitmap**; warp scan; block scan; three-phase device-wide compaction; gather. SSB flight 1 on the GPU.

**Concepts:** coalescing, warps and lanes, ballot/popc, shuffles, shared memory, `__syncthreads`, exclusive vs inclusive scan, constant memory.

**Done when:**
- [ ] Edge-size unit tests pass, including rows marked invalid; clean memcheck + racecheck
- [ ] Q1.1–Q1.3 match DuckDB on GPU

## Step 4 — Hash join

**Build:** open-addressing build with 64-bit `atomicCAS` slots; probe kernel; Q2.x on the GPU.

**Concepts:** atomics, lock-free insertion, linear probing, random access and L2 behaviour.

**Done when:**
- [ ] Build/probe tests pass for build sizes 1 → 10M
- [ ] Q2.1–Q2.3 match DuckDB on GPU

## Step 5 — Aggregation, sort, top-k

**Build:** dense-index aggregation with global atomics; shared-memory privatised aggregation; general hash aggregation; LSD radix sort (1-bit first, then 4-bit digits) supporting 64-bit keys with a payload (needed by the write path); block reduction; sort-based top-k. Q3.x and Q4.x on the GPU.

**Concepts:** atomic contention, privatisation, histograms, scatter, stable sort, reductions.

**Done when:**
- [ ] Aggregation and sort tests pass, including 64-bit keys
- [ ] Q3.1–Q4.3 match DuckDB on GPU

## Step 6 — GATE A: all 13 queries on GPU

**Build:** run the full suite at SF1 and the max laptop SF; record baseline timings (CPU, GPU kernel-only, GPU incl. host→device copy); tag `v1-reads`.

**Done when:**
- [ ] 13/13 match DuckDB on GPU at both scales
- [ ] `results/v1/` holds baseline CSVs
- [ ] Commit tagged `v1-reads`

## Step 7 — Write path: INSERT, UPDATE, DELETE (GATE B)

Follow `docs/write-path-design.md`. Use the `write-path` skill. Four sub-steps:

**7a — Statements, oracle, generator.**
- Batch format (structure-of-arrays: op, key, seq, column, value/delta, insert row values) and statement status codes.
- CPU **serial oracle** applying statements in `seq` order.
- Batch generator: size, operation mix, key skew (uniform/Zipf), hot-key conflict rate, deliberate duplicates; plus the fixed conflict-matrix cases (design §7).
- DuckDB applier that runs the same statements as SQL with a primary key, recording rejections.

**7b — Primary-key index on GPU.** Build from lineorder at load time (64-bit key claim with `atomicCAS`, parallel row-id array); probe kernel returning row id + liveness.

**7c — Segmented replay kernels (W1–W6).** Sort by (key, seq), segment heads + scan, probe, per-segment replay, insert-slot allocation with capacity check, apply/commit.

**7d — Queries after writes.** Run several batches, then all 13 SSB queries on GPU vs DuckDB with the same writes applied. Determinism check: same batch twice → identical results.

**Concepts:** serial equivalence, deterministic ordering, segmented operations, primary-key constraints, tombstones/validity bitmaps, columnar in-place updates, append with preallocated capacity, snapshot isolation by serialisation.

**Done when:**
- [ ] Every conflict-matrix test matches the serial oracle (statuses + full-table checksum)
- [ ] Random batches up to the largest size that fits match the oracle
- [ ] Capacity-overflow batch is rejected with the table unchanged
- [ ] 13/13 queries match DuckDB after writes
- [ ] Clean memcheck + racecheck on all write kernels
- [ ] Commit tagged `v2-writes`

## Step 8 — Profile

**Build:** measured peak bandwidth (copy kernel); effective bandwidth per kernel; Nsight Systems timeline of one query and one write batch; Nsight Compute on the most expensive kernels (if counters are available on WSL2).

**Done when:**
- [ ] Bandwidth table per kernel in `NOTES.md`
- [ ] Top bottlenecks for reads and for writes named, with evidence

## Step 9 — Optimise

Pick at least four, measured one at a time with `/optimise`, including at least one on the write path:
- **atomic fast path for update-only / insert-only batches vs segmented replay** (design §5), across batch size and conflict rate
- items per thread (grid-stride, 4–8 rows per thread)
- dense-key direct-index join for SSB dimensions
- small dimension table copied to shared memory
- aggregation strategy sweep (global vs privatised vs warp-aggregated atomics)
- radix digit width 4 vs 8 bits
- block size 128 / 256 / 512
- faster top-k (block bitonic + merge)

**Done when:**
- [ ] ≥ 4 optimisations logged with before/after numbers and evidence (≥ 1 on writes)
- [ ] All queries still match DuckDB; all write tests still match the oracle

## Step 10 — Fused tile kernels (can overlap with Step 9)

**Build:** one fused kernel per query for flight 1 (and flight 2 if time), keeping tiles in registers and **skipping invalid rows**; compare against operator-at-a-time on runtime and DRAM bytes; check registers and occupancy. Read Crystal only after ours works.

**Done when:**
- [ ] Fused Q1.x correct (also after write batches) and compared; tag `v3-optimised`

## Step 11 — Evaluation

**Build:** benchmark sweeps per `docs/benchmarking-protocol.md`, including the write experiments; plots from one CSV format. Optional: largest scale factors on the Param Shakti A100 if access is available.

**Done when:**
- [ ] All CSVs + charts in `results/final/`
- [ ] Largest-scale read and write results verified (DuckDB / serial oracle)

## Step 12 — Demo, report, slides

**Build:** CLI demo that runs a query, applies a conflicting write batch (showing per-statement statuses), and reruns the query with a timing breakdown; report with a dedicated section answering the professor's concurrency question; ~12–14 slides; README; public GitHub repo; two rehearsals (each presents the part they understand less well). Include the AI-assistance disclosure required by the course.

**Done when:**
- [ ] Report, slides and README complete; repository public (or code zip ready)
- [ ] Submitted on Moodle (deadline Nov 7)

---

## If we fall behind

Cut in this order: fused flight 2 → faster top-k → radix digit-width study → fused flight 1 → warp-aggregated atomics → atomic write fast path (keep it as a described design). Never cut: CPU baseline, DuckDB checks, the serial-oracle conflict tests, the segmented-replay write path, before/after measurements.

## Stretch goals (only after everything above)

- Multi-statement transactions with all-or-nothing abort
- Writes to dimension tables; foreign-key checks on insert
- Vacuum: compact deleted rows and rebuild the index (reuses compaction)
- Radix-partitioned join; decoupled look-back scan; TPC-H Q6

## Laptop-specific notes

- Keep the laptop plugged in and set Windows power mode to best performance for every timed run; laptop GPUs throttle. Record this in the report.
- `nvidia-smi` under WSL2 cannot report GPU utilisation; rely on CUDA events and Nsight.
- Pinned memory is limited under WSL2; keep transfer experiments small.
- Table capacity headroom uses extra GPU memory; size it with the scale factor in `docs/hardware.md`.
