# Ornith Q5 deployment verification — 2026-11-08

Selected profile: `ornith-q5`, `Ornith-1.5-35B-Q5_K_M.gguf`, llama.cpp
b12182 (`fd27b2034905f8dadd8bd6d2cc44595f4929d391`), two fixed 262144-token
slots, GPU Q4 K/V, GPU BF16 projector, 16 GiB RAM prompt cache and the pinned
`ornith1.5-history-v1` template.

The persistent switch verified model/projector/template hashes, generated
Quadlets in a staging directory, saved the previous Qwen selection, changed
both model and runtime image, reloaded systemd and tested real gateway inference.
The switch also checks the running image ID against the selected manifest.

Verification outputs in this directory cover the deployed gateway aliases,
streaming/tool history, hard reasoning ceilings, image input, concurrent
requests and actual OpenCode sessions. An output's final `complete`/`passed`
entry or process result is authoritative; existence alone is not a pass.

Rollback to the retained baseline: `make profile-qwen38-q4`. It selects both
Qwen Q4 weights and llama.cpp b11960, rather than retaining the newer engine.
`make profile-status` compares the current selection, arguments and image ID
with the running container. Current settings and dated selection history
are maintained in [model-profile.md](../../model-profile.md).

## Results and limitations

| Check | Result / evidence |
| --- | --- |
| Model/projector/template hashes and Quadlet generation | Passed during staged selection |
| Live model, arguments and image ID | Matched `ornith-q5` / b12182 with `make profile-status` |
| Real inference through LiteLLM | Passed during the persistent switch |
| All reasoning aliases and streaming tool round trips | Passed; `aliases.jsonl` ends with `complete: passed` |
| Low/medium hard thinking ceilings | Passed; `budgets.jsonl` |
| Duplicate thinking history and tool-error recovery | Passed; `aliases.jsonl`, `history.jsonl` |
| Local GPU image request through gateway | Passed; `history.jsonl` |
| Three short concurrent-stream repeats through gateway | Passed with two active slots and overlapping output; `concurrency.json` / `.log` |
| Native client configuration | OpenCode 1.19.5, all five aliases, 262144/16384 limits, tools/images/history metadata and empty variants passed `make client-check` |

The first post-deployment native probe, after the profile page was reorganized,
passed **2/3**: BETA read the page but returned the first section heading instead
of its H1 document title (`native1.json`). The other two jobs passed. The second
original-prompt batch was not run after that failure.

An explicit-H1 probe was added as a separate diagnostic: the expected title
and `read`-tool requirement stay the same, while the prompt specifies H1 and
its `# ` prefix. It passed **5/6** (`native-h1-{1,2}.json`); the remaining GAMMA
job attempted `/home/user/ops/docs/*`, outside the real project, and its
external-directory request was rejected. This is an observed native-agent
limitation, not a successful check, and remains in the raw outputs.

### Temperature 0.6 pilot — not selected

The upstream-recommended general-use temperature was tested via a temporary
build-agent override; no client configuration file was edited. It passed
**6/6** explicit-H1 jobs (`native-t06-{1,2}.json`). Its original-wording probe
still returned the section heading for BETA (`native-t06-original.json`, 2/3).
These small samples do not establish that incorrect-path attempts are eliminated.

The same 0.6 pilot then ran the coding/tool suite at a 16K total output cap and
6144 thinking budget. Seed 42 passed; seed 43 failed `retry_delay` by calling
`math.ldexp` without handling `OverflowError` for a huge attempt value. The
pilot stopped before seed 44; see `t06-seed42.json` and `t06-seed43.json`.
**Temperature 1.0 remains deployed**, matching the winning benchmark configuration.

The user-requested Q5 deployment remains active. Its prior 21/21 coding and
6/6 native benchmark results refer to the frozen campaign inputs; they are
not relabelled or combined with these changed-document deployment diagnostics.
Rollback remains available with `make profile-qwen38-q4`.
