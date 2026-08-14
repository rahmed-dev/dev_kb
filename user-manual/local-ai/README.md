# Local AI

Running a local LLM as the backend for a personal knowledge base, on one 16GB card.

**Setup:** RTX 5060 Ti 16GB · Windows 32GB host, model served from WSL in Docker
(`ghcr.io/ggml-org/llama.cpp:server-cuda`) on port 8090 · Qwen3.6-35B-A3B (MoE, 3B active) at
UD-IQ3_XXS · Hermes Agent on the Windows side as the client, against an Obsidian vault.

## Documents

| Doc | What it answers |
|---|---|
| [first-prompt-latency.md](first-prompt-latency.md) | Why the first prompt of a session took 33 seconds, and how it got to 0.3s. The main case study — read this one first. |
| [hermes-tool-trim.md](hermes-tool-trim.md) | Token cost of every Hermes tool schema, and which to keep for knowledge-base work. |
| [hermes-skill-audit.md](hermes-skill-audit.md) | All 77 bundled skills by category, keep/drop for a KB maintainer, and why trimming skills saves almost nothing. |

## The short version

Three findings that generalise beyond this box:

1. **Measure the breakdown before cutting.** The obvious suspect (77 skills) was 4% of the
   prompt. Tool definitions were 62%.
2. **"Cheap" auxiliary model calls are not cheap when you only have one model.** Session title
   generation — a feature nobody thinks of as a cost — was burning 10-20 seconds per session by
   evicting or competing with the real request. It never appears in any token breakdown, because
   it is a separate request.
3. **Cache preservation beats raw throughput.** Trimming 8k tokens of tool schemas saved 10s.
   Keeping the KV prefix cache saved 22s.

And one process lesson: **re-test documented constraints after upgrades.** A note saying
`-np 1` was mandatory cost nearly 2x prefill throughput and had simply gone out of date.
