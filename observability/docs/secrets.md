# Secrets and credential boundary

The agent never asks for, reads, prints, copies, logs, or commits the values in
this document. Perform these steps locally and report completion only.

## Proxmox API token

At a Proxmox root shell, create a dedicated user and privilege-separated token.
The final `token add` command prints its secret once; keep that output out of
terminal capture and chat.

```bash
pveum user add monitoring@pve --comment "Prometheus read-only monitoring"
pveum acl modify / --user monitoring@pve --role PVEAuditor
pveum user token add monitoring@pve prometheus --privsep 1
pveum acl modify / --token 'monitoring@pve!prometheus' --role PVEAuditor
```

Create `/etc/observability/secret-store/pve.yml` on VM 310, mode `0600`:

```yaml
default:
  user: monitoring@pve
  token_name: prometheus
  token_value: REPLACE_WITH_TOKEN_VALUE
  verify_ssl: false
```

`verify_ssl: false` is limited to the self-signed PVE API on the private LAN;
the token has auditor privileges only.

## Synology SNMPv3

In DSM, enable SNMPv3 and create a dedicated read-only monitoring identity with
authentication and privacy enabled. Do not enable v1/v2c. Use either the
official SNMP Exporter generator or the version-matched `v0.30.1` default
configuration with a private `synology_v3` `authPriv` entry. Install the result as
`/etc/observability/secret-store/snmp.yml`, owner root, mode `0600`.

## Grafana administrator

Create a random password locally and write it directly to
`/etc/observability/secret-store/grafana_admin_password`, owner root, mode `0600`.
Avoid command-line arguments and shell history. Grafana reads it through a
Docker secret; it never enters Compose or Ansible inventory. Use at least 20
bytes; deployment checks only file metadata and never reads the value.

## the SMTP provider SMTP

The Alertmanager receiver authenticates as the dedicated the SMTP provider mailbox
`alerts@example.com` and delivers notifications to `operator@example.com` over
STARTTLS on port 587. Write only the mailbox password, with no label, to
`/etc/observability/secret-store/smtp_password`; set owner to root and mode
to 0600. The Compose definition mounts it into Alertmanager as a file-backed
secret, and the deployment gate checks only its metadata.

Docker Compose implements local file-backed secrets as bind mounts and does not
translate root-only ownership. Alertmanager therefore runs as container root so
the host password can remain `root:root` mode 0600. Its container root
filesystem is read-only, all Linux capabilities are dropped, and only the
dedicated root-owned Alertmanager state mount remains writable. Do not relax
the host secret mode or group ownership.

Speedtest Tracker on VM 300 uses the same dedicated mailbox but keeps its copy
of the password in `/etc/homelab/secret-store/speedtest-tracker.env` as
`MAIL_PASSWORD`. Enter both copies locally and keep the password out of shell
arguments, terminal capture, Git, and chat.

## Backup and recovery

Application archives deliberately exclude `/etc/observability/secret-store`.
Maintain one encrypted offline copy in the operator's password manager or an
encrypted Synology Hyper Backup set. Test that copy during the quarterly restore
drill. Losing telemetry history is acceptable; losing the ability to recreate
read-only credentials is inconvenient but not destructive.
