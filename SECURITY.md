# Security policy

## Reporting a vulnerability

Please report security problems privately, not in a public issue: use **Report a vulnerability**
on the repository's *Security* tab (GitHub private vulnerability reporting). Include the version
(`curulia version`), what an attacker can do, and the steps to reproduce it.

You can expect an answer within a week. Once a fix is released, the report is published as a
security advisory, with credit if you want it.

## Supported versions

Fixes go into the latest release. Curulia upgrades its database automatically, so updating to
the latest release is the way to get a fix.

## Scope

In scope: the portal and the `curulia` command, including sign-in (local, AD, Kerberos,
two-step), permissions, the API, backups and database encryption.

Out of scope: problems that need an administrator account to begin with (administrators are
trusted), missing HTTPS on an installation that runs without it, and a deployment started with
`--dev`, which is for trials only.
