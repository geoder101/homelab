# Brain reasoning profiles and Home handoff — 2026-10-31

**Current selection (2026-11-08):** Ornith Q5 on b12182 serves the same aliases
and hard thinking ceilings, with two 256K slots, Q4 cache and GPU vision.
Its pinned history-fixed template does not implement Qwen's soft effort
steering. See [current configuration and selection history](model-profile.md).
This report preserves the Qwen reasoning/template handoff. Its three-slot
Q3/CPU-vision alternative is recorded in
[local capacity tuning](concurrency.md). Measurements and one-slot/Q4/Q8
descriptions below record the earlier reasoning-profile handoff.

This report describes the initial Home handoff. A subsequently authorized
follow-up adds Makefile automation and updates **Brain's native OpenCode**
config, without OmO; see [automation](automation.md). The Home Downloads copies
and Home's live configuration remain separate.

## Deployed result

LiteLLM exposes four real profile IDs plus backward-compatible `coder`. All
route to `openai/coder` at `http://llama-coder:8000/v1`, using **one** llama.cpp
process, one loaded Qwen checkpoint, and one request slot. The host-facing test
URL was `http://brain.local:4000/v1`. The backend has no published host ports.

| Alias | Thinking | Soft effort | Hard thinking ceiling | Measured reasoning, probability task | Measured reasoning, coding-plan task |
| --- | --- | --- | ---: | ---: | ---: |
| `coder-off` | disabled | `none` | 0 | 0 | not run |
| `coder-low` | enabled | low | 2,048 | 586 | 151 |
| `coder-medium` | enabled | medium | 6,144 | 690 | 205 |
| `coder-xhigh` | enabled | xhigh | 12,288 | 470 | 10,072 |
| `coder` | enabled | medium | 6,144 | 690 | not run |

Counts in this table retokenize the extracted `reasoning_content` with the
running GGUF tokenizer, excluding delimiters. They are measurements of one
seed/prompt, **not guaranteed effort ratios or quality scores**. Effort changes
the prompt; token consumption need not increase monotonically.

Thinking sampling remains temperature 1.0, top-p 0.95, top-k 20, min-p 0,
presence penalty 0 and repetition penalty 1. Non-thinking defaults are
temperature 0.7, top-p 0.8, presence penalty 1.5, with the same top-k/min-p.
These are defaults, not enforced request policies: authenticated clients can
override parameters. Do not select client variants that override the alias.

The 16,384 client output limit includes thinking and final output. The xhigh
ceiling leaves roughly 4K for a final answer at that output limit. A smaller
request `max_tokens` can still truncate before any final answer. There is no
global `--reasoning-budget` in the Quadlet.

## Runtime and immutable artifacts

- Fedora host `brain`, rootless user services as `geoder101`; operations were
  executed from toolbox with `flatpak-spawn --host`.
- llama.cpp b11960, commit `8642541b255bfb5dc39cd351a178be09013a55d6`, image
  `ghcr.io/ggml-org/llama.cpp@sha256:ae676f2ecfdeb66db48b7c32f1f013c62cdbc3e380b1f6e8b425c981dd34ac99`.
- Model `unsloth/Qwen3.8-27B-GGUF`, revision
  `4ca720788d1e01f1bff70c033e0d0028fd02e502`, Dynamic V3
  `Qwen3.8-27B-UD-Q4_K_XL.gguf`, with `mmproj-BF16.gguf`, under
  `~/.local/share/ai-station/models/qwen3.8-27b-gguf/`.
  Both local Hugging Face download metadata files identify that revision.
  Full-file SHA256 checks matched their recorded artifact hashes:
  model `3f227079003add2511437e5b1e94812e363385225bf6a9b47b0054a72bc8b01e`,
  projector `83ee4f4f205fa514161778c41df1ea14144faa0f713510893b63c2395f5c2d53`.
- 262,144 context, GPU Q8 K/V cache, Flash Attention, `--parallel 1`, no MTP.
- LiteLLM 1.106.5, digest
  `sha256:471f48b56221afba5adde2502b39a2596d04fa4ba30f2a82f002dd170fff4913`.
  Open WebUI 0.12.4 and the optional vLLM image were not upgraded.
