# ADR 0002: Use NVIDIA CDI with SELinux labels disabled only for vLLM

## Status

Accepted

## Context

NVIDIA CDI is installed and lists `nvidia.com/gpu=all`. A rootless CUDA smoke test fails with `Failed to initialize NVML: Insufficient Permissions` unless SELinux labeling is disabled. NVIDIA documents `--security-opt label=disable` for Podman CDI workloads.

## Decision

The vLLM Quadlet uses `AddDevice=nvidia.com/gpu=all` and `SecurityLabelDisable=true`. It does not use `Privileged=true`, host networking, or host filesystem mounts beyond its specific model and cache directories.

## Consequences

- GPU workloads function on this SELinux-enforcing Fedora host.
- vLLM has a weaker SELinux process boundary than the other containers.
- The exception is narrow, documented, and must not be copied to LiteLLM or Open WebUI without a demonstrated need.

## Update (2026-10-31)

The llama.cpp backend (`llama-coder.container`) now uses the same `AddDevice=nvidia.com/gpu=all` + `SecurityLabelDisable=true` pair. The decision and its consequences therefore apply to **both** GPU backends, not vLLM alone; the weaker SELinux process boundary extends to `ai-station-llama-coder` as well. The decision text above is preserved as originally written.
