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
