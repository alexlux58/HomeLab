# Architecture and data flow

```text
Proxmox API (.11) ── PVE Exporter ─────┐
Synology (.25) ───── SNMPv3 Exporter ───┤
VM 300 / VM 310 ─── Node + cAdvisor ────┤── Prometheus ── Alertmanager
HTTPS/DNS/ICMP ───── Blackbox Exporter ─┘        │
                                                 └── Grafana
VM 300 Docker+journal ── Alloy ──┐               ┌── Grafana
VM 310 Docker+journal ── Alloy ──┼── Loki ──────┘
PVE / DSM syslog (later gate) ───┘
```

VM 310 lives on `pve1` NVMe because it is the only node with sufficient CPU,
RAM, and fast local storage. It is not HA: guest disks are local on every node.
Failure recovery is rebuild-and-restore, not pretend failover.

Only Grafana, Prometheus, and Alertmanager UIs are reachable from the /22 LAN.
Loki ingestion is allowed only from VM 300; syslog 1514 is LAN-only. Exporters
stay on the private Docker network, except the two VM 300 agents, whose ports
accept only VM 310. Grafana anonymous access and user signup are disabled.

PVE Exporter uses one least-privilege cluster target. It reports all nodes,
guests, and storage without installing software on the hypervisors. Host-level
Node Exporter and rsyslog forwarding can be added later, one PVE node at a time,
after the central stack proves stable. Synology uses read-only SNMPv3 only.

Metrics and logs are useful but replaceable. Git is authoritative for topology,
dashboards, alerts, and deployment. Grafana SQLite and a coherent Prometheus
snapshot are backed up. Loki history can be discarded and recollected.
