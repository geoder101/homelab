# Benchmark Results

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
