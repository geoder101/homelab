# AI Station agent notes

## Deployment model

- This deploys rootless Podman Quadlets on Fedora host `brain` as user `geoder101`. Run host commands there; from toolbox, prefix them with `flatpak-spawn --host`. Use `systemctl --user`.
- Clients request `coder-off`, `coder-low`, `coder-medium`, or `coder-xhigh` through LiteLLM (`:4000/v1`); `coder` remains medium-compatible. Open WebUI (`:3000`) also uses LiteLLM. Backends have no published host ports.
- `llama-coder` is the default backend; `vllm-coder` is an optional alternative. Run only one GPU backend at a time. Switching requires changing LiteLLM routing and its Quadlet `After` dependency; see `docs/operations.md` for persistent selection.
- Current capacity profile: Qwen3.8-27B UD-Q3_K_XL, three fixed 262144-token slots (`--ctx-size 786432 --parallel 3 --no-kv-unified`), GPU Q4 K/V, CPU projector, 16 GiB RAM prompt cache. 786432 is aggregate, never a client context limit. See `docs/concurrency.md` for measured alternatives and rollback.
- `make profile-capacity` / `make profile-precision` perform persistent global selection from `config/coder-profiles.yaml`; precision uses Q4 weights, two fixed 256K slots and GPU vision. Both retain Q4 KV/16 GiB RAM cache and all client aliases/limits. `make profile-status` checks selection versus runtime. Run between tasks; switches validate staged artifacts/units and roll back on failure. The downloader follows `models.coder.profile`, so subsequent deploys preserve selection.
- LiteLLM's Quadlet mounts config from `%h/dev/projects/geoder101/ai-station/config/litellm-config.yaml`; deployment assumes this checkout location.

## Changes and verification

- `make help` lists the supported workflows; host commands automatically use `flatpak-spawn --host` from toolbox. `make deploy` backs up WebUI, downloads pinned artifacts, installs/reloads, waits for readiness in backend→gateway→UI order, and smoke-tests inference. Make serializes deployment even with `-j`. See `docs/automation.md`.
- `scripts/install-quadlets` now runs `scripts/preflight` (Python 3 + PyYAML) and generator validation before copying units. Preflight hashes the model/projector/template and checks default image pins/presence. Use `make preflight` independently; it reads no secret values.

- Run `scripts/validate-quadlets` on the Fedora host: it invokes the Podman generator with `--user --dryrun` against repository units. There is no configured build, lint, or unit-test suite.
- Deployment order: `scripts/create-secrets` (initial setup), `scripts/pull-images`, `scripts/download-coder-candidate`, `scripts/validate-quadlets`, `scripts/install-quadlets`, then `systemctl --user daemon-reload`. Complete downloads **before install/reload**: default units have `WantedBy=default.target`.
- `scripts/install-quadlets` copies files into `${XDG_CONFIG_HOME:-$HOME/.config}/containers/systemd`; edits to repository Quadlets need reinstall + daemon-reload, then restart affected services. Start the backend first, then LiteLLM, then Open WebUI; wait for backend readiness before testing.
- All containers use `Pull=never`. `scripts/pull-images` reads `Image=` from Quadlets, not `config/images.lock.yaml`. Image updates must synchronize the lockfile, Quadlets, and any matching hardcoded digests in `scripts/`.
- Model lockfiles are records, not downloader inputs. Keep revisions, filenames and directories in `config/{models,candidates}.lock.yaml`, `scripts/download-qwen38-candidates`, backend Quadlets and `scripts/run-coder-candidate` consistent when changing profiles. Even GGUF downloads use the pinned vLLM image's `hf` executable, so pull images first.
- `scripts/download-coder-candidate` also downloads the immutable chat template. Keep `config/chat-templates.lock.yaml`, `scripts/download-chat-template`, and the llama Quadlet's template volume revision synchronized. Download it before install/reload. See `docs/reasoning-profiles.md` for per-alias reasoning checks and rollback.
- After startup, run `scripts/smoke-test` or `scripts/smoke-test http://192.168.1.11:4000`. Its URL argument excludes `/v1`; it reads the host key and performs real inference. LiteLLM `/health/liveliness` alone does not test the backend.
- Benchmarking is a separate GPU workflow: `scripts/run-coder-candidate gguf|nvfp4` refuses to run while either deployed backend is active. Consult `docs/benchmarks/README.md` and `docs/benchmarking.md` for results and procedure.
- `scripts/tune-concurrency` runs measured profiles, stops/restores the deployed backend, and saves synthetic results/logs. Finish model downloads before experiments; private SELinux labels make concurrent download/mount operations unsafe. CPU helper profiles retain the GPU main model; GPU helper testing stops it. Do not run controllers concurrently. `make verify-concurrency` and `make client-concurrency` exercise the deployed three-slot stack without restarting it.

## Operational constraints

- Brain uses native OpenCode 1.19.4 without OmO. Its active config has real `brain/coder-*` aliases, `reasoning_options: []`, and `reasoning_content` history metadata; `make client-check` validates the resolved local runtime. This is distinct from the offline Home Downloads copies. Quit/restart OpenCode after client config changes; do not install Home's OmO configuration on Brain.

- Secrets live in `~/.config/ai-station/{litellm,open-webui}.env`, outside the repository. `scripts/create-secrets` refuses to overwrite either existing file. Model/cache/UI state lives under `~/.local/share/ai-station/`.
- Keep `[Quadlet] DefaultDependencies=false`: the host's network-online dependency caused startup timeouts despite all deployment artifacts being local.
- Both GPU Quadlets require NVIDIA CDI (`AddDevice=nvidia.com/gpu=all`) and `SecurityLabelDisable=true` for host driver access. Keep that SELinux exception limited to GPU containers; ADR 0002 predates the llama.cpp backend.
- Generated Quadlet services cannot be disabled with `systemctl disable`; remove their `[Install]` `WantedBy` entry and reinstall/reload to change automatic startup.
- Backend context settings and OpenCode client limits must agree explicitly; `/v1/models` is not authoritative for limits. Current GGUF context is 262144, NVFP4 is 163840; client output cap is 16384. See `docs/operations.md`; restart LiteLLM after routing/context changes and restart OpenCode after client-limit edits.
- Stop Open WebUI and back up its data directory, including SQLite WAL files, before upgrades. Its database persists connection keys that can override environment values; an empty model list can require syncing the saved key to `LITELLM_MASTER_KEY`, not adding a LiteLLM database.
