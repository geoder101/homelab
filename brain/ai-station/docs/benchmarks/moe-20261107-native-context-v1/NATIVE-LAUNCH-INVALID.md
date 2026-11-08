# Archived host-launch calibration

The host systemd → toolbox launch inherited PWD=/var/home/geoder101 despite
the child process cwd being the repository. The fresh Qwen control failed
relative-path native probes, so native results in this directory are invalid
for model comparisons. Valid direct-API observations are retained as diagnostics,
but are not pooled into the restarted primary comparison.

The new launcher supplies both OpenCode --dir and an aligned PWD, validates
fixture visibility, and records the working directory. The primary batch is
restarted in ../moe-20261107 after control calibration. No grader/prompt was relaxed.
