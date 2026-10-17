# Write path design: INSERT, UPDATE, DELETE with conflict handling

This answers Prof. Nasre's question: *"How will you take care of two transactions writing to the same row or inserting with the same ID?"* It is part of the required scope, not a stretch goal.

## 1. Scope and model

- **Writes target the fact table `lineorder` only.** Dimension tables (`date`, `customer`, `supplier`, `part`) stay read-only. This mirrors a warehouse where new orders keep arriving while customers and parts change rarely. Dimension writes are a stretch goal.
- **Primary key:** `(lo_orderkey, lo_linenumber)`, packed into one `uint64` as `(orderkey << 8) | linenumber`. Record the bit widths in `docs/hardware.md` after checking the data's maximum orderkey.
- **Transaction = one write statement** (autocommit style), each with a unique **sequence number** `seq` that defines commit order. Multi-statement transactions with all-or-nothing abort are a stretch goal.
- **Statements supported:**
  - `INSERT` a full row (all lineorder columns).
  - `UPDATE` one row by primary key: `SET col = value` or `SET col = col + delta` on the updatable columns `lo_quantity`, `lo_extendedprice`, `lo_discount`, `lo_revenue`, `lo_supplycost`. Primary-key and foreign-key columns are not updatable (stated as a design rule).
  - `DELETE` one row by primary key.
- **Writes are applied in batches, between queries.** A batch is a list of statements with sequence numbers. The executor runs queries and batches one after another on the same CUDA stream, so they never overlap.

## 2. Correctness guarantee (what we promise and test)

> **Serial equivalence:** applying a batch on the GPU produces exactly the same table, and the same accept/reject outcome for every statement, as applying the statements one at a time in increasing `seq` order.

And for readers:

> **Snapshot reads:** every query sees the table either entirely before or entirely after each batch, never part of one.

This answers both of the professor's cases:

| Case | Serial outcome | How the GPU gets it |
| --- | --- | --- |
| Two UPDATEs of the same row | Both apply in `seq` order; final value is from the later `SET` (or both deltas added) | Statements on the same key are replayed in `seq` order |
| Two INSERTs with the same new key | Earliest `seq` succeeds; the later one is rejected `DUPLICATE_KEY` | Same: replay in `seq` order; second sees the key exists |
| INSERT of a key that already exists | Rejected `DUPLICATE_KEY` | Probe of the primary-key index finds a live row |
| UPDATE/DELETE of a missing key | Rejected `NOT_FOUND` | Probe finds nothing live |
| DELETE then INSERT of the same key | Both succeed; the new row is live | Replay handles any interleaving |
| UPDATE after DELETE of the same key | Rejected `NOT_FOUND` | Replay sees the row is gone |

The thread schedule can never change the result, because results depend only on `seq`.

## 3. Storage changes (designed in from Step 1)

- **Capacity and row count:** each lineorder column is allocated with headroom (`capacity` > `n_rows`) so inserts append without reallocating. Capacity is fixed per run (e.g. 1.25 × initial rows, recorded in `docs/hardware.md`). A batch whose inserts would exceed capacity is rejected before any change is made.
- **Validity bitmap:** 1 bit per row slot; 1 = live, 0 = deleted or unused. **Every scan honours it from the start**: the filter kernel ANDs its predicate bitmap with the validity word, and fused kernels skip invalid rows. Dimension tables have all bits set.
- **Primary-key index:** a GPU open-addressing hash table mapping the 64-bit key to a row id. Key slots are claimed with 64-bit `atomicCAS`; the row id lives in a parallel array written after the claim, read only after a kernel boundary. Sized for `capacity` at load factor ≤ 0.5, so it never needs resizing within a run. A deleted row keeps its index entry; liveness comes from the validity bitmap. When a deleted key is reinserted, its entry points to the new row.
- **Updates are in place** on the column arrays: a columnar update writes only the changed columns.

## 4. GPU algorithm: sort-based segmented replay (the general path, must-have)

Batch of `m` statements, as structure-of-arrays: `op`, `key`, `seq`, `col_id`, `value`/`delta`, plus row values for inserts.

