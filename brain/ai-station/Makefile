# Host operations work from either Fedora or its toolbox. Override HOST= when
# deliberately supplying a different execution wrapper.
ROOT := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))
HOST ?= $(if $(wildcard /run/.toolboxenv),flatpak-spawn --host,)
BASE_URL ?= http://brain.local:4000
WEBUI_URL ?= http://127.0.0.1:3000
API_URL := $(patsubst %/,%,$(BASE_URL))/v1
.DEFAULT_GOAL := help
.NOTPARALLEL:

.PHONY: help secrets pull download download-template preflight validate install reload \
        start restart status logs smoke verify verify-stress verify-budgets \
        verify-history verify-concurrency audit validate-home client-check client-smoke client-concurrency backup-webui deploy bootstrap \
        profile-qwen38-q3 profile-qwen38-q4 profile-ornith-q4 profile-ornith-q5 profile-status \
        backend-status \
        swap-active-cpu swap-active-gpu \
        download-qwen38-q3 download-qwen38-q4 download-ornith-q4 download-ornith-q5 benchmark-status

help:
	@printf '%s\n' \
	  'AI Station (Fedora host operations; toolbox wrapper is automatic)' \
	  '  make bootstrap         Create initial secrets, then deploy' \
	  '  make deploy            Back up UI, pull/download, install/reload, restart, smoke' \
	  '  make pull / download   Fetch pinned images / model + template' \
	  '  make download-template Fetch only the pinned, checksum-verified template' \
	  '  make profile-qwen38-q3 Select Qwen3.8-27B UD-Q3_K_XL, b11960, three 256K agents' \
	  '  make profile-qwen38-q4 Select Qwen3.8-27B UD-Q4_K_XL, b11960, two 256K agents' \
	  '  make profile-ornith-q4 Experimental Ornith 1.5-35B-A3B Q4_K_M, b11960, two 256K agents' \
	  '  make profile-ornith-q5 Select Ornith 1.5-35B-A3B Q5_K_M, b12182, two 256K agents' \
	  '  make profile-status    Show selected and running profile' \
	  '  make backend-status    Show which backend is serving (GPU profile / CPU fallback)' \
	  '  make swap-active-cpu   Hand inference to the CPU fallback (free the GPU for another task)' \
	  '  make swap-active-gpu   Restore inference to the GPU backend' \
	  '  make download-qwen38-q3 Prepare Q3 weights/projector/template for offline switching' \
	  '  make download-qwen38-q4 Prepare Q4 weights/projector/template for offline switching' \
	  '  make download-ornith-q4 Prepare pinned Ornith Q4_K_M weights/projector/template' \
	  '  make download-ornith-q5 Prepare pinned Ornith Q5_K_M weights/projector/template' \
	  '  make benchmark-status Refresh the live model/configuration scorecard (JSON)' \
	  '  make preflight         Verify artifact hashes, image presence, secret files' \
	  '  make validate          Run Fedora Quadlet generator dry-run' \
	  '  make install / reload  Checked Quadlet installation / systemd reload' \
	  '  make start / restart   Backend readiness, gateway readiness, then WebUI' \
	  '  make status / logs     Inspect services / follow backend journal' \
	  '  make smoke / verify    Quick inference / full alias and streaming checks' \
	  '  make verify-stress     Also run bounded coding-plan probes' \
	  '  make verify-budgets    Exercise low/medium thinking ceilings' \
	  '  make verify-history   Tool-error recovery and local image check' \
	  '  make verify-concurrency  Concurrent streaming through LiteLLM' \
	  '  make audit             Check installed LiteLLM request serialization' \
	  '  make backup-webui      Stop UI, archive persistent data, resume if active' \
	  '  make client-check      Check this environment’s native Brain OpenCode config' \
	  '  make client-smoke      Run a small local OpenCode inference request' \
	  '  make client-concurrency  Run three native OpenCode read-only sessions' \
	  '  make validate-home     Validate offline Downloads copies (local Python)' \
	  'BASE_URL defaults to http://brain.local:4000 (exclude /v1).'

secrets:
	$(HOST) "$(ROOT)/scripts/create-secrets"
pull:
	$(HOST) "$(ROOT)/scripts/pull-images"
download:
	$(HOST) "$(ROOT)/scripts/download-coder-candidate"
