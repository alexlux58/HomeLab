# HomeLab

Sanitised automation and design documentation from a three-node Proxmox VE home
lab, its observability stack, and a secrets/recovery plane.

Everything here is published deliberately. Inventories with real addresses,
credentials, private keys, host captures, state files, VM images and backups are
not in this repository and never will be — the repository denies everything by
default and allows only this `public/` tree.

## What this is really about

The interesting part of this lab is not the service list. It is that the
automation is **gated**: it operates on live machines holding irreplaceable
data, so "make the tests pass" is not the goal — not destroying anything is.

That principle is enforced mechanically rather than by discipline:

- every playbook runs `serial: 1` with `any_errors_fatal: true`;
- there is no aggregate `deploy-all` target, and a unit test fails the build if
  anyone adds one;
- every destructive task carries both a `never` and a `destructive` tag, so it
  additionally requires `--tags all,destructive`;
- every `allow_*` flag defaults to `false` and is paired with an exact
  confirmation string, both checked by a real assertion;
- no backup archive is ever deleted or pruned — NFS storage is registered with
  `keep-all=1`;
- `ignore_errors: true`, `rm -rf`, `pvecm expected`, `StrictHostKeyChecking=no`
  and plaintext password variables appear nowhere in executable code.

The `tests/` directory in each area is what enforces those rules.

## Layout

Three internal repositories are published here as sanitised snapshots.

| area | contents |
|---|---|
| [`cluster/`](cluster/) | the Proxmox migration and steady-state Home Lab toolkit — 27 playbooks, 16 roles, 7 test modules, two import-first Terraform roots |
| [`observability/`](observability/) | the Prometheus / Grafana / Loki / Alertmanager stack — 7 playbooks, 8 roles, Compose and scrape configuration |
| [`secrets/`](secrets/) | the three-voter OpenBao recovery plane — 13 playbooks, 12 roles, PKI tooling, Terraform modules |

`docs/` and `tests/` at the top level are a curated reading path into the same
material, kept because they are the stable entry points. They are regenerated
from the same sources as the snapshots, so the two copies cannot drift apart.

### Suggested reading order

| document | subject |
|---|---|
| [cluster-architecture.md](docs/cluster-architecture.md) | migrating three standalone Proxmox hosts into one cluster without losing a VM |
| [cluster-rollback.md](docs/cluster-rollback.md) | the documented way back from every migration stage |
| [observability-architecture.md](docs/observability-architecture.md) | bounded Prometheus / Grafana / Loki / Alertmanager design |
| [observability-backup-restore.md](docs/observability-backup-restore.md) | three recovery layers, and the pitfalls that bit us |
| [secrets-architecture.md](docs/secrets-architecture.md) | three-voter Raft secrets cluster, failure domains, capacity controls |
| [secrets-threat-model.md](docs/secrets-threat-model.md) | what the secrets plane defends against, and what it does not |
| [secrets-hierarchy.md](docs/secrets-hierarchy.md) | credential migration order and break-glass paths |

Pinned component versions are in
[observability-versions.md](docs/observability-versions.md) and
[secrets-versions.md](docs/secrets-versions.md).

### Safety tests

| test | enforces |
|---|---|
| [test_cluster_repo_safety.py](tests/test_cluster_repo_safety.py) | the migration safety contract above |
| [test_observability_repository.py](tests/test_observability_repository.py) | image pinning, memory limits, retention bounds, secret boundaries |
| [test_secrets_repository.py](tests/test_secrets_repository.py) | no automated initialisation, no archive pruning, no secret logging |

### AAA deployment (in progress)

FreeRADIUS design and read-only discovery tooling — see
[aaa-deployment.md](aaa-deployment.md) and
[120_aaa_preflight.yml](120_aaa_preflight.yml).

## Reading these files

They are **sanitised copies, not a runnable checkout.** Real addresses,
hostnames, MAC addresses, key fingerprints, domains, mailboxes, usernames and
hardware serials are replaced with stable placeholders by a local script that is
itself kept out of this repository — its substitution table is a map of the real
values, so publishing it would defeat its purpose.

| placeholder | meaning |
|---|---|
| `192.168.0.0/24` | the lab network |
| `192.168.0.11` / `.12` / `.13` | the three Proxmox nodes `pve1` / `pve2` / `pve3` |
| `192.168.0.20` | the NAS (`nas1`) — DNS and NFS backup target |
| `192.168.0.30` / `.31` | services VM / observability VM |
| `192.168.0.40`–`.43` | secrets cluster VIP and voters |
| `192.168.0.50` | IPAM/DCIM VM |
| `192.168.0.81` / `.82` | other LAN devices |
| `lab.example.com` | the private DNS zone |
| `labadmin` / `labuser` | operator accounts |
| `52:54:00:00:00:00` | any MAC address |
| `SHA256:EXAMPLE_FINGERPRINT_REDACTED` | any SSH host key fingerprint |
| `SERIAL-REDACTED-n` | a disk serial number |

Because substitution is global and consistent, cross-references still line up:
the node called `pve2` in a playbook is the same `pve2` in the runbook.

A published file may therefore reference a path or fixture that exists only in
the private repositories. That is expected — these snapshots demonstrate the
shape of the automation and its safety contract, not a deployable artifact.

## Deliberately not published

- **Captured host state.** `artifacts/` and `host-configs/` hold real discovery
  output and guest configuration and are excluded entirely, as are the test
  fixtures built from them. Sanitising those would still publish the shape of
  the estate, VM by VM.
- **The LAN device inventory**, which enumerates personal devices.
- **Secrets of any kind.** Authentication is public-key only; on-host secret
  files are operator-installed, root-owned and mode `0600`, and no secret enters
  Git, Ansible inventory, or an application archive. The directory *paths* are
  masked here too.
- **The full AAA deployment plan**, until its host-specific details are
  reviewed. The stub above records that decision.
- **The sanitiser, the denylist and the publishing script**, for the reason
  given above.

Every file here is scanned before each commit against a literal denylist of real
values plus structural patterns for private IPv4 addresses, MAC addresses, SSH
fingerprints, private keys, e-mail addresses and API tokens. A pre-commit hook
blocks the commit on any hit, and also blocks staging anything outside this
tree.
