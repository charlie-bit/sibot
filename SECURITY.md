# Security Policy

## Reporting a vulnerability

**Do not open a public issue, pull request, or discussion for a security problem.**

Use either:

- GitHub's private reporting: repository **Security** tab → **Report a vulnerability**, or
- a direct message to a maintainer listed in the repository profile.

Please include:

- affected version / commit,
- the class of issue (e.g. secret exposure, injection, auth bypass),
- reproduction steps or a proof of concept,
- impact you believe it has.

## What to expect

- Acknowledgement within 7 days.
- An assessment within 14 days.
- Credit in the fix's release notes, unless you prefer to stay anonymous.

## Scope

This repository's code and configuration. Third-party dependencies should be reported
upstream, but tell us as well so we can pin or patch.

## Please do not

- Run automated scanners against infrastructure you do not own.
- Access, modify, or exfiltrate data that is not yours while proving an issue.
- Publish details before a fix is available.

## Secrets in history

If you find credentials committed to this repository, report them privately as above.
Assume any secret that reached a public commit is already compromised — the fix is rotation,
not deletion.
