# Operations

Run host-management commands from a host terminal. From toolbox, prefix them with `flatpak-spawn --host`.

## Bootstrap

1. Generate secret files with `scripts/create-secrets`.
2. Enable lingering once with `sudo loginctl enable-linger geoder101`.
3. Pull locked images with `scripts/pull-images`.
4. Run `scripts/download-coder-candidate` to download the pinned Qwen3.6 model into `~/.local/share/ai-station/models/qwen3.6-27b-nvfp4`.
5. Run `scripts/validate-quadlets`.
6. Copy Quadlets with `scripts/install-quadlets`.
7. Reload and start `vllm-coder`, then `litellm`, then `open-webui`.

All three Phase 1 services have `[Install]` sections and start with the user systemd session after `daemon-reload`. Finish the model download before installing/reloading the Quadlets. To prevent automatic startup, remove the unit's `WantedBy` entry and reload; generated Quadlet services cannot be disabled with `systemctl disable`.

## Upgrade to Qwen3.6-27B-NVFP4

Run from the repository on the host:

```bash
scripts/pull-images
scripts/download-coder-candidate
scripts/validate-quadlets
scripts/install-quadlets
systemctl --user daemon-reload
systemctl --user restart vllm-coder litellm
journalctl --user -u vllm-coder -f
# After vLLM reports that startup is complete:
scripts/smoke-test
scripts/smoke-test http://192.168.1.11:4000
```

Downloads resume when rerun. The old `models/coder` directory is retained. To roll back, stop `vllm-coder`, restore the old Quadlet's model path (`/models/coder`) and arguments, reinstall/reload, and restart the service. The API model alias remains `coder`.

The model card recommends vLLM >=0.25.0, FlashInfer >=0.6.13 and CUTLASS DSL >=4.5.2. See [the checkpoint instructions](https://huggingface.co/unsloth/Qwen3.6-27B-NVFP4). The configured context is 128K (131,072 tokens), with GPU memory utilization 0.94 and BF16 KV cache. A 121,996-token prompt passed retrieval checks through LiteLLM without CPU offloading. Measure GPU memory before increasing it further.

Keep `--max-num-seqs 8` on the 32 GiB RTX 5090. The vLLM 0.32.0 default of 512 sequences failed during CUDA graph profiling with a 12.25 GiB allocation request. With 8 sequences, startup and inference succeeded. This is a scheduling cap, not a guarantee that eight full contexts fit concurrently; at 128K the measured KV cache supports approximately 1.30 full-length requests.

### OpenCode context limit

On every OpenCode client using `brain/coder`, set `provider.brain.models.coder.limit.context` to `131072` in its `opencode.jsonc`, then quit and restart OpenCode. The local global config at `~/.config/opencode/opencode.jsonc` was updated during the context increase. The output cap remains 8192 tokens; input, reasoning and output must fit within the total context budget.

Restart LiteLLM after changing the backend context limit so its cached `/v1/models` metadata is refreshed. During the 128K change, restarting only vLLM left LiteLLM advertising 32768 even though long requests succeeded; restarting LiteLLM corrected this to 131072.

The 2026-10-28 image upgrade selects stable vLLM 0.32.0, LiteLLM 1.106.5 and Open WebUI 0.12.4. The lock file and Quadlets pin the corresponding linux/amd64 manifest digests. Before starting the upgraded Open WebUI, stop it and back up its data directory (including SQLite WAL files); the new release can migrate the database.

## API use

LAN:

```text
http://brain:4000/v1
```

Tailscale:

```text
http://brain.tail1fe190.ts.net:4000/v1
```

Use model `coder` and the `LITELLM_MASTER_KEY` as the bearer token. Open WebUI is available on port 3000 at the same hostname.

## Systemd commands

```bash
systemctl --user daemon-reload
systemctl --user start vllm-coder
systemctl --user start litellm
systemctl --user start open-webui
systemctl --user status vllm-coder litellm open-webui
journalctl --user -u vllm-coder -f
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
systemctl --user status litellm vllm-coder
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
