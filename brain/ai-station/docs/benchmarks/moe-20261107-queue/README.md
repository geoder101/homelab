# Extended comparison queue

The active control remains Qwen3.8-27B UD-Q4_K_XL precision. Controllers
temporarily serve candidates during tests and restore the selected baseline
after each trial. Promotion requires reviewing results; scheduling a variant
does not select it for persistent deployment.

## Execution order

1. Finish the original four new GGUF downloads, then run batch 1 with the
   five originally requested candidates, including a fresh Ornith trial.
2. Download and run **Flint → Nemotron 3.5 Lightning → Devstral Small 2 →
   Occamy 1.0 → Muse Glimmer 30B → gpt-oss-20b**, preserving the user's priority.
3. Run targeted tuning variants: Qwen Q3 at the same two 256K slots; matched
   Qwen Q4-weight/Q4-KV versus Q8-KV two-slot 128K configurations; the same
   Qwen Q4 precision profile on a newer pinned llama.cpp build; Qwen NVFP4 on
   pinned vLLM; Ornith Q5; Qwen3-Coder Q4; and native gpt-oss MXFP4 on vLLM.
   The Qwen3-Coder Q4 variant follows an observed Q5 two-slot KV allocation
   failure; the Q5 weights only fitted a single full 256K slot in calibration.

Batch 2 starts with Flint UD-Q4_K_XL (~25.1 GB), Nemotron MXFP4_MOE (~23.2 GB),
Devstral Q6_K (~19.3 GB), Occamy Q4_K_M (~21.2 GB), Muse UD-Q5_K_XL (~21.8 GB),
and gpt-oss Q6_K (~12.0 GB). Projectors are additional where applicable.
The gpt-oss GGUF keeps native MXFP4 expert tensors; its filename does not
mean every model tensor is quantized to six bits. Native vLLM weights are
a separate format/configuration comparison, not a tensor-identical engine test.

The original llama.cpp b11960 deployment is retained. A separate trial image,
**b12182**, is pinned by its linux/amd64 digest and commit in the image lock.
The Qwen new-engine variant isolates that runtime change. All vLLM trials use
the already-pinned 0.32.0 image. No moving image tags are executed.

The vLLM clients omit llama.cpp's `reasoning_budget_tokens` extension;
those trials retain the common 16K total-output limit and report that distinction.
GPT-OSS's mandatory-analysis path uses low effort and larger short/context-answer
budgets so an analysis-only truncated response is not mistaken for a context
failure. Its streaming throughput includes analysis tokens from the first
generated token; visible-answer TTFT is recorded separately. Output budgets
are stored with the measurements rather than treated as identical to nonthinking models.

Each batch freezes its harness sources and hashes, then records fresh controls, three coding/tool seeds, long context,
concurrency, fixture repairs and two real OpenCode batches. Model-card sampling
recipes are stored in results. One-slot or lower-context fallbacks are recorded
as capacity tradeoffs, rather than equivalent replacements. Backend-specific
unsupported controls and startup/allocation failures remain distinguishable
from answer-quality failures. Small synthetic tests do not establish broad
coding quality; native tool behavior remains a separate gate.

All calibrated batches run two deterministic image checks, each repeated
three times, for vision-capable models and their fresh controls:
solid-color recognition and Greek-language extraction of test counts/failed
filenames from a synthetic CI screenshot. These check image plumbing and
basic coding-UI interpretation, not broad multimodal quality. A dash in the
vision column means a model is text-only or a check was not enrolled.

The initial host-launched calibration is preserved separately in
`../moe-20261107-native-context-v1`. Its PWD differed from the child cwd and
native path scores failed even for Qwen, so those scores were excluded.
The launcher now supplies explicit OpenCode `--dir` and aligned PWD and checks
fixture visibility. Qwen passed both corrected three-session controls before
the primary comparison restarted; the original prompts/grader were unchanged.

## Live progress

- `SUMMARY.md` / `SUMMARY.json`: automatically updated scorecards, including
  coding-task wall time alongside generation rate; partial runs are labelled.
- `../moe-20261107/status.json`: first batch's measurements and restoration.
- `status.json`: queued second batch and tuning stage progress.
- `../moe-20261107-batch2/status.json`: second batch's detailed runtime/results.
- `../moe-20261107-tuning/status.json`: tuning configurations and measurements.

Host journals:

```bash
journalctl --user -u ai-station-moe-download -u ai-station-moe-batch1 -u ai-station-benchmark-queue -f
```

The duplicate second list was removed; the six-model corrected list is the
scheduled second batch. `config/benchmark-queue.yaml` contains immutable download
inputs and variant settings; `config/candidates.lock.yaml` retains audit records.
Root README/model-profile/operations documents used by native tests are frozen
throughout the run and their hashes are checked after restoration.
