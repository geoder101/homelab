# Operations

Run host-management commands from a host terminal. From toolbox, prefix them with `flatpak-spawn --host`.

## Bootstrap

1. Generate secret files with `scripts/create-secrets`.
2. Copy Quadlets with `scripts/install-quadlets`.
3. Enable lingering once with `sudo loginctl enable-linger geoder101`.
4. Pull locked images with the commands in `config/images.lock.yaml`.
5. Download the benchmark candidate into `~/.local/share/ai-station/models/coder`.
6. Run `scripts/validate-quadlets`.
7. Reload and start `vllm-coder`, then `litellm`, then `open-webui`.

All three Phase 1 services have `[Install]` sections and start with the user systemd session after `daemon-reload`. The vLLM unit is enabled only because the benchmarked model now exists at `/models/coder`; do not remove that directory without first stopping and disabling `vllm-coder`.

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
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="3000" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="4000" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Port 4000 is an API endpoint; port 3000 is browser-only Open WebUI. Tailscale uses its own encrypted interface and was verified locally at `100.110.211.92`; test it from another tailnet device after the firewall change.

## Updates

1. Resolve a new image digest and update `config/images.lock.yaml` and the matching Quadlet.
2. Pull the exact digest.
3. Back up `~/.local/share/ai-station/open-webui/` before Open WebUI upgrades.
4. Restart one service at a time and run a chat completion smoke test.
5. Commit the lock update and measured result.
