# Troubleshooting

## Start with read-only evidence

```bash
docker compose --project-directory /opt/observability/docker ps
docker stats --no-stream
curl -fsS http://192.168.0.31:9090/-/ready
curl -fsS http://192.168.0.31:3100/ready
curl -fsS http://192.168.0.31:3000/api/health
df -hT / /srv/observability
free -h
```

## Prometheus target down

Check `/api/v1/targets` before changing configuration. Distinguish exporter down,
target authentication failure, firewall rejection, and absent metrics. PVE uses
one cluster target intentionally. Never solve an auth failure by granting the
token administrator access.

## Loki disk growth

Confirm `compactor` is running and `retention_enabled` is true. Loki filesystem
storage does not enforce a free-space limit, so the 7-day policy and filesystem
alerts are both required. Reduce retention/ingestion or noisy sources; do not
move the active Loki store to NFS.

## Missing Docker logs

Check Alloy's Docker socket is read-only mounted, the source host can reach
`.0.31:3100`, and UFW admits only VM 300. Do not add unbounded labels to make one
query easier. Container name, host, job, and systemd unit are sufficient.

## Synology scrape slow or failing

Use one manual SNMPv3 walk from VM 310 and check DSM CPU/load. Keep the interval
at 120 seconds and `max_repetitions` at 10 on the the NAS. Never fall back to an
SNMPv2 community string for convenience.

## Synology target is up but dashboard panels say no data

Target health proves only that the exporter returned a valid scrape. Confirm the
payload schema before editing Grafana:

```bash
curl -fsSG --data-urlencode 'query=count by (__name__) ({job="synology-snmp"})' \
  http://192.168.0.31:9090/api/v1/query | jq -r '.data.result[].metric.__name__'
```

The installed Synology module exports unprefixed names including `raidStatus`,
`raidFreeSize`, `diskHealthStatus`, `diskTemperature`, and
`storageIONWrittenX`. It does not currently export `sysUpTime` or the old
`hrStorage*` family. The version-controlled dashboard deliberately uses only
metrics confirmed in the live schema. Run `make validate` to require both a
healthy target and non-empty RAID/disk payloads.

## Memory pressure

Inspect container working sets. Limits are deliberate; raise only the affected
service after measuring. Query concurrency and scrape/cardinality should be
reduced before increasing VM RAM because `pve1` also has two protected guests.

## Recovery

Do not delete a corrupt TSDB/WAL as a first response. Stop the affected service,
retain the data, validate the last application archive, and rebuild in an
isolated path or VM. See [backup-restore.md](backup-restore.md).
