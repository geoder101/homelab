# Fully local capacity tuning — 2026-10-31

**Current selection (2026-11-08):** Ornith Q5 on b12182, two fixed 256K slots
and GPU vision; see [current settings and preserved history](model-profile.md).
The earlier [Ornith Q4 trial](benchmarks/ornith-20261105.md) ended with Qwen
restored at that time. This document records the three-slot capacity study, whose
profile remains available with `make profile-qwen38-q3`.

## 2026-10-31 selection: three fixed 256K agents

This is the **capacity** alternative. The persistent global **precision**
profile remains available for rollback; `make profile-status` shows the selected and running
configuration. See [switching profiles](#global-profile-switching).

The capacity-study configuration uses Qwen3.8-27B Dynamic V3 **UD-Q3_K_XL weights**, GPU **Q4 K/V
cache**, **three fixed 262,144-token slots**, a **16 GiB RAM prompt-cache
limit**, and the vision projector on CPU. Model inference remains on the
RTX 5090. The checkpoint revision, reasoning aliases, pinned froggeric template
and client context/output limits are preserved.

This is the **best tested capacity profile**, not a claim of a universal
optimum or quality parity between quantizations. Lower-precision weights/cache
can affect model quality. All six coding contracts, three nested tool round
trips, long-context retrieval and deployed client checks described below passed.
The original higher-precision weights are retained for comparison/rollback.

The server uses `--ctx-size 786432 --parallel 3 --no-kv-unified`: 768K is the
aggregate allocation, **not** a 768K context for an individual conversation.
Every client continues to declare **262144 context / 16384 output**. Input,
reasoning, image tokens and output all share that per-agent budget. Extra
requests queue behind the three inference slots. The aliases share those
slots; selecting another reasoning alias does not create additional capacity.

No cloud inference, external machine, model fallback or new runtime package
was introduced. New model artifacts were downloaded during preparation from
immutable revisions and SHA256-checked. Steady-state inference is fully local.

## Machine and controls

- RTX 5090: 32,607 MiB reported total; about 4.2 GiB initially free with the
  original model loaded. Reported free excludes driver reservations.
- Ryzen 9 9950X: 16 cores / 32 threads; 62 GiB RAM, approximately 48 GiB
  initially available.
- llama.cpp b11960, `8642541b2`; pinned image unchanged.
- Alternative server: existing pinned vLLM 0.32.0 image, not a package upgrade.
- Main-model revisions/checksums: `config/{models,candidates}.lock.yaml`.
- Baseline: original Q4_K_XL weights, Q8 cache, one 256K slot, GPU projector.
- Candidate reasoning/template/sampling matched the current production
  settings, including thinking preservation. Historical 2026-10-29 embedded-
  template figures are not substituted for the new control measurement.
- GPU candidates ran sequentially with the production GPU backend stopped.
  CPU-helper experiments retained the main model. The harness restores the
  deployed service on normal completion, exceptions, SIGINT and SIGTERM.

Raw synthetic results and server logs are under
[`benchmarks/concurrency-20261031/`](benchmarks/concurrency-20261031/).
They include exact arguments, image digest, API token usage, timing, slot
observations, and one-second GPU/RAM/swap samples. Short screening runs are
single observations unless repeated explicitly; samples can miss brief peaks.

## Principal results

Short requests ask for a bounded diagnostic checklist with thinking disabled;
generation is capped at 384 tokens. The following are **short-context** numbers,
not throughput promises for full 256K histories.

| Setup | Observed short decode / aggregate | Capacity and decision |
| --- | --- | --- |
| Original Q4 weights / Q8 cache, one slot | ~72 tok/s single; ~71 aggregate for a queued pair | Control |
| Q4 weights / unified Q8 256K pool, two slots | ~111 aggregate | Fast, but shared capacity is only 256K total |
| Q4 weights / unified Q8 320K pool | ~111 aggregate | Mixed large/small requests passed; only ~1.6 GiB free |
| Q4 weights / Q4 cache, two fixed 256K slots | ~69 single; ~108 aggregate | Two ~253K requests passed; ~2.6 GiB free; higher-weight-precision alternative |
| Q4 weights / Q5 cache, two fixed slots | ~73 single; ~113 aggregate | Full two-agent retrieval passed, but only ~0.6 GiB free |
| Q4 weights / Q5 keys, Q4 values, two slots | ~73 single; ~112 aggregate | ~1.6 GiB free; not selected over safer headroom |
| Q3 weights / Q8 keys, Q4 values, two fixed slots | ~89 single; ~145 aggregate | Coding/tool checks passed; ~2.6 GiB free |
| Q3 weights / Q4 cache, three fixed slots, GPU vision | ~82 single; ~183 aggregate for three | Starts and streams correctly; only ~0.9 GiB free |
| **Q3 weights / Q4 cache, three fixed slots, CPU vision** | **~82–83 single; ~181 aggregate through LiteLLM for three** | **Selected; three ~253K histories passed; approximately 2 GiB free** |
| Q4 weights / Q4 unified 512K pool, four slots | ~167 aggregate for four | Four short requests work, but four full 256K histories cannot fit in that pool |
| Q4 weights / Q8 KV on CPU | ~25 single; ~32 aggregate | Functional, but substantially slower even at short context |
| Q4 weights / 48 GPU layers, remaining layers on CPU | ~13 single; ~23 aggregate | Functional but too slow for the selected goal |
| vLLM NVFP4 KV, CUDA graphs, two sequences | ~66 single; ~95 aggregate | 256K maximum works; only 306,267 aggregate cache tokens reported |
| vLLM FP8 KV + 4 GiB weight offload | Functional, much slower | Capacity gained at excessive short-request latency |
| CPU-only Qwen3.5-4B Q4 helper, 8 or 16 threads | ~20 tok/s | Main model stayed near baseline speed; helper failed the topological-sort coding contract |

Further startup screens included all-GPU Q8 pools at 384K/512K, mixed Q8/Q4
512K caches, smaller microbatches, and partial offload with 56 GPU layers.
Those configurations failed allocation. Offloading the projector let a 384K
Q8 pool start but left only about 0.3 GiB free. These failures are preserved
in the logs; allocation-failure memory samples are not total-memory estimates.

The 4B helper also ran alone on GPU and passed the basic streaming/tool screen.
GPU co-residency with the large model was not tested. Its CPU coding failure
and the capacity already obtained with three 27B slots did not justify
promoting it into automatic agent routing. The optional helper downloader and
artifact record remain available for future lightweight-task experiments.

### Why fixed slots won over a shared pool

Unified cache improves utilization when conversations are differently sized.
However, a four-slot 512K pool advertises a 256K maximum for each conversation
without reserving four times that capacity. The pinned llama.cpp server can
return context errors when the aggregate pool fills; safe unrestricted use
would require admission control or a sufficient worst-case allocation.

Three fixed slots give a simpler guarantee: **three independent 256K budgets**,
ordinary queueing beyond three, and no token-aware gateway scheduler. The
deployed queue test submitted four requests, observed at most three active
slots, and confirmed the fourth waited and completed successfully.

### Long histories and cold-prompt latency

The selected profile processed **three independent requests with 252,933 prompt
tokens each**, each recovering distinct START/MIDDLE/END values without
cross-contamination. All three slots were observed active. The total workload
took about **652 seconds** including prompt construction and a follow-up.
The resumed first conversation reused 253,005 cached tokens and completed in
about **3.1 seconds**. The full run's sampled minimum available host RAM was
about **39 GiB**, with no swap consumption.

This test accepts near-full independent histories; it does not prove coding
quality at that length. Its answers were capped at 128 tokens, not 16K. Cold
prefill is still expensive: one large prompt's processing can substantially
stall another request's decoding. Full-context decode is much slower than the
short-context table, and concurrency does not triple GPU compute capacity.

The two-slot Q4-weight alternative similarly passed two independent ~253K
histories in about 432 seconds. Mixed 160K/16K and 250K/6K workloads passed on
that profile and the unified Q8 320K profile.

vLLM with **NVFP4 cache** is a real alternative, not the old FP8 configuration:
the graph-enabled candidate reported 5.67 GiB cache / 306,267 tokens / 1.17x
full-context concurrency. Mixed workloads took about **59 seconds** for
160K/16K and **119 seconds** for 250K/6K, compared with roughly 107/214 seconds
on the two-slot llama Q4 profile. It remains attractive when cold prefill
dominates, but does not supply the selected three full histories. The optional
installed vLLM Quadlet remains its older 160K FP8 profile; these experimental
flags have not silently changed that unit.

### Quality and deployed verification

The original control, two-slot Q4 profile, and selected three-slot Q3 profile
each passed **12/12 extended functional checks**:

- Three bounded streaming probes.
- Generated Python contracts for interval merging, stable topological sort,
  environment parsing, chunked UTF-8/CRLF SSE parsing, bounded retry delay and
  FIFO token/slot admission. Code ran with resource limits in a disposable
  CPU-only container, not in the gateway or host process.
- Three seeded nested tool-call/follow-up checks.

After installation, the actual gateway passed three repetitions of
single/pair/three-request streaming. Three-way aggregate rates were
**180.40, 181.45 and 181.14 tok/s**, with three active slots and overlapping
streams. A four-request test queued correctly; an oversized ~265K request was
rejected with HTTP 400, followed by successful normal inference.

`make verify`, `make verify-history`, serialization audit and native client
checks passed. CPU vision correctly identified the local red-square image.
Three concurrent native OpenCode sessions each emitted reasoning, completed
a `read` tool call and returned the correct independently tagged heading.
Those results are saved as `native-opencode.json`.

These are bounded functional checks, not SWE-bench, broad language evaluation
or proof that Q3 weights/Q4 cache match the original model's quality on all
tasks. The lowered precision is the explicit price of the selected capacity.

## Reproduction

Host commands below can be run from toolbox with `flatpak-spawn --host`.
Model preparation is separate from GPU experiments; finish downloads before
starting candidates, since private SELinux volume labels must not be changed
by a concurrent downloader. Separate directories hold Q3 and Q4 artifacts.

```bash
make download                         # selected weights, projector, template
scripts/download-qwen38-candidates gguf  # needed for baseline/Q4 comparisons
scripts/download-local-helper         # optional 4B CPU-helper comparison

python3 scripts/tune-concurrency \
  --profiles baseline,q4-fixed2,q3-projector-cpu-fixed3 \
  --mode extended --output-dir docs/benchmarks/my-concurrency-run

python3 scripts/tune-concurrency \
  --profiles q3-projector-cpu-fixed3 --mode full \
  --output-dir docs/benchmarks/my-concurrency-run

python3 scripts/summarize-concurrency docs/benchmarks/my-concurrency-run
```

`tune-concurrency --help` documents modes; candidate names/arguments are in
`PROFILES` in the script. CPU helper profiles are deliberately run separately
from GPU candidates. `helper-gpu` also runs separately and stops/restores the
main GPU service. Never run two tuning controllers at once. Results for a
repeated profile/mode overwrite that pair of files, so use a new output
directory to retain separate runs. A killed host/process can require manual
candidate removal and `make start`; normal handled exits restore the service.

Existing standalone entry points are now:

- `scripts/run-coder-candidate gguf`: selected three-slot capacity profile.
- `scripts/run-coder-candidate gguf-quality`: Q4 weights, Q4 cache, two slots.
- `scripts/run-coder-candidate gguf-baseline`: historical one-slot Q4/Q8 setup
  with its original embedded-template settings.
- `scripts/run-coder-candidate nvfp4`: older optional FP8/160K setup.

Production checks do not replace/restart the backend:

```bash
make verify-concurrency
make client-concurrency
python3 scripts/verify-concurrency --queue-check
python3 scripts/verify-concurrency --limit-check
```

Native client limits remain 256K/16K, so **no OpenCode configuration edit or
restart is required for this capacity change**. A request interrupted during
the experiment window can simply be retried. Existing OpenAI provider access
was not edited; no remote requests participated in tuning.

## Global profile switching

```bash
make profile-qwen38-q4   # Qwen3.8 UD-Q4_K_XL; two fixed 256K agents; GPU vision
make profile-qwen38-q3   # Qwen3.8 UD-Q3_K_XL; three fixed 256K agents; CPU vision
make profile-status      # selected profile and actual running model/slots
```

Both profiles use Q4 K/V cache and a 16 GiB RAM prompt-cache limit. The same
`brain/coder-*` aliases and 256K/16K client limits apply to either profile.
Higher weight precision is not a measured guarantee of higher intelligence:
both profiles passed the same bounded coding contracts.

This is a **global backend switch**, not a per-agent model choice. Run it
between tasks, and avoid starting new requests until it finishes. The switcher
refuses when inference is active at its final pre-switch check; it does not
provide an admission gate or automatic draining for requests arriving after
that check. Backend and gateway restart briefly interrupt availability.
Open WebUI and native client configuration are not restarted or rewritten.

`config/coder-profiles.yaml` defines the two profiles. Selection updates the
model record in `config/models.lock.yaml`, the repository llama Quadlet and
its installed copy. It persists across reboot and subsequent `make deploy`:
the default downloader follows the recorded selection. These checkout changes
are intentional and remain available for Git review; the switcher never commits.

The switcher first stages the target configuration, verifies model/projector/
template hashes and image presence, and runs the Fedora Quadlet generator.
It then saves a private snapshot, installs the selected llama unit, reloads
systemd, restarts the backend followed by LiteLLM, checks actual container
arguments and performs real inference. Repeating an already-active selection
does not restart the services. A shared lock prevents overlap with another
switch or the concurrency tuner.

Switching is offline and does not automatically download missing artifacts.
Both weight files are already present on Brain. For a new installation, prepare
Q4 artifacts with `make download-precision` before selecting precision; prepare
Q3 with `scripts/download-capacity-model` before selecting capacity. The
projector/template must also exist (`make download` prepares the currently
selected profile). Finish preparation before switching or benchmarking.

Validation without changing the deployment is available on the host:

```bash
python3 scripts/switch-coder-profile precision --check
python3 scripts/switch-coder-profile capacity --check
```

On an activation/readiness/inference failure, the switcher restores the exact
previous repository files and installed unit, restarts the previous backend/
gateway, verifies inference, and returns nonzero. Snapshots are stored in
`~/.local/share/ai-station/backups/coder-profile-<timestamp>-<id>/` with the before/
after files and selection metadata. Handled SIGINT/SIGTERM also trigger recovery
once mutation has begun. Power loss or SIGKILL cannot be handled automatically;
restore the three `.before` files to the two checkout paths and installed
llama unit, then reload/restart and verify. Snapshots are private local state,
not committed artifacts, and retention is manual.

Verification performed: staged checks for both profiles; actual precision
activation and two-way streaming; a deliberately failed post-activation smoke
test with successful automatic rollback to precision; switching back to
capacity; and idempotent selection without replacing the backend container.
An attempted switch during a 1,024-token inference request was refused, and
the original request completed normally.

## Original one-slot rollback

For an exact return to the **original one-slot profile**, review and restore
these files from pre-tuning commit `82d3a38`: `config/models.lock.yaml`,
`quadlets/llama-coder.container`, and `scripts/download-coder-candidate`.
The new preflight accepts that recorded Q4/Q8/one-slot profile. Install/reload,
restart the backend and verify as above. Model files are retained, so rollback
does not require deleting or re-downloading the selected Q3 artifact.
