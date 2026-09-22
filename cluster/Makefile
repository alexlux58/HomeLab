# ===========================================================================
#  Proxmox VE 8 -> 9 three-node cluster migration
#
#  Every target below runs exactly ONE stage. There is deliberately no
#  "deploy-all" / "migrate-all" target: each destructive stage requires an
#  explicit boolean plus an exact confirmation string supplied on the command
#  line, and each stage is meant to be reviewed by a human before the next one
#  is started.
#
#  Use EXTRA_JSON for approval gates so multi-word confirmation strings remain
#  one Ansible value, e.g.
#      make join-222 EXTRA_JSON='{"allow_join_pve222":true,"join_confirmation_pve222":"JOIN pve3 192.168.0.13 TO homelab"}'
#  EXTRA remains available for simple single-word Ansible arguments.
# ===========================================================================

SHELL          := /bin/bash
.SHELLFLAGS    := -eu -o pipefail -c
.DEFAULT_GOAL  := help

VENV           ?= .venv
PY             ?= $(VENV)/bin/python
PIP            ?= $(VENV)/bin/pip
ANSIBLE        ?= $(VENV)/bin/ansible
ANSIBLE_PB     ?= $(VENV)/bin/ansible-playbook
INVENTORY      ?= inventory/hosts.yml
ARTIFACTS      ?= artifacts
EXTRA          ?=
EXTRA_JSON     ?=
HOST           ?=
SERVICE        ?=
SERVICES_INVENTORY ?= inventory/homelab-services.yml
NETKNIFE_SOURCE ?= /Users/labadmin/Desktop/PROJECTS/NetKnife-App
TERRAFORM      ?= terraform
TF_DIR         ?= terraform/homelab-guests
NETBOX_INVENTORY ?= inventory/netbox.yml
NETBOX_TF_DIR  ?= terraform/netbox-guest

# Long-running stages (backups, restores, upgrades) should survive a laptop
# sleeping. Wrap them in tmux/screen on the controller if you have any doubt.
PB             = $(ANSIBLE_PB) -i $(INVENTORY)
EXTRA_JSON_ARG = $(if $(strip $(EXTRA_JSON)),-e '$(EXTRA_JSON)',)

.PHONY: help setup lint test validate-ssh discover nas-preflight capture-config \
        backup-initial backup-final verify-backups prepare-222 test-restore \
        precheck-upgrade-146 upgrade-146 create-cluster join-222 \
        precheck-destroy-180 postinstall-180 join-180 build-restore-plan \
        restore-180 start-restored-guests verify-backups-initial validate clean-artifacts tree \
        homelab-preflight homelab-base homelab-service homelab-backups \
        netbox-preflight netbox-base netbox-deploy netbox-backups \
        netbox-terraform-fmt netbox-terraform-init netbox-terraform-validate \
        netbox-terraform-plan \
        terraform-fmt terraform-init terraform-validate terraform-plan

