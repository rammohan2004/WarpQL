# Notes

Two logs in one file. Claude Code appends here; we review. These notes become the raw material for the report and for interview answers.

---

## Daily log

Template (one entry per working session):

```
### YYYY-MM-DD
- Done:
- Broken / surprising:
- Decisions (and why):
- AI-written vs our changes:
- Next:
```

(entries below)

---

## Optimisation logbook

One entry per attempt, including the ones that made things slower.

Template:

```
### OPT-NN: <short name>
- Hypothesis:
- Change (one thing only):
- Setup: GPU, scale factor, block size, build flags, power mode, command
- Before: median ms (min–max), N runs
- After: median ms (min–max), N runs
- Evidence: Nsight metric(s) or bytes moved
- Conclusion: helped / no effect / hurt, and why
- Commit:
```

(entries below)

---

## Bugs worth remembering

Template: symptom → how we found it → root cause → fix. Good interview stories.

(entries below)

---

## Sources and inspirations

Anything we read that shaped our code (papers, blog posts, Crystal, CUB). Cite in the report.

(entries below)
