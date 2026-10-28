# Coding Model Benchmark

Choose the first coding model empirically rather than treating parameter count as quality.

## Constraints

- GPU: RTX 5090, 32 GiB VRAM.
- One base model is active at a time.
- Target workload: OpenCode software-engineering tasks, tool calls, Greek and English prompts.
- Baseline server settings: 32k context window and 90% GPU-memory utilization.

## Candidate selection

Start with coding models that have an officially supported vLLM quantization for Blackwell. Record the exact Hugging Face repository, commit revision, quantization format, vLLM image digest, and server arguments. Do not use an unpinned `main` revision.

### Candidate 1

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

1. Start vLLM with the identical server arguments.
2. Record startup success, VRAM usage, and usable context length.
3. Run the same short and long OpenCode tasks three times.
4. Record time to first token, generated tokens per second, completion quality, tool-call reliability, and any CUDA/vLLM failures.
5. Preserve the winning model under `models/coder` and add its immutable revision to this document.

## Acceptance criteria

- No out-of-memory error at the required context length.
- Stable API streaming through LiteLLM and Open WebUI.
- Useful coding quality on the real repositories used with OpenCode.
- Sufficient throughput for interactive agent use.
