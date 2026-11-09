# Operations

Run host-management commands from a host terminal. From toolbox, prefix them with `flatpak-spawn --host`.

The [Makefile workflow](automation.md) wraps these commands automatically:
`make help`, `make bootstrap` for first setup, or `make deploy` for updates.
The lower-level commands below remain available. Quadlet installation now
includes an artifact/image preflight and generator validation. Brain's active
native OpenCode config and its restart instructions are documented separately
in [automation](automation.md#brains-simple-native-opencode-configuration).

## Bootstrap

1. Generate secret files with `scripts/create-secrets`.
2. Enable lingering once with `sudo loginctl enable-linger geoder101`.
3. Pull locked images with `scripts/pull-images`.
4. Run `scripts/download-coder-candidate` to download the profile selected in `config/models.lock.yaml`. Current `ornith-q5` downloads pinned Q5_K_M weights and projector into `~/.local/share/ai-station/models/ornith-1.5-35b-gguf-q5` and prepares the checksum-pinned history-fixed template. The earlier experimental `ornith` profile downloads Q4 into the unsuffixed directory. Both model families store their external templates under `~/.local/share/ai-station/chat-templates/`; `scripts/download-chat-template` follows the selected family.
5. Run `scripts/validate-quadlets`.
6. Copy Quadlets with `scripts/install-quadlets`.
7. Reload and start `llama-coder`, then `litellm`, then `open-webui`.

`llama-coder`, `litellm` and `open-webui` have `[Install]` sections and start with the user systemd session after `daemon-reload`. The alternative `vllm-coder` has no automatic-start section. Finish the model download before installing/reloading the Quadlets. To prevent automatic startup, remove the unit's `WantedBy` entry and reload; generated Quadlet services cannot be disabled with `systemctl disable`.

## Select or upgrade the GGUF backend

For an already deployed stack, use `make profile-ornith-q5` (current Q5/b12182
selection), `make profile-qwen38-q4` (retained Qwen3.8 Q4/b11960),
`make profile-qwen38-q3` (Qwen3.8 Q3/b11960), or `make profile-ornith-q4`
(earlier experimental Q4 candidate) to
select a persistent global profile. All retain
256K per-agent context. These commands validate first, restart backend/gateway,
smoke-test and restore the previous profile on failure. `make profile-status`
checks the running selection. See [switching](concurrency.md#global-profile-switching).

Run from the repository on the host:

```bash
scripts/pull-images
scripts/download-coder-candidate
scripts/validate-quadlets
scripts/install-quadlets
systemctl --user daemon-reload
systemctl --user stop vllm-coder
systemctl --user restart llama-coder
systemctl --user restart litellm
journalctl --user -u llama-coder -f
# After llama.cpp reports that startup is complete:
scripts/smoke-test
scripts/smoke-test http://192.168.1.11:4000
```

Downloads resume when rerun. Previous checkpoints are retained. The API model alias remains `coder`.

The selected Ornith Q5 profile uses llama.cpp b12182, **two fixed 256K slots**
(524,288 aggregate tokens), Q5_K_M weights, GPU Q4 K/V, GPU vision and a
16 GiB RAM prompt cache. Medium is the default alias; MTP is disabled.
The pinned Ornith template preserves and normalizes supplied reasoning;
the aliases select hard thinking ceilings rather than Qwen's soft effort prompts.
See [current settings, exact server options and preserved selection history](model-profile.md),
[OpenCode configuration](opencode.md), and [deployment verification](deployments/20261108-ornith-q5/README.md).

### Optional NVFP4 profile

Download with `scripts/download-qwen38-candidates nvfp4`. To use the alternative profile, stop `llama-coder`, start `vllm-coder`, change LiteLLM's `api_base` to `http://vllm-coder:8000/v1` and restart LiteLLM. Set every OpenCode client's context to `163840`. For persistent selection across boot, move the `[Install] WantedBy=default.target` section from `llama-coder.container` to `vllm-coder.container`, update LiteLLM's `After` dependency, and reinstall/reload Quadlets. Never enable both GPU backends together.

The installed optional vLLM Quadlet still uses one sequence, calibrated FP8 KV
and utilization 0.94. Native 256K did not fit that profile. A separate **tested
NVFP4 KV-cache** candidate does support a 256K maximum with approximately 1.2
full-context capacity and faster cold prefill; it is not the installed optional
Quadlet. Reproduce it with `scripts/tune-concurrency`; see [results](concurrency.md).

### OpenCode context limit

On every OpenCode client using any `brain/coder-*` model (or legacy `brain/coder`), set the following in its `opencode.jsonc`, then quit and restart OpenCode:

```jsonc
"limit": {
  "context": 262144,
  "output": 16384
}
```

Input, reasoning and output must fit within the total context budget. These limits are client metadata used for request budgeting and compaction, not server settings. Keep them explicit for this custom provider: the standard `/v1/models` response is not a reliable source of context/output limits. A 16K output cap gives thinking and final output more room than the old 8K cap; it does not require generating 16K tokens on every turn. The 2026-10-31 Home handoff edits only `~/Downloads/{opencode,omo}.jsonc` on Brain; it does not install or activate client configuration there.

Restart LiteLLM after changing backend routing or context limits to clear cached metadata. llama.cpp's standard model listing identifies `coder` but does not advertise the context/output limits there. The server's per-slot context and explicit OpenCode limits are authoritative; `--ctx-size 524288` is aggregate across two precision slots, not a client context limit. Qwen capacity's `786432` is likewise aggregate across three slots.

The image locks contain vLLM 0.32.0, LiteLLM 1.106.5, Open WebUI 0.12.4, selected llama.cpp b12182 (`fd27b2034905f8dadd8bd6d2cc44595f4929d391`) and retained b11960 (`8642541b255bfb5dc39cd351a178be09013a55d6`). The lock file and Quadlets pin linux/amd64 manifest digests. Before starting an upgraded Open WebUI, stop it and back up its data directory (including SQLite WAL files); the new release can migrate the database.

## CPU warm-standby fallback backend

`llama-fallback` is a CPU-only fallback that is started on demand before a GPU
model swap and stopped afterwards to free RAM, keeping the coding agents
running across the swap without switching the client. It is a second
`llama.cpp` Quadlet (`quadlets/llama-fallback.container`) with no `[Install]`
section, so it does not autostart at boot. It serves the Qwen3.8-27B **Q3**
weights from the already-downloaded
`~/.local/share/ai-station/models/qwen3.8-27b-gguf-q3`, on the private
`ai-station` network as `llama-fallback:8001`. It loads one 262,144-token slot
and runs with no GPU, so it is never in contention with the selected profile's
GPU work.

It does not replace the selected GPU backend; it is a failover target only.
Every `coder-*` alias in `config/litellm-config.yaml` lists two deployments
under the same model name: the GPU `llama-coder:8000` backend at `order: 1`
and this CPU backend at `order: 2`. When the GPU backend is unreachable,
LiteLLM tries order 2 and a live model keeps answering; once the GPU backend
is healthy, order 1 serves again. Reload LiteLLM after editing the config.

### Riding through a GPU model swap

`make profile-*` (the common path for experimenting with new models) does this
automatically: it starts the fallback before restarting the GPU backend and
drops it once the switch is verified, leaving it up if the swap fails. The
manual procedure below is for the lower-level upgrade flow — i.e. manually
pulling images, downloading weights, and installing Quadlets (selecting or
upgrading the GGUF backend) rather than going through `make profile-*`.

Start the fallback before the swap so failover has a ready target, then stop
it afterwards to free RAM.

1. `scripts/swap-coder-standby on` — starts `llama-fallback` and waits until
   it serves `:8001`, so a ready target exists before the GPU backend is cut.
2. `systemctl --user stop llama-coder` — graceful stop; in-flight requests
   drain, the port closes, and new requests fail over to the CPU backend.
3. Perform the swap (download the pinned weights/template, reinstall the
   Quadlet, `daemon-reload`).
4. `systemctl --user start llama-coder` — traffic returns to the GPU backend.
5. `scripts/swap-coder-standby off` — stops `llama-fallback` to free RAM.

`scripts/swap-coder-standby on` and `off` are idempotent. `on` is a no-op when
the fallback already serves `:8001`; `off` clears the failed-state flag only
when present, so repeating either call is a harmless no-op.

During steps 1-5 the agents run on the local CPU fallback with no OpenCode
client change and no external provider. The fallback serves plain,
non-thinking output (its froggeric template does not carry the ornith
history-fixed template) and caps output at 16,384 tokens like the other
aliases. It is a degraded local model — measured roughly 200 tok/s in an early
probe — adequate as a fallback path, not a replacement for the Ornith Q5
profile.

Keep it running (skip step 5) if the swap fails: it keeps the agents alive as
a degraded local model until you repair the GPU backend. The fallback coexists
with the GPU backend under the order-based routing (GPU at order 1, fallback at
order 2, served only when the GPU is unreachable), so the two never race.

For instant failover instead of waiting on the graceful stop, `podman stop -t 0
ai-station-llama-coder` is acceptable: systemd does not restart a
user-initiated stop, and `Restart=on-failure` only fires on crashes. The
fallback cannot survive loss of the whole machine; it is a swap-time bridge,
not a second-host backstop.

### Temporarily reassign the GPU

`scripts/swap-active-backend <cpu|gpu>` hands inference to the CPU fallback so
the GPU can be used for another GPU task (e.g. a video game), and back again. It
does **not** change the selected profile, the model files, or the routing in
`config/litellm-config.yaml` — it only stops and starts the two running
backends and restarts LiteLLM so routing reads cleanly. Because every alias
routes order 1 to the GPU backend and order 2 to the CPU fallback, the clients
keep answering the whole time: with the GPU backend stopped requests fall
through to CPU, and once it is healthy again it serves directly.

```bash
scripts/swap-active-backend cpu   # stop llama-coder (GPU), start llama-fallback
scripts/swap-active-backend gpu   # stop llama-fallback, start llama-coder (GPU)
```

The `cpu` direction is safe by construction: `scripts/swap-coder-standby on`
starts the CPU fallback and waits for `:8001` before the GPU backend is cut, so
there is never a moment with no backend, and a trailing `scripts/smoke-test`
proves the `coder` alias still answers end-to-end. Run it between coding tasks
(it briefly interrupts the warm GPU slots). To restore, `gpu` stops the
fallback and restarts the GPU backend; if it does not come back, re-run `cpu`
to fall back to CPU and inspect with `journalctl --user -u llama-coder -f`.

### Check the active backend

`make backend-status` (or `scripts/backend-status`) reports which backend is
currently serving, without changing anything: `GPU: ornith-q5` when the
`llama-coder` backend is up (showing the selected profile), `CPU: fallback` when
only the CPU fallback is running, or `DOWN: no active backend` when both are
stopped. It reads `systemctl --user is-active` and the selected model lock, so
it is the right check after a `swap-active-backend` flip, where `make
profile-status` would instead fail because the GPU backend is temporarily down.

## API use

LAN:

```text
http://brain:4000/v1
```

Tailscale:

```text
http://brain.tail1fe190.ts.net:4000/v1
```

Use model `coder-medium` (or `coder-off`, `coder-low`, `coder-xhigh`, legacy `coder`) and the `LITELLM_MASTER_KEY` as the bearer token. Home clients use `http://brain.local:4000/v1`. Open WebUI is available on port 3000 at the same hostname. No **remote** model fallback is configured; a **local CPU backend** (`llama-fallback`) provides automatic failover across GPU swaps, documented under [CPU warm-standby fallback backend](#cpu-warm-standby-fallback-backend).

## Systemd commands

```bash
systemctl --user daemon-reload
systemctl --user start llama-coder
systemctl --user start litellm
systemctl --user start open-webui
systemctl --user status llama-coder litellm open-webui
journalctl --user -u llama-coder -f
```

## Firewall

The detected trusted LAN is `192.168.1.0/24`. Do not expose ports globally. Run these host commands once with interactive sudo authorization:

```bash
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="3000" protocol="tcp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="4000" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Port 4000 is an API endpoint; port 3000 is browser-only Open WebUI. Tailscale uses its own encrypted interface and was verified locally at `100.110.211.92`; test it from another tailnet device after the firewall change.

The active Wi-Fi interface `wlp9s0` was in zone `public` during the 2026-10-28 inspection, and both rules already existed in runtime and permanent configuration. Check `firewall-cmd --get-active-zones` before using these commands on a different connection.

## LAN troubleshooting

Check each layer from a host terminal:

```bash
systemctl --user status litellm llama-coder
podman port ai-station-litellm
ss -lnt 'sport = :4000'
curl --noproxy '*' --max-time 5 http://127.0.0.1:4000/health/liveliness
curl --noproxy '*' --max-time 5 http://192.168.1.11:4000/health/liveliness
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-rich-rules
```

From **another LAN device**, run:

```bash
curl --noproxy '*' --max-time 5 http://192.168.1.11:4000/health/liveliness
curl --noproxy '*' --max-time 5 http://192.168.1.11:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

Use the IP first to isolate hostname/DNS issues. A local request to the host's LAN IP does not prove remote firewall or Wi-Fi reachability. An HTTP 401 demonstrates connectivity but requires the correct API key. A successful liveness response only checks LiteLLM; use `scripts/smoke-test` to also verify inference. If remote access times out while local access works, check the active firewall zone and Wi-Fi client/guest isolation.

On 2026-10-28, the host's `network-online.target` was inactive and `podman-user-wait-network-online.service` timed out. The Quadlets now omit that implicit dependency via `[Quadlet] DefaultDependencies=false`, since all deployment artifacts are local. This fixes the startup wait, independently of port publishing and firewall rules.

LAN liveness was also confirmed from a second device during the upgrade. Existing firewall rules were sufficient; no firewall changes were needed.

## Open WebUI connects but lists no models

Open WebUI persists connection settings, including API keys, in its own database. A saved key can override `OPENAI_API_KEYS` from the environment. Testing with the environment key alone does not validate the connection used by the UI.

On 2026-10-28, the saved connection URL was correct (`http://litellm:4000/v1`), but its saved API key differed from `LITELLM_MASTER_KEY`. Using that saved key returned HTTP 400 with `No connected db`; the correct master key returned `coder`. LiteLLM attempted a database lookup for the unrecognized key. A LiteLLM database is not required for this static, master-key-authenticated deployment.

In Open WebUI's admin connection settings, set the connection's API key to the current `LITELLM_MASTER_KEY`, save, and refresh the model list. During the repair, the saved key was synchronized and Open WebUI restarted. The persisted connection then returned HTTP 200 with `coder`. A SQLite backup before this repair is at `~/.local/share/ai-station/open-webui/webui-before-api-key-sync-20261028.db` (mode 0600).

## Updates

1. Resolve a new image digest and update `config/images.lock.yaml` and the matching Quadlet.
2. Pull the exact digest.
3. Back up `~/.local/share/ai-station/open-webui/` before Open WebUI upgrades.
4. Restart one service at a time and run a chat completion smoke test.
5. Commit the lock update and measured result.
