# GitHub Actions integration (prepared, not enabled)

Prefer JWT/OIDC auth with repository, ref, workflow, and environment claims
bound to a narrow policy. Do not issue credentials to forked pull requests or
untrusted code. Pin every action to an immutable commit SHA and keep job output
free of secret values.

Until the trust model is reviewed, CI remains offline validation only and has no
OpenBao or Home Lab network access.