1. **W1 — sort by (key, seq).** Reuse our radix sort on a composite key (or sort by `seq` then stably by `key`). Statements on the same key become a contiguous segment, ordered by `seq`.
2. **W2 — segment heads.** Mark where the key changes (adjacent-difference), then scan to get segment ids and boundaries. Reuses the scan/compaction kernels.
3. **W3 — probe index.** For each segment head, probe the primary-key index: is there a live row for this key, and which row id?
4. **W4 — replay.** One thread per segment replays its statements in `seq` order on a small local state (exists?, row id, current values of updatable columns), writing a status per statement (`OK`, `DUPLICATE_KEY`, `NOT_FOUND`) and a final action for the key: nothing, update row *r*, delete row *r*, insert new row, or delete *r* then insert. Segments are usually length 1–3, so the serial part is tiny; hot keys make long segments (that is an experiment, not a bug).
5. **W5 — allocate insert slots.** Flag segments whose final action inserts; exclusive scan gives each a new row slot `n_rows + offset`. Check capacity first; if exceeded, reject the batch with nothing changed.
6. **W6 — apply (commit).** Write updated columns in place; write inserted rows into their new slots; clear validity bits for deletes, set them for inserts; insert or redirect primary-key index entries. Each key is touched by exactly one thread, so no two threads write the same row.
7. **Update `n_rows`** on the host after W6 completes.

Every kernel is data-parallel across statements or keys; ordering within a key comes from the sort, not from locks.

## 5. GPU algorithm: atomic fast path (should-have, compared against §4)

For batches containing only UPDATEs or only INSERTs (the common streaming case), skip the sort:

- **Update-only, `SET col = value`:** each statement does `atomicMax` on a per-row 64-bit winner slot holding `(seq << 32) | statement_index`. After a kernel boundary, the winning statement per row writes its value. Same result as replay, because the largest `seq` wins.
- **Update-only, `SET col = col + delta`:** `atomicAdd` on the column. Addition commutes, so order doesn't matter.
- **Insert-only:** claim the key in the primary-key index with `atomicCAS`; if several statements claim the same new key, `atomicMin` on `seq` in a side array picks the earliest; losers get `DUPLICATE_KEY`.

Experiment: throughput of the fast path vs segmented replay as batch size and key skew (conflict rate) vary. Expect the fast path to win when conflicts are rare and to suffer contention on hot keys; measure, don't assume.

## 6. CPU baselines and oracles

- **Serial oracle (correctness):** a plain C++ loop applying statements one by one in `seq` order with `std::unordered_map<uint64_t, row_id>`. The GPU result must match it exactly: same statuses, same final table.
- **Parallel CPU baseline (speed):** OpenMP version of the same sort-based replay (parallel sort, then threads over segments).
- **DuckDB cross-check:** apply the same statements as SQL in `seq` order to a DuckDB copy of lineorder with a primary key; a duplicate insert raises a constraint error, which the harness records as `DUPLICATE_KEY`. Then rerun all 13 SSB queries on both engines and compare.

## 7. Tests (the conflict matrix)

Each runs on the GPU and is compared with the serial oracle (statuses + full-table checksum sorted by key):

1. Many UPDATEs (`SET`) of one hot row, random `seq` order in the input array.
2. Many `+delta` UPDATEs of one hot row.
3. Two to 1,000 INSERTs of the same new key.
4. INSERT of an existing key.
5. UPDATE and DELETE of missing keys.
6. DELETE → INSERT → UPDATE of the same key in one batch, all permutations of `seq`.
7. Batch that would exceed capacity → whole batch rejected, table unchanged.
8. Large random batches (1K → largest that fits) with uniform and Zipf-skewed keys, mixed operations.
9. After several batches: all 13 SSB queries match DuckDB with the same writes applied.
10. Same batch applied twice from the same starting state on GPU gives identical results (determinism).

## 8. Limits we state honestly in the report

- No concurrent execution of queries and writes; isolation comes from running them in sequence (a snapshot per batch boundary). True MVCC is out of scope.
- Single-statement transactions only (unless the stretch goal is done).
- No durability: data lives in memory; there is no write-ahead log or crash recovery.
- Writes only on the fact table; updatable columns are fixed.
- Deleted slots are not reclaimed until a compaction ("vacuum") pass, which is a stretch goal and reuses the compaction kernel.

## 9. Interview angle

This is deterministic, batch-based concurrency control: ordering is fixed up front (sequence numbers), so the parallel execution never has to lock. It's the same idea as deterministic database systems such as Calvin, adapted to a GPU. It also contrasts nicely with the lock-free CPU hash map on our resumes: there, CAS loops resolve races at run time; here, sorting removes the races before any write happens.
