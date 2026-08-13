# nfSensei Migration (Future)

Tracking [nfSensei](https://nfsensei.org/) as a possible future replacement for OPNsense.

## What is nfSensei?

A Rust-based, Linux-hosted firewall OS built by Scott Ullrich (pfSense co-founder),
positioned as a pfSense/OPNsense successor:

- Candidate/running config model — diff-before-apply, auto-rollback on failed boot
- API-first design (one API for web UI and CLI)
- On-device AI assistant ("Clyde")
- Layer 7 / JA3 TLS-fingerprint matching enforced in a VPP dataplane
- Ships as ~5 self-contained binaries; Lua package ecosystem (nfpack)

## Status

**Pre-1.0, moving fast.** As of 2026-08-04 it was at build 0.51.163, having shipped ten
point releases in the prior two weeks. No confirmed public GitHub repo, OPNsense
config-import path, or hardware-support list as of 2026-08-13.

A weekly automated check watches [blog.nfsensei.org](https://blog.nfsensei.org/) and
[nfsensei.org](https://nfsensei.org/) for a stable/1.0/production-ready milestone,
config-import tooling, or a public source release.

## Plan

No action until nfSensei reaches a release considered stable for home/production use.
When it does, re-evaluate against the current OPNsense setup documented in
[01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md) and
[05_FIREWALL_RULES.md](05_FIREWALL_RULES.md) before migrating.
