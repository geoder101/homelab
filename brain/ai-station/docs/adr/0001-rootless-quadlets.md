# ADR 0001: Use rootless Podman Quadlets

## Status

Accepted

## Context

`brain` runs Fedora Sway Atomic. The AI station must survive reboots, remain inspectable, and avoid a separate orchestration daemon.

## Decision

Run containers as `geoder101` rootless Podman services generated from Quadlets in `~/.config/containers/systemd/`. Enable user lingering so the systemd user manager survives logout and boot.

## Consequences

- Quadlets are declarative and fit the Fedora/systemd ecosystem.
- Application data belongs in the user home directory.
- Host-level actions such as firewall rules and lingering remain explicit privileged operations.
- The deployment does not use Podman Compose.
