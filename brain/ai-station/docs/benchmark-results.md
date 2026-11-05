# Benchmark Results

## 2026-11-05: Ornith Q4_K_M trial — promotion declined

Compared the requested Ornith-1.5-35B-A3B Q4_K_M with the actual deployed
Qwen3.8 UD-Q4_K_XL precision profile, two fixed 256K slots. Ornith measured
229.94 versus 71.665 tok/s warm short decode; both passed 21/21 coding cases,
9/9 nested tool round trips and 6/6 repair fixtures. Two 253K requests finished
in 109.72 versus 414.24 seconds.

Native OpenCode checks changed the decision: Ornith passed 3/6 read-only
sessions, repeatedly choosing a subheading and once requesting an incorrect
absolute path. Qwen passed 3/3 initially and 3/3 again after rollback on the
same updated documents. The upstream Ornith template also required a pinned
duplicate-history fix before API checks passed. **Qwen precision was restored
and remains the default.** See the [full report and artifacts](benchmarks/ornith-20261105.md).

The comparisons below are historical. Qwen's three-slot capacity alternative
and measured cache tradeoffs are recorded in [capacity tuning](concurrency.md).

## 2026-10-29: Qwen3.8 NVFP4 versus GGUF

Selected **Qwen3.8-27B Dynamic V3 UD-Q4_K_XL / llama.cpp**, native 256K
context, Q8 K/V, one request slot, medium reasoning, no MTP. See
`docs/model-profile.md` for the full settings, tradeoff and limitations.

| Measurement | NVFP4 / vLLM | GGUF / llama.cpp |
| --- | ---: | ---: |
| Median short-prompt decode | 68.21 tok/s | 72.84 tok/s |
| Fresh ~154K prompt | 46.90 s | 88.42 s |
| Follow-up over ~154K history | 0.459 s | 0.487 s |
| Fresh ~253K prompt | Not supported in memory | 203.548 s |
| Follow-up over ~253K history | — | 0.653 s |
| Coding contracts / nested tools | 2/2 and 3/3 passed | 2/2 and 3/3 passed |

Raw artifacts: `docs/benchmarks/qwen38-nvfp4.json` and
`docs/benchmarks/qwen38-gguf-q4-k-xl.json`. The Qwen3.6 baseline is also
recorded, but used an initial 4096-token output budget and predates the
follow-up-turn probes. Its topological-sort generation was truncated; this
is not an apples-to-apples quality failure versus the later 8192-token runs.

Even with max sequences 1, Qwen3.8 NVFP4 256K startup required 8.33 GiB KV
cache versus 5.64 GiB available. At 160K, its measured cache capacity was
174,172 tokens and available cache memory 5.69 GiB.

After activating the selected production profile:

- A 252,996-input-token request through LiteLLM passed exact retrieval in
  208.39 seconds, with 49 output tokens.
- Chat completion, tool round trip and separate reasoning checks passed.
- Open WebUI listed `coder` through LiteLLM.
- Vision projector request through LiteLLM correctly identified a red image.
- Actual OpenCode `brain/coder` session read source/tests, edited the
  implementation, ran the existing unittest successfully and reported success.
- `llama-coder`, LiteLLM and Open WebUI active with zero restarts;
  optional `vllm-coder` inactive.

The raw suite is a small local functional/performance comparison, not a
SWE-bench result or proof of equivalent quantization quality.

## 2026-10-28: Explicit FP8 KV and 160K context validation

- Confirmed that the checkpoint already specifies FP8 KV quantization and contains K/V scales. Previous `auto` runs also used FP8 KV, not BF16.
- Explicit `--kv-cache-dtype fp8_e4m3`; context 163,840 tokens; utilization 0.94; max sequences 8; no CPU offloading.
- Native 262,144-token startup failed: 8.33 GiB cache required versus 5.56 GiB available; estimated feasible maximum was 172,480 tokens.
- At 160K: 5.61 GiB available cache, capacity 172,696 tokens, estimated full-length concurrency 1.05x.
- LiteLLM request with 153,996 prompt tokens and 49 output tokens passed exact start/middle/end retrieval in 46.30 seconds. Synthetic source context, non-thinking, temperature 0; not a coding-quality benchmark.
- Local OpenCode limit updated to 163,840; output cap remains 8192.

## 2026-10-28: 128K context validation

- vLLM context raised to 131,072 tokens; GPU memory utilization raised from 0.90 to 0.94.
- KV cache was configured as `auto`, which selected the checkpoint's FP8 KV scheme; no CPU offloading or RoPE overrides. The original BF16 description was incorrect.
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
