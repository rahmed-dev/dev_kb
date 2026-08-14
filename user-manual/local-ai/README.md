# Local AI

Everything learned running a local LLM as the backend for a personal knowledge base, on one
16GB card. Written to be re-read — by me, and on the next machine.

**Reference setup:** RTX 5060 Ti 16GB (Blackwell, sm_120) · Windows 32GB host, model served from
WSL2 in Docker (`ghcr.io/ggml-org/llama.cpp:server-cuda`) on port 8090 · Qwen3.6-35B-A3B
(MoE, 3B active) at UD-IQ3_XXS · Hermes Agent on the Windows side, against an Obsidian vault.

## Read in order

| # | Doc | What it covers |
|---|---|---|
| 01 | [Setup](01-setup.md) | From nothing to a working OpenAI-compatible endpoint. Docker vs building, GPU passthrough, WSL memory, downloading models, every launch flag explained. |
| 02 | [Model and quant](02-model-and-quant.md) | The choice that matters most. Quant naming, MoE vs dense, KV cache math, where the real context ceiling is. |
| 03 | [Tuning and benchmarking](03-tuning-and-benchmarking.md) | How to make it fast and how to prove it. `--n-cpu-moe`, slots, speculative decoding, the prefix cache. |
| 04 | [First-prompt latency](04-first-prompt-latency.md) | Case study: 33 seconds → 0.3 seconds. The best worked example of the method in 03. |
| 05a | [Client: Hermes tools](05-hermes-tool-trim.md) | Token cost of every tool schema, and which to keep for KB work. |
| 05b | [Client: Hermes skills](05-hermes-skill-audit.md) | All 77 bundled skills, keep/drop, and why trimming them saves almost nothing. |
| 06 | [Decisions and dead ends](06-decisions.md) | What was settled and why, so it doesn't get re-argued. |

## The five things worth remembering

**1. The quant, not the hardware.** A 21GB file on a 16GB card produced every symptom — slow
decode, frozen desktop, host RAM pressure — and the conclusion "this hardware can't do it." A
14GB quant of the *same model* was 3.3x faster. Never generalize a quant's footprint to a model.

**2. Measure the breakdown before cutting.** Chasing first-prompt latency, the obvious suspect
(77 skills) turned out to be 4% of the prompt. Tool definitions were 62%.

**3. "Cheap" auxiliary calls aren't cheap with one model.** Session title generation — a feature
nobody thinks of as a cost — burned 10-20 seconds per session by evicting or competing with the
real request. It never appears in any token breakdown, because it is a separate request.

**4. Cache preservation beats raw throughput.** Trimming 8k tokens of tool schemas saved 10s.
Keeping the KV prefix cache saved 22s. Losing a 51k prefix costs ~64s; peak prefill rate
differences cost ~2s.

**5. Constraints expire — date them.** Three separate notes here were once true and later cost
real performance: `-np 1 is REQUIRED with MTP` (2x prefill), `GPU-only is impossible` (wrong
mental model for weeks), `keep ≥2GB VRAM free` (inherited into a config where it didn't apply).
Record the measurement and the conditions, not just the conclusion.

## On reusing this elsewhere

The **method** transfers: benchmark every change, read the slot-selection line, trust numbers
over documentation (including this documentation).

The **numbers** don't. `--n-cpu-moe 4`, `-c 131072`, and `-np 2` are sized to a 16GB card and a
14GB model file. On different hardware, re-derive them with the procedure in
[03](03-tuning-and-benchmarking.md) — especially any threshold that reads like a law.
