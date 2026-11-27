# Inkling 2 Mini isolated trial — 2026-11-27

**Complete. Inkling 2 Mini is viable after tuning, but is not a clear default
upgrade over Ornith Q5.** The tuned configuration matched the local coding/tool
contracts and adds a validated single-agent 512K option, while decoding about
44% more slowly than Ornith. Keep Ornith as the default; Mini is an interesting
alternative when one larger context matters more than generation speed.

All four controllers restored production and passed gateway inference smoke
tests, including the negative 1M capacity attempt. Final profile verification:
`ornith-q5`, two 262144-token slots, running configuration matches selection;
LiteLLM active, on-demand CPU standby stopped.

## Final tuned scorecard

| Measurement | Ornith Q5 / b12182 | Mini Q4_K_XL / b12649, tuned Q4 KV |
| --- | ---: | ---: |
| Generated-code contracts, seven cases × three seeds | 21/21 | 21/21 |
| Nested tool round trips | 9/9 | 9/9 |
| Multi-turn repairs, including Greek and transient write errors | 6/6 | 6/6 |
| Short-context streaming decode, median of nine runs | 223.86 tok/s | 126.32 tok/s |
| Total wall time for 21 coding cases | 235.55 s | 321.48 s |

The Mini recheck is in `quality-q4-tuned/`: one 524288-token slot, Q4 K/V,
microbatch 128, logical batch 2048, GPU projector, 16 GiB RAM prompt cache,
speculation disabled. Baseline quality results are from the initial fresh
control. Mini's initial Q8-cache results below remain preserved; changing cache,
batch and context can change generations even with fixed seeds, so the recheck
does not prove Q4 is inherently more accurate than Q8.

The tuned capacity/vision run also passed 6/6 image fixtures, 32K effort/budget
controls, short two-slot concurrency/tool history, two simultaneous near-256K
retrievals and a near-512K retrieval/cache-reuse/recovery test. These are small
local functional tests, not evidence of broad SWE-bench parity or proof of
coding quality over an arbitrary half-million-token conversation.

### Tested working server settings

The runtime is the pinned Unsloth llama.cpp prebuilt, mounted at `/runtime` in
the pinned CUDA-capable container described below. Model paths are container
paths. The controller adds the private network, GPU device and volume mounts.

```bash
/runtime/llama-server \
  --model /models/Inkling-2-Mini-UD-Q4_K_XL.gguf \
  --mmproj /models/mmproj-F16.gguf \
  --alias coder --host 0.0.0.0 --port 8000 \
  --n-gpu-layers 99 --fit off \
  --ctx-size 524288 --parallel 1 --no-kv-unified \
  --batch-size 2048 --ubatch-size 128 \
  --cache-ram 16384 --flash-attn on \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --jinja --reasoning-format deepseek \
  --chat-template-kwargs '{"reasoning_effort":"medium"}' \
  --temp 1.0 --top-p 1.0 --top-k 0 --min-p 0.0 \
  --repeat-penalty 1.0 --no-webui --log-colors off
```

For two agents, the tested aggregate context stays 524288 and `--parallel`
becomes 2: **262144 per slot**. Low-effort diagnostic requests override the
template default and do not impose a zero-token reasoning ceiling.
The experiment directly exercised the private backend; persistent gateway
alias/native OpenCode integration would be a separate deployment validation.

## Measured capacity

All rows use Q4_K_XL weights, GPU F16 projector, full GPU layer/KV offload and
no automatic fit. This table records **tested configurations**, not a proof
that every intermediate context size is impossible.