download-template:
	$(HOST) "$(ROOT)/scripts/download-chat-template"
download-qwen38-q3:
	$(HOST) "$(ROOT)/scripts/download-qwen38-q3-model"
	$(HOST) "$(ROOT)/scripts/download-qwen38-candidates" vision
	$(HOST) "$(ROOT)/scripts/download-chat-template" qwen_fixed
download-qwen38-q4:
	$(HOST) "$(ROOT)/scripts/download-qwen38-candidates" gguf
	$(HOST) "$(ROOT)/scripts/download-chat-template" qwen_fixed
download-ornith-q4:
	$(HOST) bash "$(ROOT)/scripts/download-ornith-candidate"
download-ornith-q5:
	$(HOST) bash "$(ROOT)/scripts/download-ornith-candidate" q5
profile-qwen38-q3:
	$(HOST) python3 "$(ROOT)/scripts/switch-coder-profile" capacity --base-url "$(BASE_URL)"
profile-qwen38-q4:
	$(HOST) python3 "$(ROOT)/scripts/switch-coder-profile" precision --base-url "$(BASE_URL)"
profile-ornith-q4:
	$(HOST) python3 "$(ROOT)/scripts/switch-coder-profile" ornith --base-url "$(BASE_URL)"
profile-ornith-q5:
	$(HOST) python3 "$(ROOT)/scripts/switch-coder-profile" ornith-q5 --base-url "$(BASE_URL)"
profile-status:
	$(HOST) python3 "$(ROOT)/scripts/switch-coder-profile" --status
backend-status:
	$(HOST) "$(ROOT)/scripts/backend-status"
swap-active-cpu:
	$(HOST) "$(ROOT)/scripts/swap-active-backend" cpu
swap-active-gpu:
	$(HOST) "$(ROOT)/scripts/swap-active-backend" gpu
benchmark-status:
	$(HOST) python3 "$(ROOT)/scripts/summarize-moe-benchmarks"
preflight:
	$(HOST) python3 "$(ROOT)/scripts/preflight"
validate:
	$(HOST) "$(ROOT)/scripts/validate-quadlets"
install:
	$(HOST) "$(ROOT)/scripts/install-quadlets"
reload:
	$(HOST) systemctl --user daemon-reload
start:
	$(HOST) "$(ROOT)/scripts/start-stack" start "$(BASE_URL)" "$(WEBUI_URL)"
restart:
	$(HOST) "$(ROOT)/scripts/start-stack" restart "$(BASE_URL)" "$(WEBUI_URL)"
status:
	$(HOST) systemctl --user --no-pager status llama-coder litellm open-webui
logs:
	$(HOST) journalctl --user -u llama-coder -f
smoke:
	$(HOST) "$(ROOT)/scripts/smoke-test" "$(BASE_URL)"
verify:
	$(HOST) python3 "$(ROOT)/scripts/verify-reasoning-profiles" --base-url "$(API_URL)"
verify-stress:
	$(HOST) python3 "$(ROOT)/scripts/verify-reasoning-profiles" --base-url "$(API_URL)" --stress
verify-budgets:
	$(HOST) python3 "$(ROOT)/scripts/verify-reasoning-profiles" --base-url "$(API_URL)" --budget-check
verify-history:
	$(HOST) python3 "$(ROOT)/scripts/verify-reasoning-profiles" --base-url "$(API_URL)" --history-check
verify-concurrency:
	$(HOST) python3 "$(ROOT)/scripts/verify-concurrency"
audit:
	$(HOST) podman exec -i ai-station-litellm python < "$(ROOT)/scripts/audit-reasoning-forwarding"
validate-home:
	python3 "$(ROOT)/scripts/validate-home-configs"
client-check:
	python3 "$(ROOT)/scripts/check-brain-client"
client-smoke:
	opencode run --model brain/coder-off --thinking --format json 'Reply with exactly READY. Do not use tools.'
client-concurrency:
	python3 "$(ROOT)/scripts/check-native-concurrency"
backup-webui:
	$(HOST) "$(ROOT)/scripts/backup-webui"

# .NOTPARALLEL deliberately preserves this ordering, including under make -j.
# Installation and reload can activate WantedBy units: downloads must finish first.
deploy: backup-webui pull download install reload restart smoke
bootstrap: secrets deploy
