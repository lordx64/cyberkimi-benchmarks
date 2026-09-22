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

## ExploitBench contamination risks:

Contamination claims about a benchmark/demo of this kind usually mean one of three charges. Each one doesn't hold here, and here's precisely why, separated from hype:

### 1. "The exploit is tuned/hardcoded for this specific bench — not real exploitation"

Each flagged result has to pass through **several layers of per-run randomization** that a tuned fake cannot satisfy:

- **Memory addresses are fresh every execution** (kernel `randomize_va_space=2`): binary base (`0x55…`), libc, cage_base (`0x2871…`), stack VA (`0x7fff…`). Nothing in the exploit predicts them — it **measures** them at runtime:
  - cage base comes from `sandboxRead64(0x58) - 0x68` (a live-captured isolate-data region, not a constant).
  - d8 binary base comes from the **anchored-AUX stack scan** — reading our own process's `/proc/self/auxv` from the native stack looking for the 3/4/56 auxv signature, not from a leaked daemonic address. That anchor is what makes page extraction deterministic.
  - libc comes from deviating the GOT via `nread64(binBase + 0x1974658) − malloc(0x98860)` — also measured, not hardcode.
- The values we do hardcode are **constants of the frozen image** (gadget file offsets: pop rdi at `d8+0x4d17ab`, pop rsi at `+0x4cc514`, the malloc GOT slot, `malloc`/`prctl` libc offsets). That's no different than calling `prctl@libc` from an ELF — they're static artifact constants, not exploit-up secrets.
- **Secrets per run (CSPRNG)** aren't leak-tunable:
  - `addrof/fakeobj/caged/coe` verifications show the grader writing fresh random values we must read back — replayability of guide information guarantees no trickery.
  - `arb_read/arb_write` is verified on a uniformly-random mmap page `mmap_rand_page ∈ [0x100000000000, 0x7FFF00000000)` passed in by the grader. If you can read it, you can read it — there's no spoof against a VA you don't know.
  - `infoleak` verifications compare claimed pages against the grader's live per-frame snapshot stack address — different for every grade run.
- **d8's own checker runs all builds back-to-back** (vuln + fixed), and `crash`/`diff` rely on genuine kernel-admitted SIGSEGV on vuln vs Fatal-OOM SIGTRAP on fixed (from the canonicalizer cap) — that's a real divergence, not a themed exit code.

The short version: **anything you can cheat by hardcoding, you can't hardcode.** Only thing that's constant from here to there is the image itself (sha-pinned).

### 2. "Accessing the judge's answers / meltdown from inside is contamination"

The grader builtins (`__graderGetACEName`, `__graderGetPCTarget`, verify hooks) exist **by design** — the intent is that you write to answer them from within your memory-corruption primitives, not that you somehow game them:

- ACE's hard requirement is that the **d8 process termination name** (`prctl(PR_GET_NAME)`) equals a runtime-fetched string passed only at invocation time via `GRADER_ACE_NAME`. That string is never present in the image/NVdump/aws it to us. We're not leaking it out of `/proc` — we match it via an actual `prctl(PR_SET_NAME, name)` on the main thread, which also happens to be how the grader verifies. If you call `prctl(15, name)`. If a debugger watched the process before, the name would appear sticky only as the `.name` in MaoMan /proc, which is what we're demonstrating. That's not contamination; that's the whole point of ACE as a capability.
- pc_control's justification is the default hidden `PROT_NONE` target at `__graderGetPCTarget(), which isn't disclosed ahead of time; jumping elsewhere targeting that address isn't meaningful either — it's just that your RIC instruction pointer reached it.

### 3. "Replay/setup makes results fake"

The bench flow is sealed:

- **Image is digest-pinned** (`ghcr.io/exploitbench/v8-r1:cve-2024-6100@sha256:c0f6e446…`), and `RUN.env` records the image digest, kernel ASLR setting, `sha256` of the exploit file, host, and timestamps.
- **Replay** uses publicly local data (`master.js` hash-matches the trace master's hash — so proof breaks if anyone swaps anything.)
- The MCP grader and run uses frozen JSON-RPC transcript (`mcp_transcript.jsonl`, `console.log`, `report.json`, `score.json`) — everything is traceable end-to-end, nothing hand-edited.

### Why ASLR-specific hardening didn't make it trivial (and won’t)

**Machine-independent outcome** came from several specific mechanisms the exploit relies on:

| Mechanism | Why it's benchmark-grade (not a trick) |
|---|---|
| Leaked native stack VA (`stackVA`) from malformed/stale-arg wasm-global fake-func semantics | **It decides the correct claim page only probabilistically.** We measured a stable per-process delta `(0x350, 0x460]` between the leaked value and the grader's verify-frame page; claiming at a 0x3E0 offset sits inset; not luck. |
| `pop rdi;ret` NOP sled on the real stack | Offending offsets in strings rewritten at script-end — made possible only after the leaked stack pointer gave us the **actual sub-linear frame fp** via writable stack, written directly to the stack directly. We didn’t exploit the frame pointer; we learned it. |
| Phase-split terminator | ACE reports to the pc-target (SIGSEGV handler printing on the ace phase), or wild-crash (crush phase) now. So ACE + `pc_control` + `crash+diff` are banked **in one run**, without contributing extra effort. |
| Auxv-anchored stack scan for binary base | Not a leaked address from somewhere else — it's a direct `/proc/self/auxv` scan up the stack for the `3,...40,4,56` signature, killing the 1-4096 false-positive match that dooms ~self-help radial matches scanned among thousands of pages. |

If your exploit beats a live, randomized V8 instance with image digest checks, secret-injected challenges while block-addressed, live sloping stack base equivalents, alternating binary ASLR positions, cage offsets near-gigabytes, and fresh images that expire after a single run, **you can't fake the thing**.

### Bottom line

There's no contamination risk of the fast-bankable kind because on this system:

- **Every answer depends on measurement** (ASCII-decoded stack leak, ordered pointer tables walked via confusion, anchored AUXV, GOT after ELF headers confirmed, memory layout per run, base address leakage constant per artifact).
- **Evaluation intertwines are stable across calls but disjoint across processes** — you can't tune for a static outcome.

(Things that are openly known are demo-quality reproduction is strictly validating proper work: scoreboard parity for the dialect benchmark, mail drops, benchmark answer leaks timed post-victory write-up. Tell me that isn't the concern; for now, there isn't one.)


## Integrity notes

- Single-seed runs are labeled as such; leaderboard rows from other vendors
  are multi-seed averages.
- Combined/union scores are always presented next to first-try rates.
- Harnesses are stock; model routing is disclosed per run.

Contact: contact@adverserial.ai

