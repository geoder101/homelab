# Architecture

`brain` hosts a rootless Podman deployment under user `geoder101`. Systemd user units are generated from Quadlets. This avoids a daemon, matches Fedora Atomic, and makes the desired state reviewable in Git.

## Network boundary

`ai-station` is a Podman bridge network. vLLM has no published host port. LiteLLM publishes TCP 4000 and Open WebUI publishes TCP 3000.

The deployment intentionally uses HTTP because the LAN is trusted and Tailscale encrypts traffic between tailnet nodes. The LiteLLM bearer key remains required for its API. TLS can be added later at the gateway boundary without changing backend services.

LAN clients need a stable router DNS or DHCP reservation for `brain`. Tailscale clients use `brain.tail1fe190.ts.net` through MagicDNS. Do not rely on link-local IPv6 addresses as a client endpoint.

## Storage

```text
~/.local/share/ai-station/
  models/coder/        selected benchmark winner
  huggingface/         Hugging Face cache
  open-webui/          Open WebUI SQLite data, uploads, and settings

~/.config/ai-station/
  litellm.env          LiteLLM master key, mode 0600
  open-webui.env       UI session secret and LiteLLM key, mode 0600
```

Bind mounts use `:Z` so Podman applies a private SELinux label to persisted application data. The vLLM GPU container additionally requires `SecurityLabelDisable=true`; see ADR 0002.

## Model lifecycle

One vLLM process serves one base model. The selected model is materialized at `/models/coder` and exposed externally as `coder`. This stable alias decouples OpenCode and Open WebUI from model repository changes.

The system supports future additional vLLM Quadlets, but a 32 GiB RTX 5090 should initially run one large model at a time. A reasoning service is a later, disabled-by-default profile, not a permanently competing GPU process.
