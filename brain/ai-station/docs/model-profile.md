# Default coding-agent profile

## Current selected and active deployment — 2026-11-08

**Ornith-1.5-35B-A3B Q5_K_M on llama.cpp b12182** is selected persistently
as profile **`ornith-q5`**. The running model, arguments and image ID were
verified against the selected Quadlet with `make profile-status`.

This section is the latest verified deployment snapshot. Run `make profile-status`
for a live check. On future persistent selections, update this section and add a
dated history entry below; preserve the previous configuration and comparisons.

Deployment API/budget/history/vision/concurrency checks passed. Post-deployment
native probes exposed some heading interpretation and one incorrect-path
attempt on the reorganized documents; those failures are preserved in the
[verification notes](deployments/20261108-ornith-q5/README.md). The benchmark's
6/6 native result describes its earlier frozen inputs, not a guarantee for every
agent task. A 0.6-temperature pilot lost a coding edge case, so the deployed
temperature remains the measured 1.0 configuration.

### Identity and immutable pins

| Item | Current value |
| --- | --- |
| Profile / service | `ornith-q5` / `llama-coder.service` |
| Model repository | `ornith-ai/Ornith-1.5-35B-A3B-GGUF` |
| Model revision | `12393612fd4f730ff5aadc23e9b8f9648aa49ceb` |
| Weight file | `Ornith-1.5-35B-Q5_K_M.gguf` (~25.35 GB on disk) |
| Weight SHA-256 | `91df97de5845100e850b4b5ec5ff35695382020b880fad6f7f51787b3a953bd0` |
| Projector | `mmproj-Ornith-1.5-35B-BF16.gguf`, GPU-resident |
| Projector SHA-256 | `1921a36a85aee56cd2abd27f46701802c9d85a33474792e600df6c3b282a135d` |
| Engine | llama.cpp **b12182**, commit `fd27b2034905f8dadd8bd6d2cc44595f4929d391` |
| Engine image | `ghcr.io/ggml-org/llama.cpp@sha256:859efa314dc34f1843a38ff28a4818aeac51a4cce125bdea4d2678aceba95469` |
| Chat template | `ornith1.5-history-v1`, key `ornith_fixed` |
| Template upstream revision | `10fbf86fed7ecee4a061f8b499a618f46001cac1` |
| Patched template SHA-256 | `5d7c6dd4e14e15880d852536762c620e7e8c2fe54fcd9a998c6c99eca0b78e3b` |

The template is Ornith's pinned upstream template with a minimal duplicate
inline-thinking/history normalization patch. Both upstream and output hashes
are checked by `scripts/download-ornith-template`.

### Capacity, cache and reasoning

| Setting | Current configuration |
| --- | --- |
| Hardware | Brain, RTX 5090, 32 GiB VRAM |
| Per-agent context | **262,144 tokens**, shared by input, reasoning, images and output |
| Concurrent slots | **2 fixed slots**; further requests queue |
| Aggregate server context | `524288`; **not** a client context limit |
| GPU offload | `--n-gpu-layers 99`, GPU projector, Flash Attention on |
| GPU K/V cache | `q4_0` / `q4_0`; no unified KV pool |
| RAM prompt cache | `16384` MiB (16 GiB) reusable prompt states, not active-KV spill |
| MTP / speculative helper | Disabled |
| Default thinking sampling | temperature 1.0; top-p .95; top-k 20; min-p 0; presence penalty 0; repeat penalty 1 |
| Nonthinking alias sampling | temperature .7; top-p .8; presence penalty 1.5 |
| OpenCode output cap | **16,384 tokens**, including reasoning |
| History | Supplied `reasoning_content` is retained and duplicate inline history normalized |

LiteLLM is the client-facing gateway at `:4000/v1`; Open WebUI is at `:3000`.
The inference container's port 8000 is private to the Podman network.

| API ID | OpenCode ID | Hard thinking ceiling |
| --- | --- | ---: |
| `coder-off` | `brain/coder-off` | 0 |
| `coder-low` | `brain/coder-low` | 2048 |
| `coder-medium` | `brain/coder-medium` | 6144 |
| `coder-xhigh` | `brain/coder-xhigh` | 12288 |
| `coder` | `brain/coder` | 6144; legacy medium-compatible alias |

`coder-medium` is the default. Ornith's template does not implement Qwen
froggeric's soft effort steering or consecutive-tool-error warning. Select
the actual API aliases for their hard thinking ceilings. `max_tokens` still
limits the combined reasoning and final answer; requests may override defaults.

### Full llama.cpp server command

These are the active container's entrypoint/options, with line breaks added.
They are reproduced from `quadlets/llama-coder.container` and checked against
the live container; paths below are **inside the container**.

```bash
/app/llama-server \
  --model /models/coder/Ornith-1.5-35B-Q5_K_M.gguf \
  --mmproj /models/coder/mmproj-Ornith-1.5-35B-BF16.gguf \
  --alias coder \
  --host 0.0.0.0 --port 8000 \
  --n-gpu-layers 99 \
  --ctx-size 524288 --parallel 2 --no-kv-unified \
  --cache-ram 16384 \
  --flash-attn on --cache-type-k q4_0 --cache-type-v q4_0 \
  --jinja --reasoning-format deepseek \
  --chat-template-kwargs '{"reasoning_effort":"medium","preserve_thinking":true}' \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  --no-webui --log-colors off \
  --chat-template-file /templates/chat_template.jinja
```

