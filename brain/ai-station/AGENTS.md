# AI Station agent notes

The [docs/](docs/README.md) directory is the index — read it for how-to. This file holds the operational essentials that must be known before touching the running stack.

## Deployment model

- Host is Fedora `brain`, user `geoder101`; run host commands there, prefix with `flatpak-spawn --host` from toolbox, use `systemctl --user`.
- Clients hit `coder-off`/`coder-low`/`coder-medium`/`coder-xhigh` through LiteLLM (`:4000/v1`); `coder` stays medium-compatible. Aliases and reasoning profiles: [concurrency.md](concurrency.md).
- `llama-coder` is default; `vllm-coder` optional. **Run only one GPU backend at a time** — switching rewrites LiteLLM routing and the Quadlet `After` dep: [docs/operations.md](docs/operations.md).
- `llama-fallback` is an on-demand CPU fallback (`:8001`, Qwen3.8-Q3), no `[Install]` so it does not autostart. `make profile-*` (`scripts/switch-coder-profile`) starts it before the GPU restart and drops it once verified, leaving it up if the swap fails; for the manual upgrade flow in [docs/operations.md](docs/operations.md) use `scripts/swap-coder-standby on/off`. Every `coder-*` alias routes `order: 1` → GPU, `order: 2` → CPU; the fallback serves only when the GPU backend is unreachable.
- Current profile: Ornith-1.5-35B-A3B **Q5_K_M** (`ornith-q5`), llama.cpp **b12182**, two 262144-token slots, GPU Q4 K/V + BF16 projector, 16 GiB RAM cache, temp 1.0, `ornith1.5-history-v1` template (524288 is aggregate, never a client limit): [docs/model-profile.md](docs/model-profile.md).
- Selection persists to `config/coder-profiles.yaml`, runs between tasks, validates staged units and rolls back on failure; internal IDs stay `capacity`/`precision`, and selection is done via the `make profile-*` targets. `make backend-status` reports the active backend (GPU + profile, or CPU fallback); `make swap-active-cpu` / `make swap-active-gpu` temporarily reassign the GPU (documented in docs/operations.md).
- LiteLLM's Quadlet mounts `config/litellm-config.yaml` (deployment assumes this checkout path).

## Changes and verification

- Keep [docs/model-profile.md](docs/model-profile.md) as current config first, dated history below; append rather than replace on model/engine changes.
- Before committing, document any near-significant change — a new or renamed script, any Quadlet `Exec` argument, a lockfile record (model/chat-template/image selection), or anything that changes how a command switches or verifies a backend — with a one-line note here and a fuller write-up in docs/ (see docs/README.md). Trivial edits (whitespace, comments) need not be documented.
- `make help` lists workflows (host commands auto-wrap `flatpak-spawn --host`); `make deploy` backs up WebUI, pulls/downloads, installs/reloads, waits for readiness (backend→gateway→UI) and smoke-tests: [docs/automation.md](docs/automation.md).
- `scripts/install-quadlets` runs `scripts/preflight` + Quadlet generator validation before copying units; `make preflight` and `make validate` run them independently.
- Deployment order: `create-secrets`, `pull-images`, `download-coder-candidate`, `validate-quadlets`, `install-quadlets`, `daemon-reload`. **Download before install/reload** (default units `WantedBy=default.target`); repo Quadlet edits need reinstall + reload + restart.
- All containers `Pull=never`; `pull-images` pulls every digest in `config/images.lock.yaml`. Image changes must sync the lockfile, Quadlets and hardcoded digests (preflight rejects lockless images); model/chat-template lockfiles are records (except `images.lock.yaml` and `models.lock.yaml`) — keep filenames, revisions and directory names consistent when changing profiles.
- Smoke-test after startup with `scripts/smoke-test` or `scripts/smoke-test http://192.168.1.11:4000` (URL excludes `/v1`); `/health/liveliness` alone does not exercise the backend.
- Benchmarking (`scripts/run-coder-candidate gguf|nvfp4`) refuses to run while a backend is active: [docs/benchmarking.md](docs/benchmarking.md); `scripts/tune-concurrency` measures profiles and stops/restores the backend — run controllers single-threaded.

## Operational constraints

- Brain runs native OpenCode 1.19.5 without OmO (`make client-check` validates the runtime); do not install Home's OmO config on Brain.
- Secrets live in `~/.config/ai-station/{litellm,open-webui}.env` (outside the repo); `scripts/create-secrets` never overwrites them. Runtime state lives in `~/.local/share/ai-station/`.
- Keep `[Quadlet] DefaultDependencies=false` (host network-online caused startup timeouts).
- Both GPU Quadlets need CDI (`AddDevice=nvidia.com/gpu=all`) + `SecurityLabelDisable=true`; keep that SELinux exception GPU-only.
- Generated services can't be `systemctl disable`; remove `[Install] WantedBy` and reinstall/reload to change autostart.
- Client limits and backend context must agree (`/v1/models` isn't authoritative); context 262144 GGUF / 163840 NVFP4, client output cap 16384. Restart LiteLLM after routing/context changes, OpenCode after client-limit edits.
- Back up Open WebUI (incl. SQLite WAL) before upgrades; its DB holds connection keys that can override env values — a saved key may need `LITELLM_MASTER_KEY`, not a new LiteLLM database.
