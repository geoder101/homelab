# AI Station

Declarative, rootless Podman deployment for the local AI workstation `brain`.

## Current deployment

- llama.cpp provides the active OpenAI-compatible inference backend; vLLM is an optional NVFP4 profile.
- LiteLLM is the single API gateway for OpenCode and other API clients.
- Open WebUI is the browser interface and talks only to LiteLLM.
- Podman Quadlets run as `geoder101` user systemd services.

The selected coding model is **Qwen3.8-27B UD-Q4_K_XL, precision profile**, pinned in `config/models.lock.yaml`. It serves **two concurrent agents with 256K context each**, GPU Q4 K/V cache and GPU vision on the RTX 5090. LiteLLM exposes `coder-off`, `coder-low`, `coder-medium`, and `coder-xhigh`; `coder` remains medium-compatible. The [2026-11-05 Ornith Q4_K_M trial](docs/benchmarks/ornith-20261105.md) found much faster inference and matching small-suite coding results, but native OpenCode instruction/path regressions blocked promotion. Qwen was restored and verified. See [model settings](docs/model-profile.md) for current limits and controls.

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

On a deployed Brain, `make profile-ornith` selects the experimental Ornith candidate with two 256K agents.
`make profile-capacity` selects Qwen Q3 with three 256K agents and
`make profile-precision` selects Qwen Q4 with two 256K agents.
`make profile-status` shows the current selection. Switch globally between tasks;
the existing model aliases work in every profile.

Start with `make help`. `make bootstrap` handles initial secrets and deployment;
`make deploy` backs up WebUI, fetches/checks pinned artifacts, installs/reloads
Quadlets and restarts the stack in readiness order. Host commands work from
Fedora or toolbox automatically. See [automation and managed state](docs/automation.md)
for all targets, backup policy, and Brain's simple native OpenCode configuration.

Read `docs/operations.md`. In particular, do not copy secrets into this repository and do not start the inference unit until `scripts/download-coder-candidate` has finished downloading the pinned model and chat template.