- Template [froggeric/Qwen-Fixed-Chat-Templates](https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates),
  `chat_template.jinja`, version `qwen3.8-froggeric-v22.5`, exact revision
  `855bffc49448e299789730ff92c9b8d834d6cc14`.
- SHA256: `e57684bae4156211a55473c5a63be976a405a37ab5be5ae0e5abf1df5349c4b2`.
  Installed read-only (mode 0444) at
  `~/.local/share/ai-station/chat-templates/855bffc49448e299789730ff92c9b8d834d6cc14/chat_template.jinja`;
  mounted read-only as `/templates/chat_template.jinja`. The downloader verifies
  the checksum and refuses an existing different copy. The model GGUF was not
  rewritten, so its original embedded template remains available.

The original deployment used the GGUF's Unsloth-modified embedded template,
with `reasoning_effort=medium` and `preserve_thinking=false`. `--help` from the
installed binary confirmed `--jinja`, `--chat-template-file`,
`--reasoning-format deepseek`, and `--reasoning-budget` support. The new template
was downloaded and verified **before** install/reload. Quadlets were validated,
installed and reloaded, then llama-coder and LiteLLM restarted.

## What was verified about parameter handling

LiteLLM's documented `model_list[].litellm_params` holds standard parameters;
`extra_body` holds llama.cpp-specific fields. `scripts/audit-reasoning-forwarding`
uses the installed LiteLLM Router and OpenAI SDK with an intercepted HTTP
transport. It asserts the local destination, backend model ID, top-level
`reasoning_effort`, nested boolean `chat_template_kwargs.enable_thinking`,
`preserve_thinking`, and per-alias `reasoning_budget_tokens` in serialized
request bodies. This is an installed-library serialization check, not packet
capture of the live gateway. Live inference, rendered prompts and gateway
prompt-token counts independently confirmed that profile settings take effect.

Source inspection of the pinned llama.cpp `tools/server/server-common.cpp`
confirmed:

- Request `chat_template_kwargs` merges over CLI kwargs.
- `chat_template_kwargs.enable_thinking` must be a boolean.
- Top-level `reasoning_effort` overrides nested effort; `none` disables thinking.
- A standalone top-level `enable_thinking:false` is **not** the control used by
  this build. An actual `/apply-template` request with that field still ended
  in an open `<think>`; the nested boolean produced a preclosed think block.
- `reasoning_budget_tokens` (or `thinking_budget_tokens`) is the supported
  request-local budget. The discussion's generic `thinking_budget` name is not
  used here. CLI budget defaults remain unrestricted.

`coder-off` uses nested `enable_thinking:false`, top-level effort `none` and
budget 0. Its prompt ends with a **prefilled closed** think block. Probability
and streaming-tool tests returned zero generated reasoning, with final content
and tool arguments intact. This disables the generated thinking phase rather
than merely removing a response field. It does not forbid explanations in
ordinary final content, or establish anything about internal neural computation.

## Loop investigation and limits of the fix

