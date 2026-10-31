# Documentation guide

| I want to... | Read |
| --- | --- |
| Understand the system, network, storage layout | [architecture.md](architecture.md) |
| Install or update the stack step by step | [operations.md](operations.md) |
| Use `make` targets, managed state, client config | [automation.md](automation.md) |
| See the current profile settings at a glance | [model-profile.md](model-profile.md) |
| Choose between capacity and precision profiles | [concurrency.md](concurrency.md) |
| Understand the `coder-*` reasoning aliases | [reasoning-profiles.md](reasoning-profiles.md) |
| Reproduce the coding-agent quality benchmarks | [benchmarking.md](benchmarking.md), [benchmarks/README.md](benchmarks/README.md) |
| Read historical NVFP4 vs GGUF results | [benchmark-results.md](benchmark-results.md) |
| Understand why Quadlets / SELinux are configured this way | [adr/](adr/) |

Conventions:

- `config/*.lock.yaml` files are records of pinned artifacts, not input to
  downloaders except `images.lock.yaml` (read by `scripts/pull-images`) and
  `models.lock.yaml` (read by `scripts/download-coder-candidate`).
- Measurements older than the current profile are labelled *historical* in
  place; [concurrency.md](concurrency.md) is the current source of truth for
  capacity selection.
- Secrets are never documented here; only variable names appear in
  `config/*.env.example`.
