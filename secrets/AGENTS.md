# OpenBao automation contract

This repository inherits `../AGENTS.md`. The following rules are additional and
take precedence for OpenBao work.

- Never initialize, unseal, rekey, generate a root token, restore Raft, remove a
  Raft peer, or migrate a credential without an explicit operator approval for
  that exact stage.
- There is intentionally no automated `bao operator init` command. Initialization
  is an attended ceremony with five PGP recipients and a threshold of three.
- Never inspect, print, copy into chat, or commit tokens, unseal shares, SecretIDs,
  TLS private keys, backup identities, SMTP passwords, or application secrets.
- Read secret material only from operator-supplied runtime files. Secret-bearing
  Ansible tasks use `no_log: true`; Python programs never include values in output.
- Raft lives only on local VM disks. Synology is a backup destination, never a
  live storage dependency.
- Every playbook uses `serial: 1` and `any_errors_fatal: true`. Mutating stages
  require a boolean and exact confirmation string. No aggregate deploy target is
  permitted.
- No archive pruning. Retention is implemented through separately reviewed NAS
  snapshots; this repository never deletes a backup.
- A restore may run only in a verified isolated VM/network. Production restore,
  forced restore, and peer removal are emergency procedures documented but not
  executable from normal targets.

