# Rotation runbook

| item | normal interval/event | procedure |
|---|---|---|
| AppRole SecretID | 24 hours or suspected exposure | create response-wrapped replacement, install, verify Agent, revoke old accessor |
| application secret | source policy or exposure | rotate at source, write new KV version, restart one consumer, revoke old |
| PVE API token | 90 days/exposure | create restricted replacement, pilot exporter, revoke old token |
| SMTP password | the SMTP provider policy/exposure | reset mailbox, update OpenBao, validate two consumers, revoke old |
| listener certificate | before 30-day warning | issue new leaf, deploy one node, reload, verify, repeat |
| PKI intermediate | before expiry or key concern | create new intermediate, cross/dual trust, issue leaves, remove old after proof |
| age backup identity | key exposure | add new recipient, test decrypt, switch jobs, retain old identity for historical backups |
| unseal shares | custodian/key exposure | use authenticated OpenBao key-rotation procedure with new PGP recipients |

Never rotate network-device credentials automatically without a tested rollback.
Never overwrite the only copy of a credential before its replacement has been
proven. Record dates and accessors/identifiers only, not values.

