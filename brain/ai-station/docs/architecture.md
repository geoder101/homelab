# Architecture

`brain` hosts a rootless Podman deployment under user `geoder101`. Systemd user units are generated from Quadlets. This avoids a daemon, matches Fedora Atomic, and makes the desired state reviewable in Git.

## Network boundary

`ai-station` is a Podman bridge network. Neither inference backend publishes a host port. LiteLLM publishes TCP 4000 and Open WebUI publishes TCP 3000. Its default upstream is `http://llama-coder:8000/v1`.

Rootless Podman uses pasta to connect this bridge to the host. LiteLLM binds to `0.0.0.0` inside the container and publishes `0.0.0.0:4000:4000` on the host. The bridge does not prevent LAN access to published ports.

All Quadlets set `[Quadlet] DefaultDependencies=false`: images are pre-pulled (`Pull=never`) and model artifacts are local. This removes the implicit wait for `podman-user-wait-network-online.service`, which times out when the host's `network-online.target` is inactive. Explicit dependencies on the private Podman network remain in effect; normal systemd dependencies are retained.

The deployment intentionally uses HTTP because the LAN is trusted and Tailscale encrypts traffic between tailnet nodes. The LiteLLM bearer key remains required for its API. TLS can be added later at the gateway boundary without changing backend services.

LAN clients need a stable router DNS or DHCP reservation for `brain`. Tailscale clients use `brain.tail1fe190.ts.net` through MagicDNS. Do not rely on link-local IPv6 addresses as a client endpoint.

## Storage

```text
~/.local/share/ai-station/
  models/qwen3.8-27b-gguf/   active Qwen3.8 Dynamic Q4_K_XL + vision projector
  models/qwen3.8-27b-nvfp4/  optional vLLM profile
  models/qwen3.6-27b-nvfp4/  previous checkpoint, retained for rollback
  models/coder/             original AWQ checkpoint
  huggingface/         Hugging Face cache
  open-webui/          Open WebUI SQLite data, uploads, and settings

~/.config/ai-station/
  litellm.env          LiteLLM master key, mode 0600
  open-webui.env       UI session secret and LiteLLM key, mode 0600
```

Bind mounts use `:Z` so Podman applies a private SELinux label to persisted application data. GPU containers additionally require `SecurityLabelDisable=true`; see ADR 0002.

## Model lifecycle

One GPU backend serves one base model at a time, exposed externally as `coder`. This stable alias decouples OpenCode and Open WebUI from model repository changes. Only `llama-coder` has an automatic-start `[Install]` section; the optional `vllm-coder` must be started explicitly after stopping llama.cpp.

The selected llama.cpp profile uses Unsloth Dynamic V3 `UD-Q4_K_XL`, a 262,144-token context, Q8 K/V cache, Flash Attention and one request slot. Transformer layers, active K/V cache and the vision projector are GPU-offloaded. Medium reasoning is the default; historical reasoning preservation and MTP are disabled to prioritize a useful context and bounded turn latency. Standard prompt-cache reuse remains enabled. See `docs/model-profile.md`.

The alternative Qwen3.8 NVFP4 profile uses vLLM 0.32.0 with FlashInfer 0.7.0.post1 and CUTLASS DSL 4.8.0. It has 163,840-token context, explicit calibrated FP8 attention KV, utilization 0.94 and one concurrent sequence. Even with one sequence, 256K required 8.33 GiB cache versus 5.64 GiB available. Both Unsloth NVFP4 checkpoints already select FP8 KV with `auto`; earlier descriptions of those runs as BF16 were incorrect.

Additional profiles are possible, but the 32 GiB RTX 5090 runs one large model at a time.
