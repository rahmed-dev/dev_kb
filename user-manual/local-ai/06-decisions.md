# 06 — Decisions and dead ends

Things settled by argument or measurement, kept so they don't get re-litigated. Each says what
was decided, why, and what would change the answer.

## Keep llama.cpp; don't migrate to LM Studio

**Settled 2026-08-14.**

- A GUI hides the exact knobs that made this fast — `--n-cpu-moe`, KV quantization, slot counts,
  the WSL/WDDM silent-spill failure mode. It would degrade under the same conditions and report
  nothing.
- **Both cannot run at once.** One 16GB card. LM Studio on the Windows host takes VRAM the WSL
  container needs. It is a swap (`docker stop qwen` first), never an addition.
- The existing config is measured, documented, and auto-restarting. Migration discards that
  baseline.

Agreed role for a GUI: an optional scratchpad on the host for browsing and downloading models
and comparing quants. Not the backend.

**What would change it:** wanting to run several models interactively, or no longer caring about
the last 2x of performance.

## "GPU-only is impossible on 16GB" — wrong, and instructive

The claim was true of the 21GB *file*, false of the *model*. A 14GB quant of the same model needs
only 4 layers offloaded. See [02-model-and-quant.md](02-model-and-quant.md).

The failure mode is worth more than the fact: **a constraint was measured accurately under one
condition and then written down as a property of the hardware.** Every "impossible" in your notes
deserves a date and the condition it was measured under.

## Keep the MoE; don't switch to a dense 14B

Considered when cold loads were painful. A dense 14B at Q5_K_M fits entirely in 16GB — instant
first token, faster decode, less capability.

Kept the 35B MoE deliberately. Once the quant swap removed the cold-load pain, the reason to
switch mostly evaporated.

**What would change it:** capability at 3B active proving insufficient, or wanting several models
resident at once.

## Don't chase local agentic coding on this box

Concluded after measurement: the real bottleneck for coding-agent harnesses is **prefill depth**.
845 t/s @2.7k → 976 @30k → 375 t/s @71k. At 71k a single turn takes 190 seconds, and agent
harnesses cancel before it returns.

The box was repurposed to knowledge work, where prompts are short and the wiki pattern keeps them
in the fast band. Two coding clients (pi, opencode) were removed.

## A wiki beats vector RAG here — for a performance reason

Not an ideological preference. RAG stuffs many retrieved chunks into the prompt and pushes the
request into the slow prefill band. A compiled wiki reads a few short, current notes.

Embeddings were deliberately deferred — no embedding model on this box. Add one only if semantic
search over raw documents proves necessary; it would be a second model on a second port,
competing for the same GPU (see [04-first-prompt-latency.md](04-first-prompt-latency.md) for what
that competition costs).

## Things ruled out by measurement — don't re-chase

- **Host RAM pressure / `.wslconfig` sizing as the inference bottleneck.** Task Manager at 92%
  during *load* is expected; WSL reported 20GB available throughout. Not a steady-state problem.
- **mmap thrashing.** The ~39k major page faults accumulate during model load, not inference.
  Three 30k-token prompts added **+2**.
- **Slot cache misses as unavoidable.** Prefix caching works: a 30k prompt re-sent with a new
  suffix reported `cached=29203, prompt_n=520` — 1.1s versus 37s.

## Two fixes that look right and are wrong

Both were proposed, both are traps under the *single-session* usage this box actually has:

1. **`-np 1` to avoid slot contention.** Slots have prefix affinity — different conversations
   keep their own warm caches. One slot makes every request share it, so any alternation wipes
   the prefix. Trades a rare problem for a constant one. (It also costs ~2x prefill; see
   [03](03-tuning-and-benchmarking.md).)
2. **Raising `-c` to enlarge the pool.** Only helps *concurrent* agents. With concurrency at one
   there is nothing to make room for, and it costs prefill throughput plus VRAM headroom.

## The meta-lesson

Nearly every wrong turn here had the same shape: **a measurement taken under one set of
conditions, written down as a permanent law, and then obeyed after the conditions changed.**

`-np 1 is REQUIRED` cost 2x prefill. `GPU-only is impossible` cost weeks of the wrong mental
model. `keep ≥2GB VRAM free` was inherited into a config where it didn't apply.

Date your findings. Record the conditions. Re-test after upgrades.
