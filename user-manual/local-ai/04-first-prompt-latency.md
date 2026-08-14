# First-prompt latency: 32.8s → 0.3s

Investigation and fix, 2026-08-14. Client is Hermes Agent on the Windows host; server is the
`qwen` container in WSL. Every number here was measured on this box from `docker logs qwen`.

## Result

| Stage | Prompt tokens | Prefill | Wall clock |
|---|---:|---:|---|
| Starting point | 24,104 | 735 t/s | **32.8 s** |
| Tools trimmed | 16,254 | 716 t/s | 22.7 s |
| `-np 2`, `-c 131072` | 16,254 | 1,448 t/s | ~11 s |
| Title generation off | **11** | cache hit | **0.3 s** |

The last row is the real fix. The first three shrink the work; the last one stops the work from
being thrown away.

## The four causes, in the order they were found

### 1. Cold model load (~30 s) — a red herring after the first fix
Loading the 14 GB IQ3_XXS GGUF takes ~30 s cold, ~7 s with a warm page cache. This is paid only
when the container actually restarts. `docker inspect qwen --format '{{.State.StartedAt}}'`
settles it in one command. It was the cause on the very first observation and never again.

### 2. Tool definitions dominated the prompt (15,038 of 24,104 tokens)
Hermes' `/context` breaks the prompt into System prompt / Tool definitions / Rules / Skills /
MCP / Subagent definitions / Memory / Conversation. Tool definitions were 62%.

**Skills were not the problem** — the skills index is only each SKILL.md's `description:`
frontmatter, ~1,075 tokens for all 77 skills, and it did not even appear as a line item. An
early guess that trimming skills would help was wrong; measuring killed it.

Per-tool schema sizes were extracted from the literals in `hermes-agent/tools/*.py` and summed
to ~15,166 against Hermes' reported 15,038 — accurate to ~1%. The expensive ones:
`cronjob` 2,684 · `session_search` 2,151 · `delegate_task` 1,556 · `skill_manage` 1,526.

Disabled via `/tools disable <name> <name> …` (per-tool, no need to drop whole toolsets):
`cronjob`, `delegate_task`, `text_to_speech`, `process`, `computer_use`, `browser_exec`.
Kept `session_search` and `skill_manage` by preference.

24,104 → 16,254. Two bonuses that were not predicted: the System prompt shrank too
(6,432 → 5,093 — the prose describing tools goes with the tools), and the entire
**Subagent definitions** line (1,442) vanished once `computer_use` and `browser_exec` were gone,
because there was less left to describe delegating.

### 3. `-np 1` was throttling prefill — the MTP rule was stale
CLAUDE.md said `-np 1` is required by MTP speculative decoding. That was true when MTP was first
set up here; it is not true on image build 9870. With `-np 2`:

```
n_slots = 2, n_ctx_slot = 65536, kv_unified = 'false'
draft acceptance = 0.81111 (73 accepted / 90 generated), mean len = 2.62
```

MTP still works, and acceptance is *better* than the 0.56 the single-slot config was getting.
Solo prefill went 744 → 1,448 t/s and decode 60-64 → ~81-89 t/s.

`kv_unified` flips to `false`, so the pool is partitioned per slot rather than shared. `-c 65536`
would give 32,768 per slot, so `-c 131072` is needed to keep a real 64k window. That costs VRAM:
free memory sits at 1,508 MiB.

`--cache-reuse` is silently dropped in this mode: `cache_reuse is not supported by this context,
it will be disabled`. Prefix matches are all-or-nothing now.

### 4. Session title generation was destroying the cache — the actual bug
Hermes fires a second model call on the first message of every session to name it
(`agent/title_generator.py`, ~252 tokens, "cheap/fast tier", thinking disabled). That is free
when a small side model exists. **There is one model on one GPU here**, so it hit the same
llama-server, and it did damage differently depending on slot count:

- `-np 1` — the titler took the single slot first and **evicted** the cached prefix. Logs show
  `selected slot by LRU` on the 252-token task, then `by LRU` again on the real 24k prompt:
  no match, full re-prefill.
- `-np 2` — the titler got its own slot and ran **concurrently**, stealing throughput.
  Prefill fell from 1,448 t/s solo to 716 t/s alongside.

Either way, ~10-20 s spent naming a session. Turned off in `config.yaml`:

```yaml
auxiliary:
  title_generation:
    enabled: false
```

Resolved by `_auto_title_enabled()` reading `auxiliary.title_generation.enabled`, default true.
Cost of disabling: sessions get a plain fallback name. `session_search` is unaffected — it
searches session content, not titles.

With nothing evicting or competing, the prefix survives between sessions:

```
selected slot by LCP similarity, sim_best = 1.000, f_keep = 0.998
prompt eval time = 293.71 ms / 11 tokens
```

## How to read the logs

The single most diagnostic line is how llama-server picked its slot:

- `selected slot by LCP similarity, sim_best = 1.000` — prefix cache **hit**. Only the new
  tokens get prefilled.
- `selected slot by LRU` — **no match**. Full re-prefill. Something evicted the cache, or this
  is genuinely the first prompt after a restart.

```bash
docker logs qwen 2>&1 | grep -E "prompt eval time|selected slot by" | tail -8
```

A small task (a few hundred tokens) appearing immediately before a large one, both `by LRU`, is
the signature of an auxiliary call clobbering the cache.

## Warm cache across restarts — `warm.sh`

`--slot-save-path /slots` lets slot 0's KV cache be serialised (153 MB at 16k tokens) and
restored in under a second, so a warm prefix survives a container restart:

```bash
~/llamacpp-setup/warm.sh save      # snapshot slot 0 after a real session has started
~/llamacpp-setup/warm.sh restore   # on boot; automated by qwen-warm.service
```

Automated by the systemd **user** unit `~/.config/systemd/user/qwen-warm.service`, which fires
at login, waits for `/health`, sends a throwaway generation to compile CUDA graphs, then
restores `slots/warm.bin`.

**Re-run `warm.sh save` whenever the system prompt changes** — a tool enabled or disabled, a
skill list edited, a Hermes upgrade. The saved prefix is that exact prompt.

Note this only matters across *restarts*. Within a running server, the live slot cache does the
same job for free — which is why disabling the titler mattered more than any of this.

## Lessons worth keeping

- **Measure the breakdown before cutting.** The first guess (skills) was 4% of the problem. The
  actual culprit was a feature nobody thinks of as a cost.
- **A "cheap" auxiliary model call is not cheap when you have one model.** Anything that assumes
  a small side model — titling, compression, summarisation, embeddings — competes with the main
  request on this box. Check `auxiliary.*` in config.yaml before blaming the server.
- **Cache preservation beats raw throughput.** Trimming 8k tokens of tools saved 10 s. Keeping
  the cache saved 22 s and made the trimming almost irrelevant to latency (it is still worth it
  for the context budget).
- **Re-test documented constraints after upgrades.** The `-np 1` rule cost nearly 2x prefill and
  was simply out of date.