Reviewed [Qwen discussion 178](https://huggingface.co/Qwen/Qwen3.8-27B/discussions/178)
and the community template's source and claims. The discussion includes BF16
and quantized reports, so it does not establish a quantization-only cause.

1. **Template default:** both templates can steer effort, but the community
   version defaults to medium. The previous deployment already explicitly used
   medium; the upstream xhigh default alone cannot explain this deployment's
   loops. Each alias now explicitly selects effort.
2. **History/empty-think handling:** before the change, rendering in-content
   `<think>HISTORY_SENTINEL</think>Hello` with preservation enabled prepended an
   additional empty think block. After the change the installed minja renderer
   extracted the sentinel into one canonical block. Supplying both an explicit
   reasoning field and in-content thinking also rendered the sentinel once.
   The community template can still emit empty historical blocks when no
   reasoning was supplied; its broad claim to eliminate empty-think poisoning
   is not treated as a universal guarantee.
3. **Supplied reasoning:** preservation is now enabled. It cannot restore
   thoughts a client omitted. Offline Home thinking models declare
   `interleaved: {"field":"reasoning_content"}`; OpenCode 1.19.5 source shows
   this resends assistant reasoning via the compatible SDK. Runtime proof of
   Home's actual serialization remains a Home-side check.
4. **Tool serialization:** standard OpenAI JSON-string arguments were accepted,
   normalized to native XML parameters, streamed as structured JSON tool calls,
   and resent successfully. A two-error history rendered the community
   escalation warning. Live medium inference corrected `Pariss` to `Paris`
   rather than issuing the same failed arguments again. This is one synthetic
   recovery case, not proof that arbitrary agent loops terminate.
5. **Sampling:** existing thinking values already matched the recommended
   baseline. No aggressive repetition/DRY penalty was added; semantic
   self-validation can continue without lexical repetition, and penalties can
   damage code. No controlled sampling ablation was performed.
6. **Quantization:** this work tested the actual selected Dynamic Q4 checkpoint;
   no BF16/alternative-quantization comparison was run, so its contribution is
   unresolved.

The coding-plan probe still exposed xhigh overthinking: **10,072 reasoning
tokens**, 11,298 total completion tokens, about 149 seconds decoding. Much of
the reasoning repeatedly drafted and counted the final answer. It eventually
finished, but the template is not a complete cure. Output code quality is also
not guaranteed: medium's plan included invalid SQLite `FOR UPDATE` syntax;
generated plans were not executed as application tests.

Hard ceilings contain thinking duration independently of soft effort. With the
same long coding-plan task and an explicit top-level xhigh effort override,
the low alias's 2,048 ceiling yielded 2,047 extracted reasoning tokens and a
final answer (46.85 s total); the medium alias's 6,144 ceiling yielded 6,143 and
a final answer (99.59 s). The boundary-token difference is expected from
retokenizing extracted text. The xhigh 12,288 default was serialized and its
10,072-token natural run stayed below it; saturation at exactly 12,288 was
not tested. A separate request-local 32-token override yielded 31 extracted
tokens and a final answer, proving request override and forced exit, but that
answer gave **19/8 instead of 15/8**. Tight budgets can reduce correctness.

Repeated multi-turn/agent loops, exceptionally large histories, and production
coding quality remain unresolved beyond these checks. Thinking ceilings do
not cap the number of agent requests or fix repeated tool orchestration.

## Checks actually performed

- Inspected clean starting Git status, generated service definitions,
  running images/ports, backend `/props`, template and sampling settings.
- Fedora user Quadlet generator dry-run passed; installed units retained
  `DefaultDependencies=false`, NVIDIA CDI, GPU-only SELinux exception and
  `Pull=never`.
- `/v1/models` listed `coder`, `coder-off`, `coder-low`, `coder-medium`,
  `coder-xhigh`. Each completed the same probability request (seed 42).
  The normal profiles answered 15/16 and 15/8 correctly.
- Streaming tool-call assembly, JSON argument parsing and tool-result final
  answers passed on all four new aliases. Both required-tool and auto-choice
  suites were run. Auto-choice returned the expected lookup call on each alias.
- Installed minja rendering checked low/medium/xhigh steering, closed off
  prefill, explicit-plus-in-content reasoning de-duplication and error history.
- A locally generated red PNG through `coder-off` returned `Red`; no external
  media source was involved. This checks basic vision compatibility, not broad
  multimodal quality or video.
- `scripts/smoke-test http://brain.local:4000` returned `brain inference ready`.
- GPU process snapshots showed one `/app/llama-server`: 27,660 MiB before,
  27,696 MiB after template loading and 27,742 MiB after the image request.
  vLLM remained inactive. One llama container and one server slot remained;
  only LiteLLM :4000 and Open WebUI :3000 publish host ports.
- A read-only inspection of Open WebUI's persisted configuration confirmed
  its sole model connection is `http://litellm:4000/v1`, with no per-connection
  model override and no external task model. Direct connections and Ollama
  are disabled. Web search, external image generation/editing and realtime
  audio are disabled; embedding/STT/TTS engines have local/default selection.
  No WebUI database settings were changed.
- Offline Home validation passed as described below; `bash -n` and
  `git diff --check` are part of final repository verification.

Reproduce on the **Brain host**, from this checkout:

```bash
podman exec -i ai-station-litellm python < scripts/audit-reasoning-forwarding
scripts/verify-reasoning-profiles --stress
scripts/verify-reasoning-profiles --budget-check
scripts/verify-reasoning-profiles --history-check
scripts/smoke-test http://brain.local:4000
```

From toolbox, prefix host commands with `flatpak-spawn --host` (stdin redirection
for the audit still reads the checkout file). The verification script reads
the host key internally and never prints it. All inference uses local models;
schema/template downloads are development-time public artifact retrieval.

## Offline Home files and schema validation

Prepared files on Brain, ready for the user to copy manually:

- `~/Downloads/opencode.jsonc`
- `~/Downloads/omo.jsonc`

During the initial handoff, no active Brain OpenCode/OmO file, plugin/package, setting or session was edited,
installed or reloaded, and no OpenCode/OmO command was executed on Brain. Home
was not accessed remotely. No real key was copied into these files or Git.

OpenCode retains provider ID `brain`, `@ai-sdk/openai-compatible`,
`http://brain.local:4000/v1`, Home's existing
`{file:~/.config/ai-station/litellm-api-key}` reference and
`oh-my-openagent@5.1.51`. Four model entries replace client-only variants.
Primary is `brain/coder-medium`; small model is `brain/coder-off`. Context and
output remain 262,144 / 16,384. Thinking entries include the reasoning history
field; image input metadata reflects the loaded, tested projector. Build/plan
use low, general medium, explore/title off and summary/compaction low.

OmO preserves every original agent/category name and the requested target
mapping, including Sisyphus-Junior off and Oracle xhigh. Entries retain their
existing `models` versus `model` shape, with direct real aliases and no effort
or variant override. `[opencode]` permissions/disabled hook and all three
`_migrations` values were retained. No `provider_options` or `#variant` ID is
present. The enabled-provider allowlist and explicit single-model assignments
contain no remote model fallback; Home's full merged runtime must still be
inspected for project overrides.

Both files are strict JSON (a valid JSONC subset). `scripts/validate-home-configs`
reads only those offline files and public schemas; it does not start a client.
It passed with Python `jsonschema` in the development toolbox:

- Full OmO schema at the file's pinned source commit
  `c8955517dd95d2d8dd1ed6c871705a2eb9921840`; that commit's `package.json`
  identifies version **5.1.51**. Schema SHA256
  `bb72795f08e14786fbf3218119c7bd7cadbbeda2df233e76eed96d06e5ee8701`.
- Published OpenCode schema SHA256
  `7feca1416fc02b39201e468160199cbe5957792523f482e35c050f8996dcb0ad`.
  Its external models.dev catalog enumerates public IDs and rejects custom
  `brain/*` strings. The validator explicitly extends **only that catalog enum**
  with the four declared custom IDs, then checks all other constraints. This
  is not a claim that the unmodified editor catalog accepts custom IDs.
- OpenCode **1.19.5** source, tag commit
  `810c0eba1a7348036f8bdf61e6632d0c36c6876c`, was reviewed for ConfigV1,
  provider-model fields, reasoning-history serialization and CLI commands.
  Its runtime model fields accept strings, unlike the editor catalog enum.
  No assumption about Brain's installed client version was needed.

### Final verification on Home (not executed here)

First back up Home's existing active files, then copy the prepared files to
the **same active locations Home already uses**. OmO's active location was not
inferred from Brain. Quit and restart Home's OpenCode after installation.
Run these commands **on Home only**, in the intended project so project
overrides are included:

```bash
opencode --version                    # expected 1.19.5
opencode models brain                 # must show the four real aliases
opencode debug config | python3 -c 'import json,sys; c=json.load(sys.stdin); print(json.dumps({k:c.get(k) for k in ("model","small_model","enabled_providers","plugin")},indent=2))'
for agent in sisyphus hephaestus oracle librarian explore prometheus atlas sisyphus-junior; do
  opencode debug agent "$agent" | python3 -c 'import json,sys; a=json.load(sys.stdin); print(a.get("name"),a.get("model"),a.get("variant"))'
done
for model in coder-off coder-low coder-medium coder-xhigh; do
  opencode run --model "brain/$model" --thinking --format json \
    'Reply with exactly READY. Do not use tools.'
done
opencode run --model brain/coder-medium --thinking --format json \
  'Use your file listing tool to list this project root, then summarize its entries. Do not edit files.'
```

Expect no reasoning event for off; thinking profiles may emit reasoning,
depending on the trivial prompt. Use the probability question in
`scripts/verify-reasoning-profiles` for a stronger reasoning probe. Confirm a
structured tool event, a completed tool-result follow-up and no repeat loop.
If an agent's exact display name differs, `opencode agent list` lists names.
These are the remaining checks of Home's plugin loading, merged configuration,
AI SDK streaming, historical reasoning and agent/category resolution. They
were not simulated by running Brain's active client.

## Files changed

Repository:

- `config/litellm-config.yaml`: aliases, standard effort defaults, nested kwargs,
  non-thinking sampling and per-profile budgets.
- `quadlets/llama-coder.container`: immutable template volume/file, preserve history.
- `config/chat-templates.lock.yaml`: new upstream revision/version/checksum record.
- `config/models.lock.yaml`: verified artifact hashes, selected template and reasoning-history record.
- `scripts/download-chat-template`: new checksum-verified immutable downloader.
- `scripts/download-coder-candidate`: download template after model artifacts.
- `scripts/audit-reasoning-forwarding`: installed-library serialization check.
- `scripts/verify-reasoning-profiles`: live local inference checks.
- `scripts/validate-home-configs`: offline syntax/schema/invariant checks.
- `README.md`, `AGENTS.md`, `docs/operations.md`, `docs/model-profile.md`,
  `docs/benchmarks/README.md`, and this report: deployment and validation guidance.

Outside Git: the two Downloads files, the immutable state-directory template,
and the installed llama Quadlet under `~/.config/containers/systemd/` were
updated. The existing install script recopied the other unchanged Quadlets;
only llama-coder and LiteLLM were restarted. Image/model pins and secrets
were not changed.

## Rollback

The original embedded template is still in the unchanged GGUF. For a
template-only rollback, set `models.coder.chat_template` to `embedded` in
`config/models.lock.yaml`, remove the `/templates` volume and
`--chat-template-file /templates/chat_template.jinja` from the repository
llama Quadlet, reinstall/reload, restart llama-coder, wait for readiness and
restart LiteLLM. Aliases/budgets remain usable with the embedded template, but
the old history-rendering limitations return. To restore old history behavior
too, change both CLI and alias `preserve_thinking` to false.

For the **exact pre-handoff deployment**, original tracked configuration is
preserved in Git commit `b602d8cd1bb09bfc2ecd1206a113e633c88261c8`. Review the
diff before restoring these deployment files (this deliberately removes the
new aliases):

```bash
git diff b602d8cd1bb09bfc2ecd1206a113e633c88261c8 -- \
  quadlets/llama-coder.container config/litellm-config.yaml config/models.lock.yaml scripts/install-quadlets
git restore --source=b602d8cd1bb09bfc2ecd1206a113e633c88261c8 -- \
  quadlets/llama-coder.container config/litellm-config.yaml config/models.lock.yaml scripts/install-quadlets
scripts/validate-quadlets
scripts/install-quadlets
systemctl --user daemon-reload
systemctl --user restart llama-coder
# Wait for startup completion in journalctl --user -u llama-coder -f, then:
systemctl --user restart litellm
scripts/smoke-test http://brain.local:4000
```

Restoring the old installer as well is necessary because the old model lock
predates the hashes and template-selection fields required by the new preflight.
The downloaded template can remain as an unused rollback artifact. Restore
Home's backed-up client files manually if its new aliases are removed, then
quit and restart OpenCode on Home. If the new aliases are removed, also restore
Brain's original native config backup so its primary and small-model settings
do not refer to missing aliases; see [automation](automation.md).
