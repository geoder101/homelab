# Default coding-agent profile

## Selection: Qwen3.8-27B UD-Q4_K_XL precision profile

The default is `unsloth/Qwen3.8-27B-GGUF`, `UD-Q4_K_XL`, served by
llama.cpp b11960 on the RTX 5090. Artifact revisions and image digests are
in `config/models.lock.yaml` and `config/images.lock.yaml`; the external
froggeric template is pinned in `config/chat-templates.lock.yaml`.

The [2026-11-05 Ornith comparison](benchmarks/ornith-20261105.md) found matching
small-suite coding/tool pass rates and much faster inference, but actual
OpenCode read-only tasks exposed instruction-following and path regressions.
Promotion was declined and Qwen precision restored. `make profile-precision`
selects this profile; `make profile-capacity` selects the Q3/three-slot tradeoff
described in [capacity tuning](concurrency.md). `make profile-ornith` is experimental.

| Setting | Default |
| --- | --- |
| API model | `coder-medium`; `coder` remains compatible |
| Backend | `llama-coder.service` |
| Context | 262,144 tokens per slot; 524,288 aggregate, no YaRN |
| Active request slots | 2 fixed slots; additional requests queue |
| Attention K/V cache | GPU `q4_0` / `q4_0`, Flash Attention; no unified pool |
| RAM prompt cache | Up to 16,384 MiB, plus other host/runtime state |
| Default alias | `coder-medium`; soft effort steering plus hard thinking ceilings |
| Historical thinking preservation | Pinned froggeric v22.5 retains supplied reasoning and normalizes duplicate inline history |
| Reasoning ceilings | off: 0; low: 2,048; medium/legacy: 6,144; xhigh: 12,288 |
| MTP | Disabled |
| Thinking sampling | temperature 1.0, top-p 0.95, top-k 20, min-p 0, presence penalty 0, repetition penalty 1 |
| OpenCode output cap | 16,384 tokens, including reasoning |
| Vision projector | GPU; verified with a local image request through LiteLLM |

Sampling follows the [Unsloth Qwen3.8 guide](https://unsloth.ai/docs/models/qwen3.8).
`coder-off` uses .7 temperature, .8 top-p and 1.5 presence penalty. The selected
template supports soft effort steering, supplied reasoning history and its
consecutive-tool-error warning. The RAM prompt cache is reusable state, not
active-KV spill. Ornith's experimental history-fixed template and its limits
are documented with the trial results.

## Historical tradeoff, 2026-10-29 (Q4 weights, original embedded template)

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

GGUF won the historical single-agent comparison because it combined full native 256K
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
- **Medium is the default reasoning balance.** Select low/medium/xhigh aliases
  for soft prompt steering and their respective hard thinking ceilings.
- **Qwen's historical thinking preservation changed on 2026-10-31.** Its template
  retains supplied reasoning and fixes duplicated in-content think blocks.
  This grows context; Home's thinking aliases declare `interleaved.field` as
  `reasoning_content` so the client can actually resend that history. Earlier
  benchmarks above used preservation disabled. See [reasoning profiles](reasoning-profiles.md).
- **No MTP in this profile.** It is a separate optimization requiring memory
  headroom and measurement. It was not part of this comparison.
- **16K output is a ceiling, not a target.** It reserves room for reasoning and
  final answers inside the same 256K total budget. The old 4K evaluation budget
  truncated a Qwen3.6 thinking response; the selected profile has more room.

## Client configuration

For each OpenCode custom `brain/coder` model:

```jsonc
"name": "Local coder (brain)",
"reasoning": true,
"tool_call": true,
"limit": { "context": 262144, "output": 16384 }
```

Keep these limits explicit: OpenAI-compatible model-list endpoints do not
standardize context metadata, and custom providers cannot be assumed to inherit
the correct model-catalog limits. Quit and restart OpenCode after updating them.

Historical validation on 2026-10-29 read source and tests, edited only the
implementation, ran `python3 -m unittest -v`, and completed through LiteLLM.
A separate 252,996-token request passed retrieval in 208.39 seconds. These
results predate the fixed template and do not validate Home's current runtime.
The 2026-10-31 handoff prepares offline Home files in Brain's Downloads only;
see [the report and Home verification commands](reasoning-profiles.md).

For the alternative 160K NVFP4 profile and persistent startup changes, see
`docs/operations.md`. Never start both GPU backends together.
