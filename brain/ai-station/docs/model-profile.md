# Default coding-agent profile

## Selection: Qwen3.8-27B Dynamic V3 GGUF

The default is `unsloth/Qwen3.8-27B-GGUF`, `UD-Q4_K_XL`, served by
llama.cpp b11960 on the RTX 5090. Artifact revisions and image digests are
in `config/models.lock.yaml` and `config/images.lock.yaml`.

The priority is strong model quality with a practical context for long-running
coding agents, quick continuation of existing conversations, and useful decode
speed. This is not a multi-user throughput profile.

| Setting | Default |
| --- | --- |
| API model | `coder` |
| Backend | `llama-coder.service` |
| Context | 262,144 tokens (native 256K, no YaRN) |
| Active request slots | 1 |
| Attention K/V cache | GPU `q8_0` / `q8_0`, Flash Attention |
| Reasoning effort | `medium` |
| Historical thinking preservation | Disabled |
| MTP | Disabled |
| Thinking sampling | temperature 1.0, top-p 0.95, top-k 20, min-p 0, presence penalty 0, repetition penalty 1 |
| OpenCode output cap | 16,384 tokens, including reasoning |
| Vision projector | Loaded on GPU; verified with an image request through LiteLLM |

The sampling and reasoning controls follow the
[Unsloth Qwen3.8 guide](https://unsloth.ai/docs/models/qwen3.8).
The much larger 2.4T variant is outside this workstation's memory capacity.
The guide's B200 throughput is not used as an RTX 5090 prediction.

## Measured tradeoff, 2026-10-29

Both candidates used medium reasoning, no MTP, one active request, the same
prompts and sampling settings, and an 8192-token budget for thinking tests.
Generated Python was checked with edge-case tests. Nested tools were tested
at three seeds. Raw results are in `docs/benchmarks/`.

| Measurement | NVFP4 / vLLM 0.32.0 | Dynamic Q4_K_XL / llama.cpp b11960 |
| --- | ---: | ---: |
| Short-prompt decode, median of three 512-token runs | 68.21 tok/s | 72.84 tok/s |
| First streamed-text latency, first probe | 2.257 s | 2.278 s |
| First streamed-text latency, subsequent probes | 0.036–0.037 s | 0.069–0.125 s |
| Fresh ~32K prompt + short answer | 5.07 s | 9.25 s |
| Fresh ~122K prompt + short answer | 31.96 s | 60.55 s |
| Fresh ~154K prompt + short answer | 46.90 s | 88.42 s |
| Follow-up turn over ~154K history | 0.46 s | 0.49 s |
| Fresh ~253K prompt + short answer | Does not fit | 203.55 s |
| Follow-up turn over ~253K history | Does not fit | 0.65 s |
| Generated-code contracts | 2/2 passed | 2/2 passed |
| Nested tool round trips | 3/3 passed | 3/3 passed |
| Whole-GPU memory sampled during suite | 29,587 MiB | 27,921 MiB |

GGUF wins this single-agent deployment because it combines full native 256K
context, good cache reuse, somewhat faster short-context decoding in these
measurements, and more GPU memory headroom. NVFP4 remains valuable when cold
long-prompt ingestion dominates: its prefill was approximately twice as fast.

The synthetic retrieval test checks that a large prompt is accepted and that
three values can be recovered from its start, middle and end. It does not prove
coding quality at 256K. Two generated-code problems and three tool seeds are
functional checks, not a broad quality benchmark. Different quantizations and
backend random samplers can produce different reasoning lengths even at the
same seed; shorter reasoning is not treated as evidence of greater quality.

## Why these limits and controls

- **256K is capacity, not the desired size of every prompt.** Smaller prompts
  remain cheaper to prefill. OpenCode still needs compaction for extended tasks.
- **Medium is the default reasoning balance.** The model's default `xhigh`
  can use substantial output and time; low/medium/xhigh can be selected via
  request `chat_template_kwargs.reasoning_effort` when appropriate.
- **No automatic historical-thinking preservation.** Preserving all reasoning
  may help some tasks, but grows the context. The baseline avoids this cost;
  tool results and normal conversation history are still retained.
- **No MTP in this profile.** It is a separate optimization requiring memory
  headroom and measurement. It was not part of this comparison.
- **16K output is a ceiling, not a target.** It reserves room for reasoning and
  final answers inside the same 256K total budget. The old 4K evaluation budget
  truncated a Qwen3.6 thinking response; the selected profile has more room.

## Client configuration

For each OpenCode custom `brain/coder` model:

```jsonc
"name": "Qwen3.8 27B Dynamic Q4_K_XL (brain)",
"reasoning": true,
"tool_call": true,
"limit": { "context": 262144, "output": 16384 }
```

Keep these limits explicit: OpenAI-compatible model-list endpoints do not
standardize context metadata, and custom providers cannot be assumed to inherit
the correct model-catalog limits. Quit and restart OpenCode after updating them.

The local global config has been updated. End-to-end OpenCode validation read
source and tests, edited only the implementation, ran `python3 -m unittest -v`,
and completed successfully through LiteLLM. A separate 252,996-token request
through LiteLLM passed retrieval in 208.39 seconds.

For the alternative 160K NVFP4 profile and persistent startup changes, see
`docs/operations.md`. Never start both GPU backends together.
