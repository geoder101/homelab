# AI Station

Declarative, rootless Podman deployment for the local AI workstation `brain`.

## Phase 1

- llama.cpp provides the active OpenAI-compatible inference backend; vLLM is an optional NVFP4 profile.
- LiteLLM is the single API gateway for OpenCode and other API clients.
- Open WebUI is the browser interface and talks only to LiteLLM.
- Podman Quadlets run as `geoder101` user systemd services.

The coding model is `unsloth/Qwen3.8-27B-GGUF`, Dynamic V3 `UD-Q4_K_XL`, pinned in `config/models.lock.yaml`. The default profile uses 256K context, Q8 K/V cache and medium reasoning on the RTX 5090. Clients always request the stable alias `coder`. See `docs/model-profile.md` for the measured NVFP4/GGUF tradeoff.

## Architecture

```text
OpenCode / LAN client / Tailscale client
                 |
       http://brain:4000/v1
                 |
             LiteLLM
                 |
      ai-station Podman network
          |                 |
   llama.cpp :8000    Open WebUI :8080
          |                 |
       RTX 5090       http://brain:3000
```

Only LiteLLM and Open WebUI publish host ports. Inference backends are reachable only through the private Podman network.

## Repository layout

- `quadlets/`: declarative user-systemd container definitions.
- `config/`: non-secret LiteLLM configuration and image lock file.
- `docs/`: architecture, operating guide, ADRs, and benchmark procedure.
- `scripts/`: host bootstrap, validation, and benchmark helpers.

## Before deployment

Read `docs/operations.md`. In particular, do not copy secrets into this repository and do not start the inference unit until `scripts/download-coder-candidate` has finished downloading the pinned model.
