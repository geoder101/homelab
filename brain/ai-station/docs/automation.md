# Automation, managed state, and Brain's native OpenCode client

The current three-agent profile and local-only tuning measurements are in
[capacity tuning](concurrency.md). The client section below records the earlier
native-client setup; capacity tuning preserves its 256K per-agent limits.

## Makefile entry points

Run `make help` from the repository. The default target only prints help.
Host-management targets automatically use `flatpak-spawn --host` from toolbox;
the same Makefile runs directly on Fedora. Paths are anchored to the checkout,
so `make -C /path/to/ai-station TARGET` also works.

Host prerequisites are GNU Make, Bash, Podman/Quadlets, curl, Python 3.11+
and PyYAML, plus NVIDIA CDI for inference. These were present on Brain during
verification. `validate-home` additionally needs Python `jsonschema` in the
environment running Make. Native OpenCode targets use that environment's
`opencode` executable, matching the actual client rather than assuming a host
and toolbox installation are interchangeable.

### Initial deployment and routine updates

```bash
make bootstrap    # first installation: create secrets, then deploy
make deploy       # subsequent deployments; existing secrets are retained
```

`bootstrap` calls the existing secret generator, which refuses to overwrite
existing secrets. On an already configured host, use `deploy`. The one-time
linger setup remains an explicit administrative command documented in
[operations](operations.md); Make does not request sudo.

`deploy` executes these operations in order:

1. Back up existing Open WebUI data while its service is stopped; resume it
   if it was running. On initial installation, absent UI data is skipped.
2. Pull the pinned images recorded in `config/images.lock.yaml`.
3. Download the pinned default model, projector and chat template.
4. Verify local artifacts and run the Fedora Quadlet generator; only then
   install the Quadlets.
5. Reload user systemd.
6. Restart llama-coder and wait for its real `/health` readiness endpoint.
7. Restart LiteLLM and wait for gateway liveness, then restart Open WebUI and
   wait for its `/health` endpoint.
8. Perform actual inference through LiteLLM with the existing smoke test.

The Makefile disables parallel target execution, including with `make -j`,
because downloads must finish before install/reload can activate `WantedBy`
services. Individual steps remain available:

| Target | Action |
| --- | --- |
| `secrets` | Initial secret-file creation only |
| `pull` | Pull pinned images from `config/images.lock.yaml` |
| `download` | Download default GGUF, projector and template |
| `download-template` | Download/check only the selected template |
| `preflight` | Hash artifacts; check lock/Quadlet agreement, default images and secret-file presence |
| `validate` | Fedora user Quadlet generator dry-run; no artifact requirement |
| `install` | Preflight + generator check + copy Quadlets |
| `reload` | User systemd daemon reload |
| `start` / `restart` | Readiness-ordered stack start/restart |
| `status` / `logs` | Service status / follow backend journal |
| `backup-webui` | Consistent private archive of persistent UI data |
| `smoke` | Quick real inference through LiteLLM |
| `audit` | Installed LiteLLM serialization checks, with mocked HTTP transport |
| `verify` | Live alias, reasoning, streaming and tool round trips |
| `verify-stress` | Verification plus bounded coding-plan tasks |
| `verify-budgets` | Exercise the low/medium thinking ceilings |
| `verify-history` | Tool-error recovery and embedded local image request |
| `client-check` | Resolve and check Brain's native OpenCode settings and models |
| `client-smoke` | A real OpenCode off-profile request, with JSON streaming events |
| `verify-concurrency` | Concurrent streaming through the actual LiteLLM gateway |
| `client-concurrency` | Three independent native OpenCode read-only tool sessions |
| `profile-qwen38-q3` | Qwen3.8-27B UD-Q3_K_XL, b11960, three 256K slots (internal ID `capacity`) |
| `profile-qwen38-q4` | Qwen3.8-27B UD-Q4_K_XL, b11960, two 256K slots (internal ID `precision`) |
| `profile-ornith-q4` | Experimental Ornith Q4_K_M, b11960, two 256K slots (internal ID `ornith`) |
| `profile-ornith-q5` | Current Ornith Q5_K_M selection, b12182 engine, two 256K slots and GPU vision |
| `profile-status` | Show selected and actual running profile |
| `download-precision` | Prepare Q4 weights/projector before offline precision switching |
| `download-ornith` | Prepare pinned Ornith weights/projector and history-fixed template |
| `download-ornith-q5` | Prepare the selected Q5 weights/projector and history-fixed template |
| `validate-home` | Validate the two separate offline Downloads copies |

