SHELL := /bin/bash
.SHELLFLAGS := -eu -o pipefail -c
.DEFAULT_GOAL := help

VENV ?= .venv
PYTHON ?= python3
ANSIBLE_PLAYBOOK ?= $(VENV)/bin/ansible-playbook
INVENTORY ?= ansible/inventory/hosts.yml
TERRAFORM ?= terraform
TF_DIR ?= terraform
EXTRA_JSON ?=
EXTRA_JSON_ARG = $(if $(strip $(EXTRA_JSON)),-e '$(EXTRA_JSON)',)

.PHONY: help setup lint test check syntax terraform-fmt terraform-init \
	terraform-validate terraform-plan preflight prepare-hosts install-server \
	configure-tls configure-ha validate-uninitialized configure-post-init \
	validate-post-init configure-backup configure-monitoring backup-now inspect-backup \
	discover-plan recovery-bundle

help: ## Show independently gated targets
	@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-25s %s\n", $$1, $$2}'

setup: ## Install pinned offline-controller dependencies
	$(PYTHON) -m venv $(VENV)
	$(VENV)/bin/pip install -r requirements.txt
	$(VENV)/bin/ansible-galaxy collection install -r requirements.yml

lint: ## Run repository, Python, YAML, and Ansible safety checks
	$(VENV)/bin/yamllint -s .
	ANSIBLE_CONFIG=ansible.cfg $(VENV)/bin/ansible-lint ansible/
	$(VENV)/bin/ruff check scripts tests
	$(VENV)/bin/ruff format --check scripts tests
	$(VENV)/bin/python scripts/validate_repo.py --root .

test: ## Run offline unit tests
	$(VENV)/bin/python -m unittest discover -s tests -v

syntax: ## Syntax-check every playbook without contacting a host
	@for playbook in ansible/playbooks/*.yml; do \
		ANSIBLE_CONFIG=ansible.cfg $(ANSIBLE_PLAYBOOK) -i $(INVENTORY) --syntax-check "$$playbook"; \
	done

check: lint test syntax terraform-fmt terraform-validate ## Run all offline validation

terraform-fmt: ## Check Terraform formatting
	$(TERRAFORM) -chdir=$(TF_DIR) fmt -check -recursive

terraform-init: ## Initialize the pinned provider without a backend
	$(TERRAFORM) -chdir=$(TF_DIR) init -backend=false

terraform-validate: ## Validate Terraform without contacting Proxmox
	$(TERRAFORM) -chdir=$(TF_DIR) validate

terraform-plan: ## Produce a reviewed plan; no apply target exists
	$(TERRAFORM) -chdir=$(TF_DIR) plan

discover-plan: ## Refresh sanitized secret-name and Git-history inventories
	$(VENV)/bin/python scripts/secret_reference_inventory.py \
		--root .. --output artifacts/secret-references.json
	$(VENV)/bin/python scripts/git_history_secret_names.py \
		--repo ../proxmox-cluster-migration \
		--repo ../homelab-observability \
		--output artifacts/git-history-secret-filenames.json

preflight: ## Read-only guest validation after first boot
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/00_preflight.yml

prepare-hosts: ## Gated Ubuntu hardening and firewall stage
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/10_prepare_hosts.yml $(EXTRA_JSON_ARG)

install-server: ## Gated signed OpenBao binary and systemd installation
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/20_install_server.yml $(EXTRA_JSON_ARG)

configure-tls: ## Gated operator-supplied TLS deployment
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/30_configure_tls.yml $(EXTRA_JSON_ARG)

configure-ha: ## Gated HAProxy/Keepalived and OpenBao configuration
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/40_configure_ha.yml $(EXTRA_JSON_ARG)

validate-uninitialized: ## Prove TLS works and OpenBao remains uninitialized
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/50_validate_uninitialized.yml

configure-post-init: ## Gated policies/auth/engines after the attended ceremony
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/60_configure_post_init.yml $(EXTRA_JSON_ARG)

validate-post-init: ## Read-only proof of quorum, mounts, auth, audit, and TLS
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/65_validate_post_init.yml

configure-backup: ## Gated encrypted Raft snapshot timer installation
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/70_configure_backup.yml $(EXTRA_JSON_ARG)

configure-monitoring: ## Gated OpenBao telemetry and audit shipping integration
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/80_configure_monitoring.yml $(EXTRA_JSON_ARG)

backup-now: ## Trigger one encrypted snapshot on the elected backup node
	$(ANSIBLE_PLAYBOOK) -i $(INVENTORY) ansible/playbooks/90_backup_now.yml $(EXTRA_JSON_ARG)

inspect-backup: ## Offline decrypt/inspect of one snapshot (SNAPSHOT and AGE_IDENTITY required)
	@test -n "$(SNAPSHOT)" -a -n "$(AGE_IDENTITY)" || { echo "set SNAPSHOT and AGE_IDENTITY"; exit 2; }
	$(VENV)/bin/python scripts/verify_snapshot.py --input "$(SNAPSHOT)" --age-identity "$(AGE_IDENTITY)"

recovery-bundle: ## Build a secret-free configuration recovery bundle
	$(VENV)/bin/python scripts/build_recovery_bundle.py --root . --output artifacts/openbao-recovery-bundle.tar.gz