help: ## Show this help
	@echo ""
	@echo "Proxmox cluster migration — run one stage at a time, in order."
	@echo ""
	@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "} {printf "  \033[36m%-24s\033[0m %s\n", $$1, $$2}'
	@echo ""
	@echo "Destructive stages additionally require -e flags; see docs/migration-runbook.md"
	@echo ""

# ---------------------------------------------------------------------------
# Local tooling (no network access to the Proxmox hosts)
# ---------------------------------------------------------------------------
setup: ## Create the local virtualenv and install controller dependencies
	python3 -m venv $(VENV)
	$(PIP) install --upgrade pip
	$(PIP) install -r requirements.txt
	$(VENV)/bin/ansible-galaxy collection install -r requirements.yml
	@echo "OK: controller tooling installed into $(VENV)"

lint: ## Run yamllint + ansible-lint + ruff + shellcheck (offline)
	$(VENV)/bin/yamllint -s .
	ANSIBLE_CONFIG=ansible.cfg $(VENV)/bin/ansible-lint
	$(VENV)/bin/ruff check scripts tests
	$(VENV)/bin/ruff format --check scripts tests
	@if command -v shellcheck >/dev/null 2>&1; then \
		find roles scripts -name '*.sh' -print0 | xargs -0 -r shellcheck -S style; \
	else \
		echo "WARN: shellcheck not installed locally (CI enforces it)"; \
	fi

test: ## Run the offline unit test-suite
	$(VENV)/bin/pytest -q

tree: ## Print the repository tree
	@find . -path ./$(VENV) -prune -o -path ./.git -prune -o -print | sort

clean-artifacts: ## Remove generated reports (NEVER touches backups on the NAS)
	@echo "This removes local reports under $(ARTIFACTS)/ only. Backups are never touched."
	find $(ARTIFACTS) -mindepth 1 ! -name '.gitkeep' -delete

# ---------------------------------------------------------------------------
# Stage 0 — read-only
# ---------------------------------------------------------------------------
validate-ssh: ## Stage 0a: prove existing SSH key auth works on all three hosts
	$(PB) playbooks/00_validate_ssh.yml $(EXTRA_JSON_ARG) $(EXTRA)

discover: ## Stage 0b: read-only discovery of all three hosts
	$(PB) playbooks/01_discover.yml $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Stage 1-2 — NAS preflight and host configuration capture
# ---------------------------------------------------------------------------
nas-preflight: ## Stage 1: validate the Synology NFS target from all three hosts
	$(PB) playbooks/05_synology_preflight.yml $(EXTRA_JSON_ARG) $(EXTRA)

capture-config: ## Stage 2: capture sanitized host configuration from .11 and .12
	$(PB) playbooks/10_capture_host_config.yml $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Stage 3-4 — backups
# ---------------------------------------------------------------------------
backup-initial: ## Stage 3: snapshot-mode backup   (usage: make backup-initial HOST=pve1)
	@test "$(HOST)" = "pve1" || { echo "ERROR: only HOST=pve1 has protected guests"; exit 2; }
	$(PB) playbooks/20_backup_initial.yml --limit '$(HOST)' $(EXTRA_JSON_ARG) $(EXTRA)

backup-final: ## Stage 4: stopped-guest backup     (usage: make backup-final HOST=pve1)
	@test "$(HOST)" = "pve1" || { echo "ERROR: only HOST=pve1 has protected guests"; exit 2; }
	$(PB) playbooks/21_backup_final_stopped.yml --limit '$(HOST)' --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

verify-backups-initial: ## Stage 3b: verify the initial generation before final shutdown
	$(PB) playbooks/25_verify_backups.yml -e '{"backup_verify_generations":["initial"]}' $(EXTRA_JSON_ARG) $(EXTRA)

verify-backups: ## Stage 4b: checksum + vma/tar verify every archive
	$(PB) playbooks/25_verify_backups.yml $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Stage 5 — .13 clean install; the old test-restore target is retained as legacy tooling
# ---------------------------------------------------------------------------
prepare-222: ## Stage 5a: emit the .13 clean-install checklist / validate after reinstall
	$(PB) playbooks/30_prepare_pve222.yml $(EXTRA_JSON_ARG) $(EXTRA)

test-restore: ## Legacy plan only (not used by the current rebuild-later migration)
	$(PB) playbooks/31_test_restore.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Stage 6 — align versions on the seed node
# ---------------------------------------------------------------------------
precheck-upgrade-146: ## Stage 6a: pve8to9 --full + repository change report for .11
	$(PB) playbooks/40_precheck_pve146_upgrade.yml $(EXTRA_JSON_ARG) $(EXTRA)

upgrade-146: ## Stage 6b: gated PVE 8 -> 9 upgrade of .11 (requires console access)
	$(PB) playbooks/41_upgrade_pve146.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Stage 7-9 — cluster
# ---------------------------------------------------------------------------
create-cluster: ## Stage 7: create the cluster on .11 only
	$(PB) playbooks/50_create_cluster.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

join-222: ## Stage 8: join .13 (requires allow_join_pve222 + confirmation string)
	$(PB) playbooks/60_join_pve222.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

precheck-destroy-180: ## Stage 9a: verify every gate before .12 may be clean-installed
	$(PB) playbooks/70_precheck_destroy_pve180.yml $(EXTRA_JSON_ARG) $(EXTRA)

postinstall-180: ## Stage 9b: validate .12 after its manual clean install
	$(PB) playbooks/71_postinstall_pve180.yml $(EXTRA_JSON_ARG) $(EXTRA)

join-180: ## Stage 9c: join .12 to the cluster
	$(PB) playbooks/72_join_pve180.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# Legacy restore tooling plus final validation
# ---------------------------------------------------------------------------
build-restore-plan: ## Legacy plan only: generate the retired .12 restore plan
	$(PY) scripts/build_restore_plan.py \
		--discovery $(ARTIFACTS)/discovery.json \
		--collisions $(ARTIFACTS)/vmid-collisions.json \
		--manifest $(ARTIFACTS)/backup-manifest.json \
		--source-host pve2 --target-node pve2 \
		--disposable-host pve \
		--out-json $(ARTIFACTS)/restore-plan.json \
		--out-markdown $(ARTIFACTS)/restore-plan.md

restore-180: ## Legacy plan only: restore .12 guests (not in current migration)
	$(PB) playbooks/80_restore_pve180.yml --tags all,destructive $(EXTRA_JSON_ARG) $(EXTRA)

start-restored-guests: ## Legacy plan only: start restored .12 guests
	$(PB) playbooks/81_start_restored_guests.yml $(EXTRA_JSON_ARG) $(EXTRA)

# ---------------------------------------------------------------------------
# LAN inventory dashboard (independent of the migration stages)
# ---------------------------------------------------------------------------
lan-scan: ## Read-only LAN sweep -> artifacts/lan-inventory.json
	ssh -o BatchMode=yes pve1 'bash -s' < scripts/lan_scan.sh > $(ARTIFACTS)/lan-inventory.json
	@$(PY) -c "import json;d=json.load(open('$(ARTIFACTS)/lan-inventory.json'));print(f'  {len(d)} devices')"

lan-site: ## Build the static LAN inventory page -> site/index.html
	$(PY) scripts/build_lan_inventory_site.py

lan-preview: ## Serve site/ locally at http://127.0.0.1:8899
	@echo "http://127.0.0.1:8899/index.html  (ctrl-c to stop)"
	@cd site && python3 -m http.server 8899 --bind 127.0.0.1

lan-deploy: ## Copy the page to the Synology (requires Web Station + a Volume 1 docroot)
	@test -f site/index.html || { echo "run 'make lan-site' first"; exit 2; }
	scp site/index.html nas:/volume1/web/lan/index.html
	@echo "deployed -> http://192.168.0.20/lan/"

# ---------------------------------------------------------------------------
# Steady-state Home Lab guest and application automation
# ---------------------------------------------------------------------------
homelab-preflight: ## Read-only validation of VM 300 before reconciliation
	$(ANSIBLE_PB) -i $(SERVICES_INVENTORY) playbooks/100_homelab_services_preflight.yml

homelab-base: ## Reconcile VM 300 Docker, hardening, and filesystem prerequisites
	$(ANSIBLE_PB) -i $(SERVICES_INVENTORY) playbooks/099_configure_homelab_guest.yml

homelab-service: ## Reconcile one app (usage: make homelab-service SERVICE=homepage)
	@case "$(SERVICE)" in \
		homepage|it-tools|linkding|netknife|npm|pairdrop|speedtest-tracker|uptime-kuma) ;; \
		*) echo "ERROR: set SERVICE to one reviewed Home Lab application"; exit 2 ;; \
	esac
	$(ANSIBLE_PB) -i $(SERVICES_INVENTORY) playbooks/101_deploy_homelab_service.yml \
		-e homelab_services_service_name='$(SERVICE)' \
		-e homelab_services_controller_netknife_source='$(NETKNIFE_SOURCE)' \
		$(EXTRA_JSON_ARG) $(EXTRA)

