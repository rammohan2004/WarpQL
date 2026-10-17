# WarpQL — project rules for Claude Code

This file is loaded in every session. Follow it over any default habit.

## What this project is

WarpQL is a GPU-native columnar analytical query engine, built by two students for CS6023 (GPU Programming, Prof. Rupesh Nasre, IIT Madras). It runs the 13 Star Schema Benchmark (SSB) queries with hand-written CUDA kernels (filter, scan/compaction, gather, hash join, aggregation, radix sort, top-k, fused tile kernels), supports batched INSERT, UPDATE and DELETE on the GPU with a serial-equivalence guarantee (the answer to Prof. Nasre's question on conflicting writes), compares against a C++/OpenMP CPU engine using the same algorithms, and checks every query result against DuckDB and every write batch against a serial oracle.

- Full design: `docs/project-overview.md`
- Write path and concurrency design: `docs/write-path-design.md`
- Step-by-step plan: `plan.md`
- Where we are now: `PROGRESS.md` (read it at the start of every session)
- Logs: `NOTES.md` (daily log + optimisation logbook)
- Hardware facts and chosen data scale: `docs/hardware.md` (created by `/hardware-check`)

## The two of us

Two students work together on one laptop, side by side. Treat both as the user. Both must be able to explain every line in an interview, so teaching matters as much as finishing.

## The teaching protocol (mandatory, every step)

Before doing ANY step that creates or changes code, build files, data or the environment, run the `/explain-step` protocol:

1. **What** we are about to do, in 2–4 sentences.
2. **Why** — where it sits in the engine and what breaks or is missing without it.
3. **How** — the approach: thread/block mapping, memory spaces used, synchronisation, data layout, algorithm, and the alternatives we are *not* choosing and why.
4. **Concepts** — the GPU/CS concepts involved, each in one line.
5. **Pitfalls** — the 2–3 bugs most likely here.
6. **Files** — which files will be created or changed.
7. Then **stop and wait** for us to reply "go" (or ask questions). Never start writing before that.

After writing: walk through the code in small pieces, build it, run its tests, show the real output, then run `/explain-back`.

Small housekeeping (formatting, typo fixes, reading files, running an existing test) does not need the full protocol; one line saying what you are doing is enough.

## How code gets written

- We chose "Claude writes, we review". Write code in small reviewable chunks: one kernel, one class or one test file at a time, ideally under ~150 lines per chunk. Never write several steps' worth of code in one go.
- After each chunk, pause for questions before continuing.
- Prefer clarity over cleverness in the first version; optimisation comes in Step 9 with measurements.
- Every kernel gets a unit test against the CPU reference before it is used in a query.
- Do not copy code from Crystal, CUB, cuDF or other projects. We may read them only after our own version works, and anything inspired by them must be cited in `NOTES.md`.

## Hard rules

- **Never fabricate numbers.** Every timing, speedup or bandwidth figure must come from a run we actually did, with the command and configuration recorded. If something has not been measured, say "not measured yet".
- **Correctness before speed.** No optimisation work until Gate A (Step 6: all 13 SSB queries match DuckDB) and Gate B (Step 7: write batches match the serial oracle and queries still match after writes) both pass.
- **Every scan honours the validity bitmap.** Any new read kernel must skip deleted rows, and its tests must include deleted rows.
- **Write semantics are fixed by `docs/write-path-design.md`** (serial equivalence by sequence number). Do not change them without asking us; if an implementation choice would break serial equivalence, stop and explain.
- **One change at a time** when optimising; log hypothesis, change, before, after, evidence, conclusion in `NOTES.md`.
- **Do not use CUDA Unified Memory** on the main path (limited on WSL2). Use explicit `cudaMalloc` + `cudaMemcpy`.
- **Never install an NVIDIA Linux driver inside WSL**, and never install the `cuda`, `cuda-12-x` or `cuda-drivers` meta-packages there. Only the WSL-Ubuntu `cuda-toolkit-*` package. See `docs/environment-setup-wsl2.md`.
- Keep the project inside the WSL Linux filesystem (`~/...`), not under `/mnt/c/...` (very slow file I/O).
- Use the GPU architecture recorded in `docs/hardware.md` (e.g. `CMAKE_CUDA_ARCHITECTURES`), not a guess.
- Respect the data scale in `docs/hardware.md`; a laptop GPU has far less memory than an A100.
- Do not add web frontends, databases, Docker or services. This is a GPU course project.

## Coding conventions

- C++17, CUDA, CMake, GoogleTest, OpenMP. Python only for data conversion and plots.
- Columns are structure-of-arrays; strings are dictionary-encoded with **sorted** dictionaries; decimals are scaled integers; dates are `yyyymmdd` int32.
- Wrap every CUDA API call in an error-check macro; check the last error after every launch in debug builds.
- RAII for device memory; a memory pool so nothing is allocated while timing.
- Sums use 64-bit accumulators. Watch for 32-bit overflow in indices and products.
- Kernel tests always include sizes 0, 1, 31, 32, 33, 1023, 1024, 1025, 1,000,007 and one large size.

## Session rhythm

- Start: run `/start-session`.
- During: `/explain-step` before each step; `/verify-step` before marking anything done.
- End: run `/end-session` (updates `PROGRESS.md`, `NOTES.md`, commits).

## Academic integrity

Code written with AI assistance must be disclosed according to the course's policy. Remind us of this when writing the report, and keep `NOTES.md` honest about what was AI-written and what we changed.