| Per-slot context | Slots | KV | Logical / physical batch | Startup | Whole-GPU memory after load |
| --- | ---: | --- | --- | --- | ---: |
| 524288 | 1 | Q4 | 2048 / 512 | Passed | 27402 MiB |
| 1048576 | 1 | Q4 | 2048 / 512 | OOM, compute-buffer reserve | — |
| 262144 | 2 | Q4 | 2048 / 512 | OOM, compute-buffer reserve | — |
| 262144 | 1 | Q8 | 2048 / 512 | Passed | 24724 MiB |
| 524288 | 1 | Q4 | 2048 / 128 | Passed | 26978 MiB |
| 262144 | 2 | Q4 | 2048 / 128 | Passed | 27860 MiB |
| 1048576 | 1 | Q4 | 2048 / 128 | OOM, compute-buffer reserve | — |
| 1048576 | 1 | Q4 | 512 / 128 | OOM, compute-buffer reserve | — |

`capacity-tuning/` verifies actual inference with low reasoning effort:

- **515,195 prompt tokens**, one 524,288-token slot: all start/middle/end values
  retrieved, 360.74 s wall time (353.50 s server prefill), cached follow-up
  **1.83 s**, 515,191 cached tokens, then recovery passed.
- **Two 253,050-token prompts**, two 262,144-token slots: both retrieved all
  values without cross-request leakage; maximum active slots = 2. Pair wall
  time **265.45 s**, follow-up **2.37 s**, then recovery passed. Prompt ingestion
  and decoding contend: one request's decode rate included a long prefill
  interruption, so it is not a steady-state decode speed measurement.

One million tokens failed startup even after reducing both logical batch and
microbatch. The reduced-batch 1M attempt is retained in
`capacity-1m-batch512/`; its error status is an expected capacity rejection,
and production recovery was still verified.

## Initial coding comparison

| Measurement | Ornith Q5 / b12182 control | Mini Q4_K_XL / b12649, Q8 KV |
| --- | ---: | ---: |
| Seven generated-code contracts × three seeds | 21/21 | 19/21 |
| Nested tool round trips | 9/9 | 9/9 |
| Multi-turn repairs, including Greek and transient write errors | 6/6 | 6/6 |
| Short-context streaming decode, median of nine runs | 223.86 tok/s | 125.76 tok/s |
| Total wall time for 21 coding cases | 235.55 s | 376.20 s |

Mini failed `retry_delay` at seeds 42 and 44 (numeric/overflow edge cases).
Full generated sources and test failures are preserved in `inkling-seed*.json`.
These measure model/runtime/sampling configurations, not model-only causality.
The control kept its production two-slot context; Mini's initial quality run
used one 256K slot with Q8 KV. Each suite submitted one active request at a time.

DSpark depth two did not improve the short streamed checklist in this trial:
125.29 tok/s median versus 125.76 without it. Its separate seed-42 coding/tool
results and runtime command are preserved; this is not a claim that speculation
never helps other prompts or decoding settings.

## Compatibility findings

The initial Q8/default-microbatch run failed all three synthetic long-context
retrieval probes and all six vision probes, emitting repeated special tokens
or no visible answer under forced nonthinking/zero-budget settings. A short
concurrency marker contract also failed due to an unnecessary refusal; tool
history passed. These failures must not be interpreted as successful usable
context merely because the runtime accepted the request.

The tuned Q4/microbatch-128 run passed all six vision cases with **low reasoning
effort and no zero-token hard ceiling**, plus the full long-context checks above.
Its 32K control archive passed `none` with budget zero, `none` without a hard
budget, and `low` without a hard budget. `none` without a hard budget still
produced reasoning. Thus the failures are configuration/prompt dependent;
this experiment does **not** isolate a single root cause or establish reliable
zero-reasoning compatibility for existing `coder-off` clients.

## Reproduction

Run on Brain (prefix with `flatpak-spawn --host` from toolbox):