homelab-backups: ## Reconcile VM 300 application-aware backup timer
	$(ANSIBLE_PB) -i $(SERVICES_INVENTORY) playbooks/102_configure_homelab_backups.yml

# ---------------------------------------------------------------------------
# NetBox (planned). Nothing below has been run: VM 330 does not exist yet.
# Each stage is separate on purpose; there is no aggregate NetBox target.
# ---------------------------------------------------------------------------
netbox-preflight: ## Read-only validation of the NetBox guest before reconciliation
	$(ANSIBLE_PB) -i $(NETBOX_INVENTORY) playbooks/110_netbox_preflight.yml

netbox-base: ## Reconcile NetBox guest Docker, hardening, and prerequisites
	$(ANSIBLE_PB) -i $(NETBOX_INVENTORY) playbooks/111_configure_netbox_guest.yml

netbox-deploy: ## Deploy NetBox (requires the documented approval, see docs/netbox-plan.md)
	$(ANSIBLE_PB) -i $(NETBOX_INVENTORY) playbooks/112_deploy_netbox.yml \
		$(EXTRA_JSON_ARG) $(EXTRA)

netbox-backups: ## Reconcile the NetBox application-aware backup timer
	$(ANSIBLE_PB) -i $(NETBOX_INVENTORY) playbooks/113_configure_netbox_backups.yml

netbox-terraform-fmt: ## Check planned NetBox guest Terraform formatting
	$(TERRAFORM) -chdir=$(NETBOX_TF_DIR) fmt -check -recursive

netbox-terraform-init: ## Initialize the pinned Proxmox provider without a backend
	$(TERRAFORM) -chdir=$(NETBOX_TF_DIR) init -backend=false

netbox-terraform-validate: ## Validate the planned NetBox guest Terraform configuration
	$(TERRAFORM) -chdir=$(NETBOX_TF_DIR) validate

netbox-terraform-plan: ## Plan the planned NetBox guest. Apply stays a manual, reviewed step.
	$(TERRAFORM) -chdir=$(NETBOX_TF_DIR) plan

terraform-fmt: ## Check Home Lab guest Terraform formatting
	$(TERRAFORM) -chdir=$(TF_DIR) fmt -check -recursive

terraform-init: ## Initialize the pinned Proxmox provider without a backend
	$(TERRAFORM) -chdir=$(TF_DIR) init -backend=false

terraform-validate: ## Validate Home Lab guest Terraform configuration
	$(TERRAFORM) -chdir=$(TF_DIR) validate

terraform-plan: ## Import-first, non-destructive Terraform plan for VMs 300 and 310
	$(TERRAFORM) -chdir=$(TF_DIR) plan

validate: ## Stage 11: full three-node cluster validation + migration report
	$(PB) playbooks/90_validate_cluster.yml $(EXTRA_JSON_ARG) $(EXTRA)
