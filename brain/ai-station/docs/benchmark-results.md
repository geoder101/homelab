# Benchmark Results

## 2026-10-28: 128K context validation

- vLLM context raised to 131,072 tokens; GPU memory utilization raised from 0.90 to 0.94.
- KV cache remains BF16 (`auto`); no CPU offloading or RoPE overrides.
- Available KV cache: 5.61 GiB, 170,393 tokens; estimated full-length concurrency: 1.30x.
- Whole-GPU memory after the long-context test: 30,088 MiB of 32,607 MiB.
- A request through LiteLLM used 121,996 prompt tokens and 49 output tokens, completing in 40.57 seconds. Non-thinking, temperature 0.
- Exact retrieval of three configuration values from the start, middle and end of synthetic source context passed. This is a functional long-context check, not a coding-quality benchmark.
- Local OpenCode context limit updated to 131,072; output cap remains 8192.
- Automatic tool calling, a tool-result round trip and thinking mode also passed with the 128K configuration. vLLM remained active with zero restarts.
- LiteLLM restarted to refresh cached model metadata; `/v1/models` now advertises 131,072 tokens.

## 2026-10-28: Qwen3.6 upgrade validation

| Field | Result |
| --- | --- |
| Model | `unsloth/Qwen3.6-27B-NVFP4` |
| Immutable revision | `ccdaab7e68af2409599b8949a8f2685703c9bae5` |
| vLLM / LiteLLM / Open WebUI | `0.32.0` / `1.106.5` / `0.12.4` |
| FlashInfer / CUTLASS DSL | `0.7.0.post1` / `4.8.0` |
| Context / max sequences | 32,768 tokens / 8 |
| Model VRAM during load | 21.22 GiB |
| Steady GPU memory after smoke checks | 28,487 MiB of 32,607 MiB (whole GPU) |
| KV cache | 4.37 GiB, 110,440 tokens |
| Estimated full 32K concurrency | 3.37 requests |
| NVFP4 kernel | `FlashInferCutlassNvFp4LinearKernel` |

Functional validation passed:

- All three upgraded systemd services active, with zero restarts after the final configuration.
- Chat completion through LiteLLM on both loopback and `192.168.1.11:4000`, returning `brain inference ready`.
- Automatic tool selection with parsed JSON arguments, followed by a successful tool-result round trip.
- Thinking mode produced separate reasoning and the correct final answer.
- Open WebUI health check and internal authenticated model listing through LiteLLM (`coder`).
- LAN liveness confirmed by the user from a second device; requests to `/v1/models` without a key returned HTTP 401.
- Quadlet generation, shell syntax checks, and `git diff --check`.

The default 512-sequence setting exceeded VRAM during CUDA graph profiling. Capping it at 8 fixed startup while retaining the 32K context. These are functional and memory checks, not throughput or coding-quality benchmark results.

Open WebUI data was backed up before startup to `~/.local/share/ai-station/open-webui-backup-20261028.tar.gz` (mode 0600).

## 2026-07-12: Candidate 1 baseline

| Field | Result |
| --- | --- |
| Model | `cyankiwi/Qwen3-Coder-30B-A3B-Instruct-AWQ-4bit` |
| Immutable revision | `2831070b7b8c7aa6b7012333c6c4a2bd257f6cdf` |
| vLLM | `v0.25.0`, image digest in `config/images.lock.yaml` |
| Model files | 17 GiB on disk |
| Context setting | 32,768 tokens |
| Startup to API ready | about 75 seconds |
| Model VRAM | 16.95 GiB during model load |
| Total steady VRAM | 29.65 GiB |
| Free VRAM | 2.46 GiB |
| KV cache | 9.68 GiB, 105,696 tokens |
| Estimated 32k concurrency | 3.23 requests |

### Functional checks

- vLLM served the OpenAI-compatible API successfully.
- LiteLLM routed the stable alias `coder` successfully.
- Open WebUI reached LiteLLM through the internal Podman network and received `coder` from `/v1/models`.
- A deterministic text completion returned `brain inference ready`.
- Automatic function calling with `qwen3_xml` returned a structured `add(a=2, b=3)` tool call.

### Interpretation

The candidate is a viable Phase 1 baseline: it supports the required context size and agent tool calls while retaining a small but usable VRAM safety margin. It is not yet a performance winner. Compare tokens per second and coding-task quality against a Blackwell-optimized NVFP4 candidate before treating it as the long-term default.
