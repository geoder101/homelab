# AI Station

Declarative, rootless Podman deployment for the local AI workstation `brain`.

## Phase 1

- vLLM provides one active OpenAI-compatible inference backend.
- LiteLLM is the single API gateway for OpenCode and other API clients.
- Open WebUI is the browser interface and talks only to LiteLLM.
- Podman Quadlets run as `geoder101` user systemd services.

The coding model is `unsloth/Qwen3.6-27B-NVFP4`, pinned in `config/models.lock.yaml`. The service loads `/models/qwen3.6-27b-nvfp4`; clients always request the stable alias `coder`.

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
      vLLM :8000      Open WebUI :8080
          |                 |
       RTX 5090       http://brain:3000
```

Only LiteLLM and Open WebUI publish host ports. vLLM is reachable only through the private Podman network.

## Repository layout

- `quadlets/`: declarative user-systemd container definitions.
- `config/`: non-secret LiteLLM configuration and image lock file.
- `docs/`: architecture, operating guide, ADRs, and benchmark procedure.
- `scripts/`: host bootstrap, validation, and benchmark helpers.

## Before deployment

Read `docs/operations.md`. In particular, do not copy secrets into this repository and do not start the vLLM unit until `scripts/download-coder-candidate` has finished downloading the pinned model.
