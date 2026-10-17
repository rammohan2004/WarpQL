# Benchmarking protocol

Every number in the report must come from a run following these rules. Claude Code must refuse to report a number that was not produced this way.

## Before a run

- Laptop plugged in, Windows power mode at best performance, other GPU apps closed.
- Release build (`-O3`, no debug checks inside timed regions).
- Record: GPU model, driver, CUDA version, CPU model and cores, RAM given to WSL, scale factor, block size, commit hash.

## How to time

- GPU kernels: CUDA events around the kernel(s); report milliseconds.
- CPU: `std::chrono::steady_clock` around the same work.
- End-to-end GPU: include host→device copies of the columns the query reads.
- 1–2 warm-up runs, then at least 5 timed runs. Report median, min and max.
- Never compare GPU kernel-only time with CPU end-to-end time without labelling it.

## Peak bandwidth

Measure it once with a simple copy kernel over ~1 GB (or less if GPU memory is small): bandwidth = 2 × bytes / time. Use this measured number as 100%, not the spec sheet.

## Effective bandwidth of a kernel

Bytes the kernel must read + write ÷ kernel time. Compare with the measured peak.

## One CSV format

`experiment,parameter,engine,variant,run,time_ms,bytes,sf,commit`

All plots are generated from these CSVs only. Keep raw CSVs in `results/`.

## Experiments (scale to the laptop's memory per `docs/hardware.md`)

| Experiment | Sweep |
| --- | --- |
| SSB end-to-end | all 13 queries × chosen scale factors × {CPU, GPU, GPU fused} |
| Selectivity | filter over the largest array that fits, selectivity 0.1% → 100% |
| Join build size | build rows 1K → as large as fits, fixed probe count |
| Group count | 1 → 10M groups (or as fits), uniform and skewed keys |
| Sort size | 1M → as large as fits, 32- and 64-bit keys |
| Small inputs | 10K → 1M rows, to show where the CPU wins |
| Write throughput | batch size 1K → largest that fits; statements/s for segmented replay, atomic fast path, OpenMP CPU |
| Write conflicts | fixed batch size, hot-key fraction 0% → 100% and Zipf skew; mixed vs update-only vs insert-only |
| Writes + reads | queries before and after batches; query time vs fraction of deleted rows (0% → 50%) |

For write runs, also record: operation mix, key distribution, conflict rate, capacity headroom, and the number of statements rejected. Throughput counts all statements processed, accepted or rejected; report both.

## Optimisation runs

One change at a time, everything else fixed. Log each in `NOTES.md` using the logbook template, including failures.

## Laptop caveats to state in the report

- Laptop GPUs throttle under sustained load; variance may be higher than on a server GPU.
- WSL2 adds a virtualisation layer; we report it as our environment.
- GPU utilisation from `nvidia-smi` is unavailable on WSL2.
