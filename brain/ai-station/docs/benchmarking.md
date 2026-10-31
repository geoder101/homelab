# Coding Model Benchmark

For the current fully local concurrency experiments, use
[capacity tuning](concurrency.md) and `scripts/tune-concurrency`. The original
comparison procedure below remains useful for historical single-agent results.

Choose coding profiles empirically rather than treating parameter count or maximum advertised context as quality. The current comparison and selected Qwen3.8 profile are recorded in `docs/benchmarks/README.md`, `docs/benchmark-results.md` and `docs/model-profile.md`.

## Constraints

- GPU: RTX 5090, 32 GiB VRAM.
- One base model is active at a time.
- Target workload: OpenCode software-engineering tasks, tool calls, Greek and English prompts.
- Original July baseline: 32K context and 90% GPU-memory utilization. The current target is one long-running agent with useful cache reuse and at least 128K context.

## Candidate selection

Compare Blackwell-oriented NVFP4/vLLM and Dynamic GGUF/llama.cpp when they offer different memory/performance tradeoffs. Record the exact Hugging Face repository, commit revision, quantization format, backend image digest, and server arguments. Do not use an unpinned `main` revision.

### Original candidate 1 (historical)

| Field | Value |
| --- | --- |
| Repository | `cyankiwi/Qwen3-Coder-30B-A3B-Instruct-AWQ-4bit` |
| Revision | `2831070b7b8c7aa6b7012333c6c4a2bd257f6cdf` |
| Format | `compressed-tensors` AWQ 4-bit |
| Stored size | about 18 GiB |
| Why first | Leaves meaningful KV-cache headroom on 32 GiB VRAM while retaining the Qwen3 Coder 30B-A3B base model. |

This is a third-party quantization, not an official Qwen release. It is a benchmark candidate, not an architectural commitment. Compare it later with a Blackwell-oriented NVFP4 candidate after this baseline is stable.

## Measurements

For each candidate:

1. Start one backend at a time with comparable sampling, reasoning effort and concurrency. Record backend-specific context/cache settings rather than assuming flags are interchangeable.
2. Record startup success, VRAM usage, and usable context length.
3. Run the same short and long OpenCode tasks three times.
4. Record time to first token, generated tokens per second, completion quality, tool-call reliability, and any CUDA/vLLM failures.
5. Keep artifacts in revision-specific model directories, update `config/models.lock.yaml`, and preserve the stable API alias `coder`.

## Acceptance criteria

- No out-of-memory error at the required context length.
- Stable API streaming through LiteLLM and Open WebUI.
- Useful coding quality on the real repositories used with OpenCode.
- Sufficient throughput for interactive agent use.