```bash
python3 scripts/prepare-inkling-mini-trial
python3 scripts/benchmark-inkling-mini

# Separate buffer/effort diagnostics, after the first run finishes:
python3 scripts/benchmark-inkling-mini --capacity-only --ubatch 128 \
  --thinking-probes --output docs/benchmarks/inkling-mini-20261127/capacity-tuning

# Targeted 1M startup attempt (failed/OOM; always restores production):
python3 scripts/benchmark-inkling-mini --capacity-only --ubatch 128 --batch 512 \
  --single-context 1048576 --thinking-probes \
  --output docs/benchmarks/inkling-mini-20261127/capacity-1m-batch512

# Quality recheck on the working Q4-cache / 512K configuration:
python3 scripts/benchmark-inkling-mini --quality-only --ubatch 128 \
  --single-context 524288 \
  --output docs/benchmarks/inkling-mini-20261127/quality-q4-tuned
```

The downloader checks SHA-256 hashes from `config/inkling-mini-trial.lock.json`.
It downloads the 17.63 GB Q4_K_XL weights, F16 projector, Q8 DSpark drafter and
Unsloth llama.cpp `b12649-mix-c7059d6` CUDA 13 portable archive. Runtime source
commit: `189f431bf58f433d376d718c7788b1bf4467fa72`.
The prebuilt is mounted in the existing digest-pinned vLLM image **as a runtime
environment for llama.cpp**, not as a vLLM inference server. The base image
provides CUDA libraries absent from the host. Its digest is recorded in the
controller and status file. No packages are installed on the host.

The controller uses the shared GPU-controller lock, snapshots the benchmark
scripts, runs a fresh Ornith control, starts CPU standby before stopping the
GPU service, then trials one GPU backend at a time. `--fit off` prevents silent
context reduction or automatic layer offload. It checkpoints `status.json`,
records startup commands/logs, and restores/smoke-tests Ornith in `finally`.
The output directory refuses to overwrite an earlier status file; choose a new
`--output` for reruns. `--capacity-only` omits the fresh baseline/coding suite;
`--thinking-probes` adds effort/budget controls and low-effort vision checks.
`--single-context` targets one Q4-cache allocation; `--batch` and `--ubatch`
override logical and physical batch sizes independently. `--quality-only`
rechecks coding/repairs without rerunning context diagnostics;
`--quality-after-capacity` can append that recheck to a successful capacity run.
The initial quality run uses a 4 GiB RAM prompt cache; capacity/recheck modes use
16 GiB so large reusable prompt states fit. These are RAM caches, not KV spill.

## Scope

- Startup capacity: 512K/one slot, 1M/one slot, 256K/two slots, 256K/one slot Q8 KV;
  smaller fallbacks if none start.
- Three coding/tool seeds per model, six multi-turn repair fixtures, API
  streaming, Greek/English contracts, vision, short concurrency/recovery.
- Retrieval/cache-reuse near the largest successfully allocated context and,
  in capacity-only tuning, simultaneous near-full requests for two slots.
- Separate DSpark depth-two run; baseline quality runs have speculation disabled.

Mini sampling: temperature 1, top-p 1, top-k 0, min-p 0, repeat penalty 1.
Coding tests request medium effort and a 6,144-token reasoning budget within
16,384 total output tokens. Initial nonthinking text probes explicitly request
`none`; the legacy vision suite disables thinking with a zero-token budget.
The diagnostic vision option `--reasoning-effort low` removes that hard ceiling.
These are small local contract tests, not a reproduction of published SWE-bench
scores or proof of useful coding quality across a million-token history.
The trial directly tests the private backend API. Gateway alias compatibility
and native OpenCode deployment remain separate promotion checks.

## Context estimate correction

The earlier chat estimate counted all 24 physical layers as global attention
and treated sliding-window attention as vision-only. The upstream text config
actually has 20 local layers (1,024-token windows), four global layers, and
loops physical layers 6–17 three times. Loops also affect runtime cache layout;
the measured startup logs and long-prompt requests are authoritative. A 1M
startup allocation is therefore tested rather than ruled out from the earlier
formula. Quant block metadata and compute buffers also invalidate a simple
4-bit-equals-half-a-byte estimate as an exact memory budget.
