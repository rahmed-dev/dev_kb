# 01 — Setting up a local LLM server from scratch

Target: an OpenAI-compatible API on `localhost`, backed by your own GPU, with no cloud
dependency. Reference box is Windows 11 + WSL2 + RTX 5060 Ti 16GB, but only the WSL-specific
steps are unique to it.

## Why Docker, not a local build

llama.cpp needs to be compiled against CUDA, and a Blackwell card (sm_120) needs CUDA 12.8+.
Building that yourself means installing the CUDA toolkit, matching driver versions, and
rebuilding on every update. The prebuilt image skips all of it:

```
ghcr.io/ggml-org/llama.cpp:server-cuda        # ~7GB, llama-server only
ghcr.io/ggml-org/llama.cpp:full-cuda          # adds llama-cli for terminal chat
```

GPU passthrough into the container needs the **NVIDIA Container Toolkit** installed on the host.
Verify before anything else:

```bash
docker run --rm --gpus all ghcr.io/ggml-org/llama.cpp:server-cuda --version
nvidia-smi        # should show your card from inside WSL too
```

If `--gpus all` errors, the toolkit is missing or the Docker daemon needs a restart. Nothing
downstream will work until this passes.

## WSL-specific: memory sizing

WSL2 gets a fraction of host RAM by default. Loading a large GGUF spikes memory hard — on the
reference box, loading a 21GB model pushed Windows to 97% and froze the desktop for minutes.

`C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=14GB
swap=8GB
processors=8
```

Then `wsl --shutdown` in PowerShell and reopen. Files persist; only the VM restarts.

**`memory=` is a cap, not a reservation.** WSL consumes it during load and releases it after —
measured: 30.9GB/31.8GB during load, 20.2GB after. Setting it *lower* bounds how much Windows
can lose to that transient spike. Size it to what the model actually needs on the host side
(with most weights on GPU, that is small), not to what you can spare.

## Downloading the model

```bash
HF_HUB_DISABLE_XET=1 hf download <repo> <file.gguf> --local-dir ~/llamacpp-setup/models/<name>
```

**`HF_HUB_DISABLE_XET=1` matters** — Xet transfers stall in WSL. An `HF_TOKEN` speeds things up
but treat it as a secret; do not paste it into a chat log.

See [02-model-and-quant.md](02-model-and-quant.md) for choosing *what* to download. That choice
matters more than any flag in this document.

## Port choice

Map the container's 8080 to something else on the host if 8080 is taken (Frappe bench uses it on
the reference box). Pick once and keep it — every client config references it.

```
-p 8090:8080        # host 8090 → container 8080
```

## Launching

```bash
docker run -d --name qwen --restart unless-stopped --gpus all \
  -p 8090:8080 \
  -v ~/llamacpp-setup/models:/models \
  -v ~/llamacpp-setup/slots:/slots \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  -m /models/<name>/<file>.gguf \
  -ngl 99 --n-cpu-moe 4 \
  -c 131072 -fa on --jinja \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  -np 2 -b 512 -ub 256 \
  --slot-save-path /slots \
  --host 0.0.0.0 --port 8080
```

Flag by flag:

| Flag | Why |
|---|---|
| `--restart unless-stopped` | Auto-starts on Docker/WSL boot. Restarts re-pay the full model load, so never let it stay down. |
| `-ngl 99` | Offload all layers it can to GPU. |
| `--n-cpu-moe N` | MoE only: keep N expert layers in host RAM. See [03](03-tuning-and-benchmarking.md). |
| `-c` | Total KV pool. **With `-np 2` this is split per slot** — `-c 131072` gives 64k each. |
| `-fa on` | Flash attention. Required for quantized V cache. |
| `--jinja` | Use the model's own chat template. |
| `--cache-type-k/v q4_0` | Quantize the KV cache. Roughly halves KV memory vs q8_0. |
| `--spec-type draft-mtp` | Multi-Token Prediction speculative decoding, if the model ships MTP weights. Literally `draft-mtp`, not `mtp`. |
| `-np 2` | Two slots, so a background request can't evict your conversation's cache. |
| `--slot-save-path` | Enables the slot save/restore API for warm caches across restarts. |

## Verify

```bash
docker ps --filter name=qwen                                    # want "healthy"
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090/health   # want 200
docker logs qwen 2>&1 | grep -E "n_slots|model loaded"
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

The startup line tells you what you actually got, which is not always what you asked for:

```
n_slots = 2, n_ctx_slot = 65536, kv_unified = 'false'
```

Web UI at `http://localhost:8090`, OpenAI-compatible API at `http://localhost:8090/v1`.

**The first request after startup is slow** — CUDA graph compilation. Send a throwaway prompt
before benchmarking or your first number is garbage.

## Managing

```bash
docker start qwen | docker stop qwen | docker restart qwen
docker logs -f qwen
docker rm -f qwen                    # then re-run the full launch command
docker pull ghcr.io/ggml-org/llama.cpp:server-cuda    # update
```

`docker ps -a --filter name=qwen` shows exit codes. **Exit 137 = SIGKILL**, usually the WSL OOM
killer — lower `-c` or raise `--n-cpu-moe`.

## Next

- [02-model-and-quant.md](02-model-and-quant.md) — the choice that matters most
- [03-tuning-and-benchmarking.md](03-tuning-and-benchmarking.md) — making it fast
