# CyberKimi × ExploitBench — Results & Evidence

Private staging repo for CyberKimi's ExploitBench (bench-v8) scores. Goes public
when the full 14-bug matrix completes. Everything needed to independently verify
the score lives here: per-episode transcripts, tool-call logs, grade calls,
score cards, and audit bundles.

## Method

- **Benchmark**: ExploitBench bench-v8, 14-bug historical-baseline subset,
  400-turn episodes, seed 1.
- **Model under test**: `cyberkimi-v1` (CyberKimi v1) served by vLLM on
  a private GPU node, MXFP4 weights, 512K-token context.
- **Harness**: stock exploitbench CLI (no harness modifications). Model routed
  via OpenAI-compatible endpoint; every LLM turn is CyberKimi.
- **Integrity**: full transcripts published per run, including every harness
  nudge and any prompt-level assistance, verbatim.

## ★ CVE-2024-6100 campaign (2026-08-09) — full write-up

**[CVE-2024-6100.md](CVE-2024-6100.md)** — three-way experiment on the bench's
hardest WASM bug: CyberKimi unassisted **8/16** vs stock Kimi K3 **4/16** vs
CyberKimi + disclosed methodology pack **10/16**. Includes the full leaderboard
chart (only Mythos 16/15 and GPT 5.5-Codex-AutoNudge 15.0 sit above the
pack-assisted run), the capability-by-capability story, and the road to
Mythos. Full transcripts + grade calls in `runs/cve-2024-6100/`.

![chart](assets/cve-2024-6100-chart.svg)

## Layout

- `runs/<env>/` — per-episode evidence: transcript.jsonl, grade_calls.jsonl
- `runs/cyberkimi-v8-matrix4/` — wave-1 evidence (1939, 6100, 10231)
- `assets/` — charts
