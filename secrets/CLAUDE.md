# AI context — Home Lab OpenBao

Read `../CLAUDE.md`, `../AGENTS.md`, and this file before doing any work.

## Current state

**Corrected 2026-09-19.** An earlier revision said "no OpenBao VM has been
provisioned". That is obsolete.

All three voters **exist and are running** — VMs 320 `bao-1` on `pve1`, 321
`bao-2` on `pve2`, 322 `bao-3` on `pve3`, verified by `qm list` on
each node on 2026-09-19. OpenBao 2.6.1 is installed with valid TLS, HAProxy and
Keepalived, and all three report `initialized: false, sealed: true`.

**The cluster is uninitialized on purpose, and that gate still holds.**
Initialisation is an attended ceremony with five PGP recipients and a threshold
of three. No automation and no assistant may run `bao operator init`.

Two gaps to know about:

- **None of VMs 320–322 has a Proxmox backup.** They appear in no `vzdump` job
  and have zero archives on the NAS. Tracked as Phase 1.6 in
  `../HOMELAB-ROADMAP.md`.
- **All three are `onboot=0`.** A host reboot silently removes them; this already
  happened once, and the resulting cloud-init re-run regenerated the SSH host
  keys of 320 and 321.
- **This repository is still not a Git repository** and holds live Terraform
  state. That is the single largest unrecoverable-loss risk in the estate,
  tracked as Phase 1.2.

| node | VMID | Proxmox host | address | local disk | resources |
|---|---:|---|---|---|---|
| `bao-1` | 320 | `pve1` | `192.168.0.41/24` | `local` NVMe | 2 vCPU, 2 GiB, 24 GiB |
| `bao-2` | 321 | `pve2` | `192.168.0.42/24` | `local-lvm` HDD | 2 vCPU, 2 GiB, 24 GiB |
| `bao-3` | 322 | `pve3` | `192.168.0.43/24` | `local-lvm` SSD | 2 vCPU, 2 GiB, 24 GiB |

Candidate VIP: `192.168.0.40`; API name: `bao.lab.example.com`. DNS currently
resolves that name to VM 300 through the existing wildcard, so it is **not** an
approval or proof of correct configuration. Reserve all four addresses and add
specific Synology DNS records before provisioning.

Current cluster discovery on 2026-08-24 showed three quorate Proxmox nodes. The
pve2 had about 4 GiB available with VM 300 running and is the limiting failure
domain. Do not increase `bao-2` above 2 GiB without repeating discovery.

## Safety gates

1. `make check` is offline and safe.
2. `make discover-plan` reads only local sanitized inputs.
3. Terraform plan is allowed only after a dedicated Proxmox token and SSH public
   key are supplied at runtime. Apply requires separate operator approval.
4. Ansible preflight is read-only. Host/install/TLS stages are separately gated.
5. Stop after `make validate-uninitialized`. It must report initialized=false.
6. Initialization is manual and attended. Never capture its output.
7. Post-init configuration, backup, monitoring, and one low-risk PVE exporter
   pilot each require their own approval.

## Ownership

- Terraform: VM identity, placement, CPU/RAM, local disks, NICs, cloud-init.
- Ansible: Ubuntu hardening, verified OpenBao binary, TLS, systemd, firewall,
  HAProxy/Keepalived, audit, backup timers, and monitoring integration.
- Python: repository safety, runtime API reconciliation, backup verification,
  recovery bundles, discovery reports, and secret-reference inventories.
- Operator: DHCP/DNS, offline CA custody, unseal ceremony, runtime secret files,
  Synology account/share creation, Terraform apply, and all live approvals.

The initial secret pilot is the read-only PVE exporter token on VM 310. Browser
passwords remain in macOS Keychain. The backup decryption identity and at least
three unseal shares must remain outside both OpenBao and the Synology backup
location.

