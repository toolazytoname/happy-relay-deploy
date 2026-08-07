# happy-relay-deploy

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Network default](https://img.shields.io/badge/default-tailnet--only-blue.svg)](SECURITY.md)

[简体中文](README.md) · [English](README.en.md)

A Claude Code and Codex skill for self-hosting `happy-server-light` when a phone must control an AI coding session on another machine.

If Happy server and its daemon both run locally on the phone, you do **not** need this repository. Use [`android-ai-stack`](https://github.com/toolazytoname/android-ai-stack) for that local-first topology. This repository owns only the optional remote-relay boundary.

## Use cases

- Reach a home, office, or cloud workstation from the Happy mobile app through a private tailnet.
- Apply the pinned compatibility patch for the Happy App v1.2+ v3 session-message route.
- Evaluate a public HTTPS relay only after explicitly accepting its open-registration and abuse risks.

## Security model

The default deployment binds `happy-server-light` to loopback and publishes it only inside a Tailscale tailnet. Public ingress is a documented exception, not the default. The systemd service runs as a dedicated unprivileged `happy` user, and operational files live under `/home/happy`.

## Install as a skill

```bash
git clone https://github.com/toolazytoname/happy-relay-deploy.git
cp -r happy-relay-deploy ~/.config/agents/skills/
```

The repository can also be placed under `~/.kimi/skills/` or `~/.claude/skills/`. Restart the agent, then ask it to deploy a Happy relay.

## Contents

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | Local/remote decision tree, private-tailnet deployment, and public exposure risks |
| `assets/v3SessionRoutes.ts` | Pinned copy of the v3 session-message compatibility patch |
| `assets/loopback-bind.patch` | Makes the pinned upstream revision honor `HAPPY_BIND_HOST` instead of always binding all interfaces |
| `assets/happy-server.service` | Hardened systemd service template |
| `references/ops-troubleshooting.md` | Operations, backup, diagnosis, and unknown-account audit steps |
| [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) | Patch provenance and pinned upstream commit |

## Upstream projects

- [Happy](https://github.com/slopus/happy)
- [happy-server-light](https://github.com/leeroybrun/happy-server-light)
- [Upstream compatibility PR](https://github.com/leeroybrun/happy-server-light/pull/2)

## License

MIT. Third-party provenance is documented separately.
