# HomeLab

Sanitised design documentation and safety tooling from a three-node Proxmox VE
home lab, its observability stack, and a planned secrets/recovery plane.

Everything here is published deliberately. Inventories, addresses, credentials,
private keys, host captures, state files, VM images and backups are not in this
repository and never will be — the repository denies everything by default and
allows only this `public/` tree.

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

The test files below are what enforce those rules.

## Contents

### Design documentation

| document | subject |
|---|---|
| [cluster-architecture.md](docs/cluster-architecture.md) | migrating three standalone Proxmox hosts into one cluster without losing a VM |
| [cluster-rollback.md](docs/cluster-rollback.md) | the documented way back from every migration stage |
| [observability-architecture.md](docs/observability-architecture.md) | bounded Prometheus / Grafana / Loki / Alertmanager design |
| [observability-backup-restore.md](docs/observability-backup-restore.md) | three recovery layers, and the pitfalls that bit us |
| [observability-versions.md](docs/observability-versions.md) | pinned image and component versions |
| [secrets-architecture.md](docs/secrets-architecture.md) | three-voter Raft secrets cluster, failure domains, capacity controls |
| [secrets-threat-model.md](docs/secrets-threat-model.md) | what the secrets plane defends against, and what it does not |
| [secrets-hierarchy.md](docs/secrets-hierarchy.md) | credential migration order and break-glass paths |
| [secrets-versions.md](docs/secrets-versions.md) | pinned versions |

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

They are **sanitised copies**, not a runnable checkout. Real addresses,
hostnames, MAC addresses, key fingerprints, domains and mailboxes have been
replaced with stable placeholders:

| placeholder | meaning |
|---|---|
| `192.168.0.0/24` | the lab network |
| `192.168.0.11` / `.12` / `.13` | the three Proxmox nodes `pve1` / `pve2` / `pve3` |
| `192.168.0.20` | the NAS (`nas1`) — DNS and NFS backup target |
| `192.168.0.30` / `.31` | services VM / observability VM |
| `192.168.0.40`–`.43` | secrets cluster VIP and voters |
| `lab.example.com` | the private zone |
| `52:54:00:00:00:00` | any MAC address |
| `SHA256:EXAMPLE_FINGERPRINT_REDACTED` | any host key fingerprint |

The test files therefore reference paths that exist only in the private
repositories. They are published to show the shape of the safety contract, not
to be executed here.
