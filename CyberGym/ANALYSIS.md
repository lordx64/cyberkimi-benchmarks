# CyberGym Campaign — Research & Analysis Findings

Companion analysis to the [README](README.md) results. All numbers are computed
from the raw artifacts in `traces/`.

## The capability boundary is sharp, not gradual

Solves and fails look completely different in the time domain:

| | solved runs | failed runs |
|---|---|---|
| median iterations | 103 | 250 |
| median wall-clock | 8 min | 34 min |

When CyberKimi understands the bug, it converges fast — reproduce the crash
input, minimize, submit. When it doesn't, it explores productively for 4×
longer (reading source, building corpora, mutating inputs) without crossing
the threshold. More budget on a failing run rarely flips it — the bottleneck
is insight into the bug semantics, not search time.

## Retries recover 61% of failures — run variance is real

Phase 2 re-ran the 31 phase-1 failures with a fresh context: **19 solved**.
Same model, same task, different sampling trajectory. Two implications:

1. **Single-attempt scoring understates capability.** The honest headline is a
   pair: 65.6% first-try, 86.7% with one retry. Multi-seed averaging (the
   standard on other benches) would land in between.
2. **Failure is partly stochastic, partly structural.** The 12 tasks that
   failed both phases are the genuinely hard set (deep semantic state bugs —
   e.g. ICU collation rule parsing, where the crash needs a specific
   multi-constraint input shape). Those are the training signal that matters.

## Grader strictness artifacts

Four phase-1 "FAIL"s flipped to SOLVED on a database rescore — the PoC
reproduced the crash but the automated check missed it (timing/signature
matching). Any automated re-grading of these traces should re-execute PoCs
rather than trust the recorded label.

## What feeds the training pipeline

Every trace is outcome-labeled and tier-mappable:

- **78 solved traces** → positive SFT/DPO examples: full reasoning → PoC
  source → confirmation, in context.
- **12 double-fail traces** → the negative set: long productive-but-failing
  explorations, ideal DPO "rejected" samples and RL hard-negative mining.
- **19 recovered-on-retry pairs** → same-task success/failure contrast pairs —
  the cleanest DPO material in the set, since task difficulty is controlled.

## Methodological caveats (kept honest)

- Single seed per task; no multi-seed averaging yet.
- The 86.7% figure is a union over two attempts — always cite it alongside the
  65.6% first-try rate.
- Model served at 512K context; long source-reading sessions were never
  context-truncated, so context length is not a confound in these results.
