# Five-candidate MoE comparison

This is the restarted, calibrated primary run. The initial host-launch probe
in `../moe-20261107-native-context-v1` exposed a PWD/client-directory mismatch;
its native scores are invalid and are not pooled here. Explicit OpenCode
`--dir`, aligned PWD and fixture-root validation restored the Qwen control to
6/6 with unchanged prompts/grader before this run was started.

Baseline: deployed Qwen3.8-27B UD-Q4_K_XL precision (two fixed 262144-token
slots, GPU Q4 K/V, GPU vision, 16 GiB reusable RAM prompt cache).

Requested candidates: Ornith 1.5 Q4_K_M, Qwen3-Coder 30B Q5_K_M,
GLM-4.7-Flash Q5_K_M, Gemma 4 26B-A4B UD-Q5_K_M, and Qwen3.6 35B-A3B
UD-Q4_K_M. Exact artifacts are pinned in `config/candidates.lock.yaml`.

`scripts/download-moe-candidates all` prepares the four new models with the
pinned vLLM image's HF CLI and verifies SHA-256. Ornith already has pinned
weights and a history-fixed template from the previous trial.

`scripts/benchmark-moe-candidates` runs on Brain after downloads finish. It
serializes GPU use with the existing controller lock and restores the
original deployed backend and gateway after each candidate, including failures.
It never changes installed Quadlets, model selection or client configuration.
During native checks the gateway temporarily resolves `llama-coder` to the
candidate's private network alias; a gateway restart clears old connections.

Each model receives three 16384-total-token coding/tool suites at seeds
42–44, a 6144 thinking ceiling where applicable, long-context retrieval,
concurrent streaming/context probes, repair-agent fixtures and two unchanged
native OpenCode read-only batches. The baseline is measured with the same
harness. Native probes run through Brain's real toolbox OpenCode client.
The three source documents used by the native probes remain untouched during
the comparison.

Vision-capable configurations additionally receive color recognition and
Greek-language CI-screenshot extraction checks, each repeated three times.
Text-only models are marked not applicable rather than scored as image failures.

Model-card sampling recipes are recorded in each short-suite result. Qwen3-Coder
is non-thinking and text-only. GLM's native context is shorter than the baseline.
Gemma's template drops old thinking as recommended by its card. Lower-context
or one-slot fallbacks are explicitly recorded, not presented as equal-capacity
drop-in replacements. Weight quantizations are never substituted to make a model fit.
All start attempts retain logs and runtime metadata; unsupported builds and
allocation failures are reported separately from model-answer failures.

The duplicate second batch was cancelled after the user's copy-paste correction.
Results are synthetic local checks and small native-agent samples, not SWE-bench.
