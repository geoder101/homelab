# AI Station agent notes

## Deployment model

- This deploys rootless Podman Quadlets on Fedora host `brain` as user `geoder101`. Run host commands there; from toolbox, prefix them with `flatpak-spawn --host`. Use `systemctl --user`.
- Clients request stable model alias `coder` through LiteLLM (`:4000/v1`); Open WebUI (`:3000`) also uses LiteLLM. Backends have no published host ports.
- `llama-coder` is the default backend; `vllm-coder` is an optional alternative. Run only one GPU backend at a time. Switching requires changing LiteLLM routing and its Quadlet `After` dependency; see `docs/operations.md` for persistent selection.
- LiteLLM's Quadlet mounts config from `%h/dev/projects/geoder101/ai-station/config/litellm-config.yaml`; deployment assumes this checkout location.

## Changes and verification

- Run `scripts/validate-quadlets` on the Fedora host: it invokes the Podman generator with `--user --dryrun` against repository units. There is no configured build, lint, or unit-test suite.
- Deployment order: `scripts/create-secrets` (initial setup), `scripts/pull-images`, `scripts/download-coder-candidate`, `scripts/validate-quadlets`, `scripts/install-quadlets`, then `systemctl --user daemon-reload`. Complete downloads **before install/reload**: default units have `WantedBy=default.target`.
- `scripts/install-quadlets` copies files into `${XDG_CONFIG_HOME:-$HOME/.config}/containers/systemd`; edits to repository Quadlets need reinstall + daemon-reload, then restart affected services. Start the backend first, then LiteLLM, then Open WebUI; wait for backend readiness before testing.
- All containers use `Pull=never`. `scripts/pull-images` reads `Image=` from Quadlets, not `config/images.lock.yaml`. Image updates must synchronize the lockfile, Quadlets, and any matching hardcoded digests in `scripts/`.
- Model lockfiles are records, not downloader inputs. Keep revisions, filenames and directories in `config/{models,candidates}.lock.yaml`, `scripts/download-qwen38-candidates`, backend Quadlets and `scripts/run-coder-candidate` consistent when changing profiles. Even GGUF downloads use the pinned vLLM image's `hf` executable, so pull images first.
- After startup, run `scripts/smoke-test` or `scripts/smoke-test http://192.168.1.11:4000`. Its URL argument excludes `/v1`; it reads the host key and performs real inference. LiteLLM `/health/liveliness` alone does not test the backend.
- Benchmarking is a separate GPU workflow: `scripts/run-coder-candidate gguf|nvfp4` refuses to run while either deployed backend is active. Consult `docs/benchmarks/README.md` and `docs/benchmarking.md` for results and procedure.

## Operational constraints

- Secrets live in `~/.config/ai-station/{litellm,open-webui}.env`, outside the repository. `scripts/create-secrets` refuses to overwrite either existing file. Model/cache/UI state lives under `~/.local/share/ai-station/`.
- Keep `[Quadlet] DefaultDependencies=false`: the host's network-online dependency caused startup timeouts despite all deployment artifacts being local.
- Both GPU Quadlets require NVIDIA CDI (`AddDevice=nvidia.com/gpu=all`) and `SecurityLabelDisable=true` for host driver access. Keep that SELinux exception limited to GPU containers; ADR 0002 predates the llama.cpp backend.
- Generated Quadlet services cannot be disabled with `systemctl disable`; remove their `[Install]` `WantedBy` entry and reinstall/reload to change automatic startup.
- Backend context settings and OpenCode client limits must agree explicitly; `/v1/models` is not authoritative for limits. Current GGUF context is 262144, NVFP4 is 163840; client output cap is 16384. See `docs/operations.md`; restart LiteLLM after routing/context changes and restart OpenCode after client-limit edits.
- Stop Open WebUI and back up its data directory, including SQLite WAL files, before upgrades. Its database persists connection keys that can override environment values; an empty model list can require syncing the saved key to `LITELLM_MASTER_KEY`, not adding a LiteLLM database.
