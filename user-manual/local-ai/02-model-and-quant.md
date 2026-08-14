# 02 — Choosing the model and the quant

This choice decides more about your experience than every flag combined. On the reference box, a
different quant of **the same model** gave 3.3x decode speed and turned a minutes-long load into
30 seconds.

## The single most important lesson

> **Never generalize a quant's footprint to the model.**

The original file was `Qwen3.6-35B-A3B-MXFP4_MOE.gguf` at 21GB, on a 16GB card. 20 expert layers
had to live in host RAM. Everything bad followed from that one fact: cold loads that froze the
Windows desktop, ~10GB of host RAM pressure, 25 t/s decode, and a conclusion — written down as
settled — that "GPU-only is impossible on this hardware."

That conclusion was **wrong**. It was true of the 21GB *file*, not of the model. An IQ3_XXS quant
of the identical model is 14.07GB and needs only 4 layers offloaded:

| | MXFP4, `--n-cpu-moe 20` | IQ3_XXS, `--n-cpu-moe 4` |
|---|---|---|
| Decode | 25.0 t/s | **80-83 t/s** |
| Prefill @30k | 776 t/s | **1,398 t/s** |
| Load time | minutes, desktop hangs | **~30 s** |
| Host RAM | ~10 GB | **2 GB** |

The hardware was never the constraint. Before buying a card or downgrading to a smaller model,
check whether a smaller quant of the model you want fits.

## Reading quant names

- **Q4_K_M, Q5_K_M, Q6_K** — k-quants, the mainstream choice. Higher = better quality, bigger.
- **IQ2/IQ3/IQ4_XXS/XS** — "importance matrix" quants. Smaller at equal quality, slightly slower
  per-token on some hardware. `IQ3_XXS` ≈ 3.2 bits per weight.
- **UD-** prefix (Unsloth Dynamic) — different layers quantized differently, protecting the ones
  that matter. Holds up notably better than a flat quant of the same size.
- **MXFP4** — 4.25 bpw microscaling float.

**Rule of thumb:** the model file plus your KV cache must fit in VRAM with ~1-2GB spare. Work
backwards from your card.

### The quality cost is real but hard to see

IQ3_XXS at ~3.2 bpw against MXFP4's ~4.25 is a genuine drop. Unsloth's dynamic quants benchmark
within ~0.6 points of full precision on a 750-prompt suite, but that was at a different model
size. **If output quality degrades noticeably, the quant is the first suspect** — and it will not
announce itself. Note the tradeoff when you make it so future-you can reverse it.

## MoE vs dense

A Mixture-of-Experts model (Qwen3.6-35B-A3B: 35B total, **3B active**) only runs a few experts
per token. You pay VRAM for all 35B of weights but compute for 3B.

**Why that wins on a small card:** decode speed tracks *active* parameters, so a 35B MoE decodes
roughly like a 3B dense model while reasoning far better. And `--n-cpu-moe` gives you a knob
dense models don't have — expert layers can be pushed to host RAM at a graceful cost, because
any given token only needs a few of them.

The tradeoff is VRAM: 35B of weights must live *somewhere*. A dense 14B at Q5_K_M fits entirely
in 16GB and gives instant first tokens and faster decode, at the cost of capability. The
reference box weighed this explicitly and kept the MoE.

## KV cache math — the hidden VRAM cost

The KV cache grows linearly with context and is charged against the same VRAM as the weights:

```
bytes/token ≈ 2 (K and V) × n_layers × n_kv_heads × head_dim × bytes_per_element
```

For Qwen3.6-35B-A3B (41 layers, **2 KV heads**, head_dim 256):

| Precision | Per token | 64k context |
|---|---:|---:|
| f16 | 82 KiB | ~5.1 GiB |
| q8_0 | 41 KiB | ~2.56 GiB |
| q4_0 | ~20 KiB | ~1.3 GiB |

**Two KV heads is unusually cheap** — most 16GB-class models cost 80-128 KiB/token. That is
precisely why large contexts fit here. When comparing models for long-context work, check
`n_kv_heads`, not just parameter count.

Quantizing the KV cache (`--cache-type-k/v q4_0`) roughly halves it again. Quantized V **requires
`-fa on`**.

## Where the context ceiling really is

Every model has a native RoPE maximum (128k for Qwen3). Beyond it you need RoPE scaling, which
costs quality. But you will usually hit a *speed* wall first — prefill throughput degrades with
depth, measured on the reference box at 845 t/s @2.7k → 976 @30k → **375 t/s @71k**. A 71k prompt
took 190 seconds.

So pick your context for the *fast band*, not the theoretical max. This is also why a compiled
wiki of a few short notes beats stuffing many RAG chunks: fewer tokens, faster band.

## Next

- [03-tuning-and-benchmarking.md](03-tuning-and-benchmarking.md) — flags, and how to prove a
  change helped