The model-specific `profile-*` targets are the canonical selection commands.
The earlier targets have been removed: replace `profile-capacity` with
`profile-qwen38-q3`, `profile-precision` with `profile-qwen38-q4`, and
`profile-ornith` with `profile-ornith-q4`.
The internal profile IDs in existing model records stay `capacity`, `precision`
and `ornith`, so historical selections and direct script commands remain valid.

Gateway tests default to `http://brain.local:4000`. Override it without `/v1`:

```bash
make smoke BASE_URL=http://192.168.1.11:4000
make verify BASE_URL=http://brain.local:4000
```

`BASE_URL` changes the probe address, not LiteLLM routing or client settings.
`WEBUI_URL` controls the UI health probe and defaults to `http://127.0.0.1:3000`.

Profile switching checks artifacts/units before applying changes, snapshots the
previous state, restarts only the backend and gateway, and rolls back on failure.
It persists the selection in the checkout and installed unit; `make download`
and `make deploy` follow it. Run between tasks. See
[global profile switching](concurrency.md#global-profile-switching) for details.
`HOST=` can explicitly disable the toolbox wrapper; normally leave it automatic.

The workflow targets the default **llama.cpp** deployment. Startup refuses
an active vLLM service, running vLLM container or GPU benchmark candidate.
Alternative GPU benchmarking and persistent NVFP4 switching retain their
separate explicit procedures in `docs/benchmarking.md` and `docs/operations.md`.

### Artifact preflight

`scripts/install-quadlets` itself invokes `scripts/preflight` and the Quadlet
generator, so direct script usage gets the same checks as Make. Failure occurs
before creating the install target directory or copying units.

Preflight reads the model, chat-template and image lockfiles and checks:

- Exact model/projector filenames and template paths agree with the llama
  Quadlet's arguments and mounts.
- The selected template version agrees with its immutable revision directory.
- Full SHA256 hashes match for model weights, projector and template. This
  reads the large GGUF file; expect more I/O than a file-existence check.
- Images required by automatically started containers are pinned in the image
  lock and already present in Podman storage. Optional backend weights are not
  required for the default deployment.
- Both required secret files exist. Their values are not read or printed.

Preflight performs no download, model loading or service changes. For a
deliberate embedded-template rollback, set the model lock's `chat_template` to
`embedded` and remove the explicit template arguments/volume from the llama
Quadlet. Other model/hash checks continue to apply. See the more complete
[rollback instructions](reasoning-profiles.md#rollback).

## What belongs in Git, downloads, or backups

No symlink or bidirectional state synchronization is needed for templates.
The repository records **which immutable bytes to deploy**; the downloader
materializes them under `~/.local/share/ai-station/`. The template mount is
revision-specific and read-only. Do not edit that downloaded file in place.

| State | Treatment |
| --- | --- |
| Model weights/projector | Recreate with pinned downloader; verify against model lock hashes |
| Chat templates | Recreate from pinned revision/checksum; immutable revision directories |
| Hugging Face/download cache | Reusable cache; not authoritative deployment configuration |
| Open WebUI data | Back up; contains persistent accounts, chats, connections and keys |
| Secrets | Stay outside Git under `~/.config/ai-station/`; protect separately |
| Selected benchmark results | Record intentionally in `docs/benchmarks/` |
| Client configuration backup | Private local rollback copy, outside Git |

`make backup-webui` writes a mode-0600 compressed archive under the mode-0700
`~/.local/share/ai-station/backups/` directory. The stopped-service snapshot
includes SQLite databases and any WAL files. Gzip integrity is checked before
the helper reports success. The archive includes private UI state, so treat
it like a secret-bearing backup. Retention and off-machine backup storage are
manual; Make does not prune or upload data.

To restore a UI snapshot, stop Open WebUI, move its current data directory
aside, extract the selected archive into `~/.local/share/ai-station/`, then
start Open WebUI. For an upgrade rollback, also select the matching previous
Open WebUI image; restoring an old database against a migrated/new schema is
not assumed safe. No archive restore was performed during this work.

## Brain's simple native OpenCode configuration

This is separate from the **Home** handoff and its offline Downloads files.
The user subsequently authorized updating Brain's active config:

```text
~/.config/opencode/opencode.jsonc
```

Brain runs **OpenCode 1.19.5**, checked locally on 2026-11-08. Its config contains no external
plugins and no OmO settings. No packages were installed or updated. The
existing `@opencode-ai/plugin` package dependency is an SDK dependency, not an
OmO configuration.

The client keeps provider `brain`, `@ai-sdk/openai-compatible`, its local
`http://127.0.0.1:4000/v1` gateway and the existing API-key file reference.
Built-in authenticated providers remain available alongside `brain`; the
accidental provider allowlist was removed. Defaults use Brain, and the local
gateway has no remote model fallback.

| Use | Model |
| --- | --- |
| Primary/default, including native build and plan | `brain/coder-medium` |
| Small/background model | `brain/coder-off` |
| Explore and title | `brain/coder-off` |
| Summary and compaction | `brain/coder-low` |
| Explicit deeper reasoning choice | `brain/coder-xhigh` |
| Existing sessions referencing the old ID | `brain/coder` remains available |

All five model entries declare 262,144 context / 16,384 output, tools and
image input. Thinking models declare
`interleaved: {"field":"reasoning_content"}` to resend supplied reasoning in
assistant history. `reasoning_options: []` suppresses OpenCode's automatically
generated effort variants; select the actual LiteLLM model aliases instead.
Both fields were originally checked against the **1.19.4 source and installed runtime**;
the native client checks also passed on 1.19.5 during the 2026-11-05 trial.
The 2026-11-08 Ornith Q5 backend selection keeps the same client configuration;
see [OpenCode guidance](opencode.md) and the [post-deployment findings](deployments/20261108-ornith-q5/README.md).
The current published editor schema may lag these runtime-supported fields.

The original config was backed up before editing:

```text
~/.local/share/ai-station/backups/opencode-before-profiles-20261031.jsonc
```

Restoring that file over the active config restores the previous native client
settings. The backup contains a key-file reference, not the actual API key.

### Restart and verification

**Quit and restart Brain's OpenCode once after this configuration update.**
The running session keeps its previously loaded config. New CLI processes
already pick up the updated settings. Existing sessions can retain a saved
model choice; select `brain/coder-medium` or start a new session if necessary.
No inference-service restart is needed for client-only edits.

```bash
make client-check
make client-smoke
opencode run --model brain/coder-medium --thinking --format json \
  'Use the read tool to inspect README.md and report its first Markdown heading. Do not edit files or use bash.'
```

These checks were actually performed. The resolved runtime listed all five
models with empty variants, correct limits and reasoning-history metadata.
The off request returned `READY` with **zero reasoning tokens**. The medium
request emitted reasoning, completed a structured `read` tool call and a
second inference turn, then returned `# AI Station`. Build/plan inherit the
medium default, while the four auxiliary agent overrides resolved as above.
Only new verification sessions were created; the user's existing session was
not restarted or rewritten.

### Automation verification performed

- `make help`, automatic toolbox wrapper and ordered `make -j8 -n deploy`.
- `make preflight`, `make validate`, `make audit`, and `make smoke` passed.
- An install attempt with an empty temporary HOME failed for the missing
  template before creating its intended installation directory.
- `make start` passed against the already running stack (idempotent start).
- `make backup-webui` created and gzip-checked a real archive, then returned
  the previously running UI to active status. All three services were active.
- Native client checks and actual off/medium inference passed as above.
- Shell/Python syntax and `git diff --check` are checked before completion.

The complete `make deploy` path was dry-run, not executed end-to-end during
this follow-up: pinned artifacts were already present, and no backend/image
upgrade was required. Restart timeout/failure paths and archive restoration
were not exercised against a deliberately failing production service.
