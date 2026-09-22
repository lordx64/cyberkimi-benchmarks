# CyberKimi — Benchmarks & Evidence

Public evidence for CyberKimi's cyber capability claims. Every score published
here ships with the raw material needed to audit it: transcripts, tool-call
logs, grader outputs.

CyberKimi is Kimi K3 with the refusal layer ablated and cyber-tuning on top,
served on our own GPU infrastructure. [adverserial.ai](https://adverserial.ai)

## Benchmarks

### [`CyberGym/`](CyberGym/) — real-world crash reproduction (90 targets)

Real sanitizer bugs from production fuzzing (arvo / oss-fuzz); the agent must
write a PoC that reproduces the crash under deterministic re-execution grading.

**78 / 90 solved (86.7%)** — 65.6% first-try, 61% of failures recovered on a
single retry. Full traces, results tables, and analysis included.

### [`ExploitBench/`](ExploitBench/) — V8 exploitation ladder (CVE-2024-6100)

Beyond crash reproduction: 16 graded capabilities from coverage to arbitrary
code execution on the bench's hardest WASM type-confusion bug.

**16/16 unassisted** (stock Kimi K3 control at 4/16)

## Integrity notes

- Single-seed runs are labeled as such; leaderboard rows from other vendors
  are multi-seed averages.
- Combined/union scores are always presented next to first-try rates.
- Harnesses are stock; model routing is disclosed per run.

Contact: contact@adverserial.ai