The Quadlet mounts `~/.local/share/ai-station/models/ornith-1.5-35b-gguf-q5`
at `/models/coder`, and the locked Ornith template directory at `/templates`.
It uses NVIDIA CDI, the GPU-only SELinux exception, `Pull=never` and
`DefaultDependencies=false`. LiteLLM routing remains `http://llama-coder:8000/v1`.

### Client setup, verification and rollback

- [OpenCode settings](opencode.md): existing `brain/coder-*` clients retain
  their endpoint, API key, model IDs and 262144/16384 limits.
- [Deployed verification](deployments/20261108-ornith-q5/README.md): gateway
  inference, alias streaming, budgets, history, vision, concurrency and native-client evidence.
- [Full benchmark scorecard](benchmarks/moe-20261107-queue/SUMMARY.md): model and
  configuration comparisons; raw results remain alongside it.
- Declarative sources: `config/models.lock.yaml`, `config/coder-profiles.yaml`,
  `config/images.lock.yaml`, `config/chat-templates.lock.yaml` and the llama Quadlet.
- CPU failover: `quadlets/llama-fallback.container` serves as the `order: 2` failover target for every `coder-*` alias; routing and the GPU-swap procedure are documented in [operations.md](operations.md#cpu-warm-standby-fallback-backend).

```bash
make profile-status       # Selected model, live arguments and image match
make profile-ornith-q5    # Select this model AND engine persistently
make profile-qwen38-q4   # Roll back to Qwen3.8 Q4 AND llama.cpp b11960
make preflight           # Artifact hashes, image pins/presence, unit agreement
```

The model-specific selection commands are `profile-qwen38-q3`,
`profile-qwen38-q4`, `profile-ornith-q4` and `profile-ornith-q5`.
Earlier names (`profile-capacity`, `profile-precision`, `profile-ornith`) have
been removed. Historical entries below retain their original names; use the
model-specific replacements above when following those older instructions.

## Selection history

### 2026-11-08 — Ornith Q5 / b12182 promoted

The user selected this configuration after the expanded comparison. It passed
21/21 coding cases, 9/9 nested tool round trips, 6/6 repair fixtures, 6/6 native
OpenCode sessions and 6/6 vision checks, plus context/concurrency/recovery checks.
Warm generation measured 261.3 tok/s versus roughly 71.7 for the Qwen control;
the 21 coding cases took 231.39 versus 462.46 seconds. These are local test
results, not broad SWE-bench superiority claims. Q5 and the newer engine are
a distinct configuration from the earlier unsuccessful Ornith Q4 trial.

The deployment switch retained a rollback snapshot at
`~/.local/share/ai-station/backups/coder-profile-20261108T141841Z-660e611e`.
Qwen weights, template and b11960 image remain available. Client aliases and
context/output limits carry across the selection.

### Previous deployment, retired 2026-11-08 — Qwen Q4 precision

The following snapshot preserves the prior profile and its original comparisons.
Words such as "default" and "selected" inside this archived section refer
to that historical deployment.

| Historical pin | Value |
| --- | --- |
| Profile | `precision` |
| Repository / revision | `unsloth/Qwen3.8-27B-GGUF` / `4ca720788d1e01f1bff70c033e0d0028fd02e502` |
| Weight | `Qwen3.8-27B-UD-Q4_K_XL.gguf` |
| Weight SHA-256 | `3f227079003add2511437e5b1e94812e363385225bf6a9b47b0054a72bc8b01e` |
| Projector | `mmproj-BF16.gguf`; SHA-256 `83ee4f4f205fa514161778c41df1ea14144faa0f713510893b63c2395f5c2d53` |
| Engine | llama.cpp b11960, commit `8642541b255bfb5dc39cd351a178be09013a55d6` |
| Image | `ghcr.io/ggml-org/llama.cpp@sha256:ae676f2ecfdeb66db48b7c32f1f013c62cdbc3e380b1f6e8b425c981dd34ac99` |
| Template | froggeric v22.5; revision `855bffc49448e299789730ff92c9b8d834d6cc14` |
| Template SHA-256 | `e57684bae4156211a55473c5a63be976a405a37ab5be5ae0e5abf1df5349c4b2` |
| Capacity | Two fixed 262144-token slots, Q4 GPU K/V, GPU projector, 16 GiB RAM cache |
| Rollback | `make profile-qwen38-q4` restores its model, template and engine |

<details>
<summary>Retained exact Qwen precision server options</summary>

The pre-switch Quadlet recorded the following options (container paths):

```bash
/app/llama-server \
  --model /models/coder/Qwen3.8-27B-UD-Q4_K_XL.gguf \
  --mmproj /models/coder/mmproj-BF16.gguf \
  --alias coder --host 0.0.0.0 --port 8000 \
  --n-gpu-layers 99 --ctx-size 524288 --parallel 2 --no-kv-unified \
  --cache-ram 16384 --flash-attn on \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --jinja --reasoning-format deepseek \
  --chat-template-kwargs '{"reasoning_effort":"medium","preserve_thinking":true}' \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  --no-webui --log-colors off \
  --chat-template-file /templates/chat_template.jinja
```

</details>

<details>
<summary>Preserved Qwen profile, original rationale, client notes and 2026-10-29 comparison</summary>

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

</details>
