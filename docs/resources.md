# Resources, by step

| Step | Resource | Use it for |
| --- | --- | --- |
| 0 | CUDA on WSL User Guide — https://docs.nvidia.com/cuda/wsl-user-guide/index.html | Driver/toolkit rules, WSL limitations |
| 0 | ssb-dbgen — https://github.com/eyalroz/ssb-dbgen | Generating SSB data |
| 0–2 | SSB paper (O'Neil et al., 2009) — https://cs.umb.edu/~poneil/StarSchemaB.pdf | Schema and official query definitions |
| 0–2 | ClickHouse SSB page — https://clickhouse.com/docs/get-started/sample-datasets/star-schema | SQL text of the 13 queries |
| 0–2 | DuckDB docs — https://duckdb.org/docs/ | `read_csv`, Python client, `fetchnumpy()` |
| 1–5 | CUDA C++ Programming Guide — https://docs.nvidia.com/cuda/cuda-c-programming-guide/ | Warp intrinsics, atomics, shared memory |
| 3 | GPU Gems 3, Ch. 39, Parallel Prefix Sum (Scan) with CUDA — https://developer.nvidia.com/gpugems/GPUGems3/gpugems3_ch39.html | Scan algorithms, scanning large arrays |
| 3, 8 | Using CUDA Warp-Level Primitives — https://developer.nvidia.com/blog/using-cuda-warp-level-primitives/ | ballot, shuffle, match, masks |
| 3, 8 | CUDA Pro Tip: Warp-Aggregated Atomics — https://developer.nvidia.com/blog/cuda-pro-tip-optimized-filtering-warp-aggregated-atomics/ | Leader election; note nvcc may already aggregate some atomics automatically |
| 5 | CUB documentation (DeviceRadixSort, BlockScan) | Reference baseline only, not for our implementation |
| 3–6 | Compute Sanitizer — https://docs.nvidia.com/compute-sanitizer/ | memcheck, racecheck, synccheck |
| 7 | Nsight Systems — https://docs.nvidia.com/nsight-systems/ | Timelines |
| 7 | Nsight Compute — https://docs.nvidia.com/nsight-compute/ | Per-kernel metrics |
| 7–8 | CUDA C++ Best Practices Guide — https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/ | Coalescing, occupancy, effective bandwidth |
| 9 | Crystal paper — https://arxiv.org/abs/2003.01178 | Tile-based fused execution; read code only after ours works |
| 7 | `docs/write-path-design.md` (in this kit) | Our write semantics, kernels and test matrix |
| 7 | DBMS textbook chapters on serializability, concurrency control and isolation levels (e.g. Silberschatz, Korth & Sudarshan; your CS3700 TA material) | Vocabulary for the report and the professor's question |
| 7 | "Calvin: Fast Distributed Transactions for Partitioned Database Systems", Thomson et al., SIGMOD 2012 (search the title) | Deterministic ordering of transactions before execution: the idea behind our design |
| 7 | "High-Throughput Transaction Executions on Graphics Processors", He & Yu, PVLDB 2011 (search the title) | Prior work on batched transactions on GPUs; cite and compare, don't copy |
| 7 | DuckDB docs on constraints and INSERT/UPDATE/DELETE — https://duckdb.org/docs/ | Applying the same statements in the oracle harness |
| Stretch | "Single-pass Parallel Prefix Scan with Decoupled Look-back", Merrill & Garland, 2016 | One-kernel scan |
| Stretch | "Hardware-conscious Hash-Joins on GPUs", Sioulas et al., ICDE 2019 | Partitioned joins |
| Claude Code | Skills docs — https://code.claude.com/docs/en/skills | Editing the skills in `.claude/skills/` |

Also useful: CS6023 lecture notes on scan, reduction, atomics and coalescing; your Δ-stepping SSSP code for device-memory and timing patterns.
