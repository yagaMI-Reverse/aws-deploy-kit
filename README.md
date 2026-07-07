# AWS Deploy Kit

**One-paste provisioning for an AWS EC2 free-tier box: a FastAPI service + self-hosted n8n behind nginx.**

Launch a blank Ubuntu `t3.micro`, paste [`cloud-init.yaml`](./cloud-init.yaml)
into the *User data* field, and the instance provisions itself on first boot —
no manual SSH-and-apt, no snowflake server. Infrastructure as a single, readable
file you can diff and re-run.

```
                          AWS EC2 t3.micro (Ubuntu, free tier)
        ┌──────────────────────────────────────────────────────────┐
        │                        nginx  :80/:443                      │
        │             /  ──────────►  FastAPI (uvicorn) :8000         │
        │             /n8n/  ──────►  n8n (systemd)     :5678         │
        │                                                             │
        │   cloud-init.yaml installs + wires all of the above         │
        └──────────────────────────────────────────────────────────┘
```

## What it provisions

| Component | How | Reachable at |
|---|---|---|
| **FastAPI app** (DocuChat RAG backend) | cloned + venv + `systemd` | `/` |
| **n8n** self-hosted automation | `npm i -g n8n` + `systemd` | `/n8n/` |
| **nginx** reverse proxy | apt + our site config | `:80` (→ `:443` with certbot) |

Services run under `systemd` (auto-restart, boot-persistent). Secrets go in
`/etc/docuchat.env` and `/etc/n8n.env` — never baked into the image.

## Files

| File | Purpose |
|---|---|
| [`cloud-init.yaml`](./cloud-init.yaml) | the whole machine, as user-data |
| [`nginx/app.conf`](./nginx/app.conf) | reverse-proxy config (also embedded in cloud-init) |
| [`RUNBOOK.md`](./RUNBOOK.md) | the account-dependent steps — **billing alarm first**, launch, verify, HTTPS, cost guardrails |

## Why this shape

- **Reproducible** — the box is defined by one file; rebuild it identically any time.
- **Cost-safe** — the runbook makes a **$1 billing alarm the first step**, before any resource exists.
- **Portable** — the same cloud-init runs on any Ubuntu VPS (Hetzner, DigitalOcean, Lightsail); EC2 is just the free-tier target.
- **Real services** — it deploys an actual FastAPI RAG backend and n8n, not a hello-world.

## Status

The kit is complete and validated (YAML + nginx config lint clean). Execution on
a live instance is a ~20-minute run of `RUNBOOK.md` once an AWS account exists.

---

*Part of the [yagaMI-Reverse](https://github.com/yagaMI-Reverse) portfolio.*
