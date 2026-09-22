# Initialization and unseal ceremony

This is manual because its output is the recovery key material. Never run it in
Ansible, a terminal recorder, CI, `tee`, shell history, chat, or an AI session.

## Preparation

- Five distinct PGP public keys are ready and independently controlled.
- The operator has verified the VIP and all three direct TLS endpoints.
- Screen recording, terminal logging, clipboard managers, and shell history are
  disabled for the ceremony terminal.
- A second person or written checklist verifies the five-share/three-threshold
  parameters.

## Initialize exactly once

On the attended workstation, set `BAO_ADDR`, `BAO_CACERT`, and run:

```bash
bao operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys=/secure/recipient1.pub,/secure/recipient2.pub,/secure/recipient3.pub,/secure/recipient4.pub,/secure/recipient5.pub \
  -root-token-pgp-key=/secure/operator-root-token.pub
```

Distribute each encrypted share to its named custodian. The decrypted shares
must not be placed together, on the Synology, in Git, in cloud notes, or beside
the encrypted snapshots. Decrypt the initial root token only for bootstrap.

Unseal each node with three independently supplied shares:

```bash
bao operator unseal
```

Do not put a share on the command line. Repeat interactively until each node is
unsealed. Confirm one active and two standby nodes with:

```bash
bao operator raft list-peers
bao status
```

After policies, durable operator authentication, audit, backup, and recovery
access are proven, revoke the initial root token. Record only its revocation
date and accessor-free evidence—not the token.

After any reboot, three custodial shares are again required per sealed node.
Auto-unseal is intentionally deferred until a recovery design exists that does
not depend circularly on this OpenBao cluster.

