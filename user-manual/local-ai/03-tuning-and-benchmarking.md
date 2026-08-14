# 03 — Tuning and benchmarking

## Rule zero: benchmark everything, trust nothing

**On WSL there is no OOM error.** The WDDM driver silently spills GPU memory into host RAM
instead of failing. A config that is badly over-budget loads cleanly, reports no error, and runs
2.7x slower. Measured on the reference box: `--n-cpu-moe 15` looked fine and delivered 357 t/s
where 20 delivered 976.

So **a successful load proves nothing**. Every change needs a number.

```bash
# prefill + decode at ~16k depth
python3 - <<'EOF'
import json,urllib.request
b='alpha beta gamma delta epsilon zeta eta theta iota kappa '*1450
d=json.dumps({'messages':[{'role':'user','content':'Reply one word. '+b}],'max_tokens':64}).encode()
r=urllib.request.urlopen(urllib.request.Request('http://localhost:8090/v1/chat/completions',
    data=d,headers={'Content-Type':'application/json'}),timeout=600)
t=json.load(r)['timings']
print('prompt_n',t['prompt_n'],'prefill',round(t['prompt_per_second'],1),
      't/s  decode',round(t['predicted_per_second'],2),'t/s')
EOF

nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

Send a throwaway prompt first — the very first request after startup pays CUDA graph compilation.

Watch for silent spilling: `nvidia-smi` free memory near zero *plus* a jump in major page faults.

```bash
cut -d' ' -f12 /proc/$(pgrep -f llama-server)/stat
```

## `--n-cpu-moe` — experts on CPU vs GPU

MoE only. **Lower = more experts on GPU = more VRAM used.** Higher = more in host RAM.

**Lower is not automatically faster.** Prefill needs free VRAM for compute buffers; starve it and
throughput collapses. The reference box measured, at `--n-cpu-moe 20` with a 21GB model:

| Config | VRAM free | Prefill @30k | Decode |
|---|---:|---:|---:|
| moe 20, `-c 131072` | 1.1 GB | 794 t/s | 33-35 t/s |
| moe 20, `-c 65536` | 2.5 GB | **976 t/s** | ~32 t/s |
| moe 15, `-c 65536` | **0.35 GB** | **357 t/s** ❌ | 12.8-23.5 t/s ❌ |

The lesson generalizes even though the numbers don't: **to buy speed, free VRAM by reducing
context — don't relocate experts.** Dropping 128k → 64k gained 23% prefill at the same
`--n-cpu-moe`.

Each expert layer is ~0.6GB of VRAM here, so each ±1 moves ~0.6GB.

### The headroom "rule" and its limits

That table produced a rule — *keep ≥2GB VRAM free* — which was then treated as a hard constraint.
It isn't. After switching to a quant that fits (`--n-cpu-moe 4`), the box runs at **1,508 MiB
free with prefill at 1,448 t/s**: below the "floor", no degradation whatsoever.

The rule was real *under heavy CPU offload*, where compute buffers had to be large. It did not
survive the conditions changing. **Re-derive thresholds after any major config change** rather
than inheriting them.

## Slots (`-np`) — the one that surprised us

`-np N` sets how many requests the server handles concurrently. The consequences run deeper than
concurrency:

- **`-np 1`** — one slot, unified KV pool. Any background request (a client's title generator,
  a health probe) takes the slot and **evicts** your conversation's cached prefix. Next real
  prompt re-prefills from scratch.
- **`-np 2`** — `kv_unified` flips to `'false'` and the pool is **partitioned**, so `-c` is
  divided among slots. `-c 131072` with 2 slots = 65,536 each.

Measured on the reference box, solo:

| | `-np 1` | `-np 2` |
|---|---:|---:|
| Prefill @16k | 744 t/s | **1,448 t/s** |
| Decode | 60-64 t/s | **81-89 t/s** |

Nearly 2x prefill for a flag that looks like it only governs concurrency.

Caveat: `--cache-reuse` is unsupported with a partitioned pool — the log says
`cache_reuse is not supported by this context, it will be disabled`. Prefix matches become
all-or-nothing.

## Speculative decoding (MTP)

If the model ships Multi-Token Prediction weights, `--spec-type draft-mtp` drafts several tokens
per step and verifies them in one pass.

```
--spec-type draft-mtp --spec-draft-n-max 2
```

Check it is working — the server reports acceptance per request:

```
draft acceptance = 0.81111 (73 accepted / 90 generated), mean len = 2.62
```

Above ~0.7 is healthy. `--spec-draft-n-max 3` dropped acceptance 81% → 70% here for no gain;
drafting deeper wastes compute when the guesses miss.

**A documented constraint worth re-testing:** this box recorded "`-np 1` is REQUIRED with MTP".
That was true when MTP was first set up and false a build later — and it was costing 2x prefill.
Constraints expire.

## The prefix cache is worth more than throughput

llama-server caches the KV state of each slot and reuses any matching prefix. Since a client's
system prompt is identical every session, a hit means near-zero prefill.

**The single most diagnostic line in the logs:**

```bash
docker logs qwen 2>&1 | grep -E "prompt eval time|selected slot by" | tail -8
```

- `selected slot by LCP similarity, sim_best = 1.000` — **hit**. Only new tokens prefill.
- `selected slot by LRU` — **miss**. Full re-prefill; something evicted the cache.

A small task appearing immediately before a large one, both `by LRU`, is the signature of a
background call clobbering the cache. Chasing that pattern took a first prompt from 32.8s to
0.3s — see [04-first-prompt-latency.md](04-first-prompt-latency.md).

Arithmetic for trade-offs: losing a 51k cached prefix costs ~64s of re-prefill, while the
prefill-rate difference between two context sizes costs ~2s on a typical delta. **Protecting the
cache is worth ~30x more than peak prefill rate.**

## Warm cache across restarts

`--slot-save-path` lets a slot's KV cache be serialised (153MB at 16k tokens) and restored in
under a second:

```bash
curl -X POST "http://localhost:8090/slots/0?action=save"    -d '{"filename":"warm.bin"}'
curl -X POST "http://localhost:8090/slots/0?action=restore" -d '{"filename":"warm.bin"}'
```

Automate the restore at boot. **Re-save whenever the system prompt changes** — a tool toggled, a
client upgrade. Note this only helps across *restarts*; within a running server the live cache
does the same job for free, which is why stopping evictions matters more.

## Checklist for any change

1. Note the current prefill/decode/VRAM numbers.
2. Change **one** thing.
3. Throwaway prompt (CUDA graphs), then benchmark.
4. Check `nvidia-smi` free memory and the startup `n_slots` line — confirm you got what you asked
   for.
5. Write down the number, not just the conclusion. Conclusions expire; measurements don't.
