# OpenCode connection for the selected coding model

## Existing Brain provider: required changes

**No connection or model-ID change is required** if your client already uses
`brain/coder-*` through LiteLLM. Ornith Q5 is selected on the server. The
model's Hugging Face repository/quantization is not its client-facing API ID.

| Client setting | Value |
| --- | --- |
| Provider adapter | `@ai-sdk/openai-compatible` |
| Brain-local base URL | `http://127.0.0.1:4000/v1` |
| Home/LAN base URL | Existing Brain URL, typically `http://brain.local:4000/v1` |
| Authentication | Keep the existing LiteLLM key/reference |
| Default model | `brain/coder-medium` |
| Small/background model | `brain/coder-off` |
| Context / output | `262144` / `16384` for every alias |
| Thinking history | `interleaved: {"field":"reasoning_content"}` for thinking aliases |
| Tools / image input | Enabled |

Brain's native OpenCode 1.19.5 configuration already has these values and
was verified as-is. Its generic "Brain / Coder" display names remain accurate.
If a remote client's labels still say Qwen, updating `name` to Ornith is cosmetic.

## Model-entry fragment for an existing provider

Merge the model settings below into your existing
`~/.config/opencode/opencode.jsonc` (or `opencode.json`). Preserve your existing
provider options/API-key reference, other providers, agents and plugin settings.
This fragment shows the default thinking alias and nonthinking background alias;
it is not a replacement for a complete personal configuration.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "brain/coder-medium",
  "small_model": "brain/coder-off",
  "provider": {
    "brain": {
      "models": {
        "coder-medium": {
          "name": "Ornith 1.5 35B-A3B Q5_K_M / Medium (Brain)",
          "reasoning": true,
          "interleaved": { "field": "reasoning_content" },
          "tool_call": true,
          "attachment": true,
          "modalities": { "input": ["text", "image"], "output": ["text"] },
          "limit": { "context": 262144, "output": 16384 }
        },
        "coder-off": {
          "name": "Ornith 1.5 35B-A3B Q5_K_M / Off (Brain)",
          "reasoning": false,
          "tool_call": true,
          "attachment": true,
          "modalities": { "input": ["text", "image"], "output": ["text"] },
          "limit": { "context": 262144, "output": 16384 }
        }
      }
    }
  }
}
```

For existing `coder-low`, `coder-xhigh` and legacy `coder` entries, retain the
same limits, reasoning/tool/image capabilities and `interleaved` field as
`coder-medium`. Retain Brain 1.19.5's already-working `reasoning_options: []`
setting when present; it suppresses automatic client variants in that runtime.
This runtime-specific field is not in every published editor schema/version.

To reproduce the selected sampler, allow Brain's temperature **1.0** default
to apply. An explicit client/agent temperature overrides that default. The
0.6 pilot did not retain all coding-case passes and was not selected; details
and the native-probe limitations are in the [deployment report](deployments/20261108-ornith-q5/README.md).

## Choosing a thinking budget

Select real model IDs rather than automatic effort variants:

| Model | Hard thinking ceiling |
| --- | ---: |
| `brain/coder-off` | 0 |
| `brain/coder-low` | 2048 |
| `brain/coder-medium` | 6144 |
| `brain/coder-xhigh` | 12288 |
| `brain/coder` | 6144; legacy medium |

Ornith's template supports thinking on/off and preserved reasoning history,
but not Qwen's soft effort-steering prompts. The aliases' token ceilings are
the actual controls. The 16K output allowance includes both thinking and the
final answer; everything also fits within the per-request 256K total context.
The server's 524288-token setting is aggregate across two slots.

## Reload and check

If you edit client configuration, **quit and restart OpenCode**; existing
processes keep their loaded settings. Select `brain/coder-medium` in the model
picker if an existing session has another saved model choice. Server-only
model selection needs no client configuration reload when these settings match.

```bash
opencode run --model brain/coder-off 'Reply exactly READY. Do not use tools.'
opencode run --dir /path/to/your/repository --model brain/coder-medium \
  'Use the read tool to inspect README.md and report its first top-level Markdown H1 title, including the # prefix. Do not edit files or use bash.'
```

From this repository on Brain, `make client-check` validates the resolved
native client without printing credentials; `make client-concurrency` checks
three read-only sessions (the third can queue behind the two server slots).

The complete active model/engine pins and llama.cpp command are at
[model-profile.md](model-profile.md). Historical Qwen details remain there.
