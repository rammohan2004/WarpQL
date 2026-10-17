# Progress tracker

Claude Code reads this at the start of each session (`/start-session`) and updates it at the end (`/end-session`). Keep it short and current.

**Current step:** Step 0 — Environment and hardware
**Next action:** Follow `docs/environment-setup-wsl2.md`, then run `/hardware-check`
**Submission deadline:** Sat Nov 7, 2026 (no other dates; go as fast as possible)
**Gate A (reads, Step 6):** not reached
**Gate B (writes, Step 7):** not reached

## Steps

| Step | Status | Started | Finished | Notes |
| --- | --- | --- | --- | --- |
| 0 Environment + hardware | not started | | | |
| 1 Skeleton + write-ready storage | not started | | | |
| 2 CPU engine + DuckDB harness | not started | | | |
| 3 Filter, scan, compaction | not started | | | |
| 4 Hash join | not started | | | |
| 5 Aggregation, sort, top-k | not started | | | |
| 6 GATE A: 13/13 on GPU | not started | | | |
| 7a Statements, oracle, generator | not started | | | |
| 7b Primary-key index | not started | | | |
| 7c Segmented replay kernels | not started | | | |
| 7d Queries after writes (GATE B) | not started | | | |
| 8 Profile | not started | | | |
| 9 Optimise | not started | | | |
| 10 Fused kernels | not started | | | |
| 11 Evaluation | not started | | | |
| 12 Demo, report, slides | not started | | | |

Status values: not started · in progress · done · cut

## SSB query status

| Query | CPU vs DuckDB | GPU vs DuckDB | GPU after writes | Fused |
| --- | --- | --- | --- | --- |
| Q1.1 | | | | |
| Q1.2 | | | | |
| Q1.3 | | | | |
| Q2.1 | | | | |
| Q2.2 | | | | |
| Q2.3 | | | | |
| Q3.1 | | | | |
| Q3.2 | | | | |
| Q3.3 | | | | |
| Q3.4 | | | | |
| Q4.1 | | | | |
| Q4.2 | | | | |
| Q4.3 | | | | |

## Write conflict tests (vs serial oracle; see `docs/write-path-design.md` §7)

| # | Case | Replay path | Atomic fast path |
| --- | --- | --- | --- |
| 1 | Many SET updates of one hot row | | |
| 2 | Many +delta updates of one hot row | | |
| 3 | Duplicate INSERTs of a new key | | |
| 4 | INSERT of an existing key | | |
| 5 | UPDATE/DELETE of missing keys | | n/a |
| 6 | DELETE → INSERT → UPDATE, all seq orders | | n/a |
| 7 | Capacity overflow rejects whole batch | | |
| 8 | Large random mixed batches, uniform + Zipf | | n/a |
| 9 | 13 queries after writes vs DuckDB | | |
| 10 | Determinism (same batch twice) | | |

## Open problems / blockers

- (none yet)

## Explain-back record

Topics each of us has explained back without help (from `/explain-back`):

- (none yet)
