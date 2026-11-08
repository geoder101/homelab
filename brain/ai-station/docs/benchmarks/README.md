# Qwen3.8 coding-agent comparison

**Current selection (2026-11-08):** Ornith Q5_K_M on b12182, selected after the
[expanded comparison](moe-20261107-queue/SUMMARY.md). See [current configuration
and retained Qwen history](../model-profile.md) and [deployed verification/limitations](../deployments/20261108-ornith-q5/README.md).

**Earlier trial (2026-11-05):** [Ornith Q4_K_M versus Qwen precision](ornith-20261105.md).
Q4 was faster and matched the small coding suite but failed native checks;
Qwen precision was retained at that time.

**Capacity-study results:** [fully local concurrency tuning](../concurrency.md),
with raw results in `concurrency-20261031/`. The comparison below is historical;
its Q4 weights/one-slot settings are not the current Q4/two-slot precision deployment.

The 2026-10-29 comparison below uses the original embedded template. The
2026-10-31 production alias/template checks, measured reasoning lengths and
budget tests are recorded in [reasoning profiles](../reasoning-profiles.md).
Those checks run against the existing production backend, not a second GPU
candidate instance.

Pinned evaluation artifacts are recorded in `config/candidates.lock.yaml`.
The comparison uses the same Qwen3.8-27B family, medium thinking, no MTP,
and the sampling parameters recommended by the
[Unsloth guide](https://unsloth.ai/docs/models/qwen3.8).

The two candidates are vLLM/NVFP4 with calibrated FP8 attention KV cache,
and llama.cpp Dynamic V3 `UD-Q4_K_XL` with Q8 K/V cache. Both load their vision
encoder/projector, but this comparison tests text-based coding-agent behavior.
Both candidate profiles schedule one request at a time, matching a single
coding agent and avoiding reservations for large concurrent batches. Set
`CONTEXT_TOKENS` when launching the vLLM candidate to test a different context cap.

The small suite in `scripts/benchmark-coder` checks:

- Streamed first-token latency and generation rate (three 512-token-budget runs;
  the first starts cold, subsequent runs can reuse the prompt prefix).
- Generated Python functions against edge-case tests, in a disposable CPU-only
  benchmark container, with a constrained child process.
- Three nested JSON tool calls at seeds 42–44 followed by tool-result handling.
- Exact retrieval from the beginning, middle and end of long synthetic context,
  followed by another turn over the same history to measure cache reuse.

These checks do not establish SWE-bench performance or broad quality parity.
Speed figures from the Unsloth page are not assumed to apply to an RTX 5090.
Cold versus warm prefix cache and different kernels can affect timings.

Run downloads from a host terminal:

```bash
scripts/download-qwen38-candidates nvfp4
scripts/download-qwen38-candidates gguf
podman pull ghcr.io/ggml-org/llama.cpp@sha256:ae676f2ecfdeb66db48b7c32f1f013c62cdbc3e380b1f6e8b425c981dd34ac99
```

Stop the active production backend before starting either GPU candidate. Run only one at a time:

```bash
systemctl --user stop llama-coder vllm-coder
scripts/run-coder-candidate nvfp4  # or gguf-baseline, in a separate host terminal
```

After candidate readiness, run the suite from the repository on the host:

```bash
podman run --rm --pull never --network ai-station --memory 2g --cpus 4 \
  --security-opt no-new-privileges --cap-drop all \
  --volume "$PWD/scripts/benchmark-coder:/benchmark.py:ro,z" \
  --volume "$PWD/docs/benchmarks:/results:z" \
  --entrypoint python3 \
  docker.io/vllm/vllm-openai@sha256:6ee133356d49ed24cee03dd0cda3d3980279a11c107ce3e752c89e76540f41c9 \
  /benchmark.py --backend vllm --label qwen38-nvfp4 \
  --contexts 32000,122000,154000 --output /results/qwen38-nvfp4.json
```

For GGUF, use `--backend llama --label qwen38-gguf-q4-k-xl`, add a
`253000` context probe, and write `/results/qwen38-gguf-q4-k-xl.json`.
The results capture actual API-reported token counts, not just target sizes.

Stop the candidate container before restoring production:

```bash
podman stop ai-station-coder-candidate
systemctl --user start llama-coder
```
