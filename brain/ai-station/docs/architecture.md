# Architecture

`brain` hosts a rootless Podman deployment under user `geoder101`. Systemd user units are generated from Quadlets. This avoids a daemon, matches Fedora Atomic, and makes the desired state reviewable in Git.

## Network boundary

`ai-station` is a Podman bridge network. vLLM has no published host port. LiteLLM publishes TCP 4000 and Open WebUI publishes TCP 3000.

Rootless Podman uses pasta to connect this bridge to the host. LiteLLM binds to `0.0.0.0` inside the container and publishes `0.0.0.0:4000:4000` on the host. The bridge does not prevent LAN access to published ports.

All Quadlets set `[Quadlet] DefaultDependencies=false`: images are pre-pulled (`Pull=never`) and model artifacts are local. This removes the implicit wait for `podman-user-wait-network-online.service`, which times out when the host's `network-online.target` is inactive. Explicit dependencies on the private Podman network remain in effect; normal systemd dependencies are retained.

The deployment intentionally uses HTTP because the LAN is trusted and Tailscale encrypts traffic between tailnet nodes. The LiteLLM bearer key remains required for its API. TLS can be added later at the gateway boundary without changing backend services.

LAN clients need a stable router DNS or DHCP reservation for `brain`. Tailscale clients use `brain.tail1fe190.ts.net` through MagicDNS. Do not rely on link-local IPv6 addresses as a client endpoint.

## Storage

```text
~/.local/share/ai-station/
  models/qwen3.6-27b-nvfp4/  active Qwen3.6 checkpoint
  models/coder/             previous checkpoint, retained for rollback
  huggingface/         Hugging Face cache
  open-webui/          Open WebUI SQLite data, uploads, and settings

~/.config/ai-station/
  litellm.env          LiteLLM master key, mode 0600
  open-webui.env       UI session secret and LiteLLM key, mode 0600
```

Bind mounts use `:Z` so Podman applies a private SELinux label to persisted application data. The vLLM GPU container additionally requires `SecurityLabelDisable=true`; see ADR 0002.

## Model lifecycle

One vLLM process serves one base model. The selected model is materialized at `/models/qwen3.6-27b-nvfp4` and exposed externally as `coder`. This stable alias decouples OpenCode and Open WebUI from model repository changes.

The Unsloth checkpoint uses mixed NVFP4/FP8 compressed tensors. The pinned vLLM 0.32.0 image includes FlashInfer 0.7.0.post1 and CUTLASS DSL 4.8.0. Quantization and kernel selection are automatic. The context limit is 131,072 tokens on the RTX 5090, with the `qwen3_xml` tool parser and `qwen3` reasoning parser. GPU memory utilization is 0.94, providing enough BF16 KV cache for a full 128K request without CPU offloading. Concurrent sequences are capped at 8: the default 512 causes CUDA graph profiling to exceed 32 GiB VRAM for this hybrid model. The measured cache supports approximately 1.30 full-length 128K requests, not eight simultaneous full-length requests.

The system supports future additional vLLM Quadlets, but a 32 GiB RTX 5090 should initially run one large model at a time. A reasoning service is a later, disabled-by-default profile, not a permanently competing GPU process.
