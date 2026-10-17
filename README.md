# WarpQL setup kit

Setup for building **WarpQL, a GPU-native columnar query engine** (CS6023 course project) with Claude Code. WarpQL runs analytical queries with hand-written CUDA kernels and supports batched **INSERT, UPDATE and DELETE** on the GPU, with conflicting writes resolved deterministically by sequence number.

The kit contains no project code; we write that step by step with Claude Code, which explains what, why and how before every step.

## What's in here

| Path | Purpose |
| --- | --- |
| `CLAUDE.md` | Rules Claude Code follows in every session (teaching protocol, hard rules, conventions) |
| `plan.md` | Step-by-step build plan, run back to back as fast as possible; Gate A (reads) at Step 6, Gate B (writes) at Step 7 |
| `PROGRESS.md` | Where we are, query status, write conflict-test status |
| `NOTES.md` | Daily log, optimisation logbook, bug stories, sources |
| `docs/write-path-design.md` | INSERT/UPDATE/DELETE semantics, conflict handling, kernels, test matrix — the answer to Prof. Nasre's question |
| `docs/project-overview.md` | Architecture, data model, kernels, experiments, scope |
| `docs/environment-setup-wsl2.md` | Installing CUDA on Windows + WSL2, profilers, limitations |
| `docs/benchmarking-protocol.md` | How every number must be measured (reads and writes) |
| `docs/interview-prep.md` | Questions to answer after each step |
| `docs/resources.md` | Reading list by step |
| `.claude/skills/` | Claude Code skills (below) |

## Skills

| Command | When |
| --- | --- |
| `/start-session` | Beginning of every session |
| `/explain-step` | Before every step (Claude also runs it automatically) |
| `/verify-step` | Before calling anything done |
| `/explain-back` | After each step: Claude quizzes both of us |
| `/hardware-check` | Step 0: detect the GPU, pick data scale and insert capacity |
| `/write-path` | Step 7 and any write-path work (also loads automatically) |
| `/optimise <idea>` | Steps 9–10: one measured experiment |
| `/end-session` | End of every session: update progress, log, commit |
| `cuda-kernel` | Loads automatically when editing `.cu` files |

## First run

1. Follow `docs/environment-setup-wsl2.md` sections 1–6 (Windows driver, WSL2, CUDA toolkit, build tools, Claude Code).
2. Inside WSL, unzip this kit into your Linux home: `~/warpql` (not under `/mnt/c`).
3. Run `claude` in that folder.
4. Type `/start-session`. It will propose Step 0; reply **go** after each explanation.

Good first message after `/start-session`:

> We're starting Step 0. Walk us through checking the environment, then run /hardware-check.

## Working rhythm

- Sit together; one person reads Claude's explanation aloud, the other answers the check-in question.
- Swap who answers `/explain-back` questions each step.
- Don't skip `/verify-step`; a broken scan silently corrupts every query built on it, and a racy write kernel can pass once and fail the next run.

## Submission

Moodle by 7 November 2026: a code zip or a link to a public GitHub repository. Presentation during end-sems.

## Academic integrity

Check the course's policy on AI-assisted code with Prof. Nasre, and disclose it in the report as required. `NOTES.md` keeps a record of what was AI-written and what you changed.
