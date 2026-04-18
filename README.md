# Deploy n8n on Render

> [!IMPORTANT]
> **View full deployment instructions in the [**Render docs**](https://render.com/docs/deploy-n8n).**

This template defines a [`render.yaml`](https://github.com/render-examples/n8n/blob/main/render.yaml) file you can use to deploy [n8n](https://n8n.io/) on Render. Click **Use this template** in the upper right to copy this template into your account as a new repo.

The `render.yaml` file defines the following resources:

- A web service that pulls and runs the official n8n Docker image
- A Render Postgres database that stores n8n data

Each of the above uses a free instance type by default.
 
This repository contains **infrastructure code only** — no workflows, no business logic, no agents.

---

## Purpose

n8n acts as the orchestration layer in a multi-agent automation system.  
It runs as a Docker-based Web Service on Render, with GitHub as the source of truth.

---

## Architecture

```
WhatsApp → n8n → LLM / Agents → MCP → GitHub → Render
```

| Component | Role |
|-----------|------|
| **WhatsApp** | Event source (messages / triggers) |
| **n8n** | Workflow orchestration & routing |
| **LLM / Agents** | AI processing (e.g., OpenAI, Anthropic) |
| **MCP** | Model Context Protocol server |
| **GitHub** | Source of truth / version control |
| **Render** | Cloud hosting (Docker Web Service) |

---

## Requirements

- A [GitHub](https://github.com) account with access to this repository
- A [Render](https://render.com) account (free tier is sufficient)

---

## Deployment Guide (Render)

### 1 — Create a Web Service

1. Log in to [Render Dashboard](https://dashboard.render.com).
2. Click **New → Web Service**.
3. Connect this GitHub repository.

### 2 — Set Environment Variables

In the **Environment** tab of the Web Service, add the following variables:

| Variable | Value |
|----------|-------|
| `N8N_BASIC_AUTH_ACTIVE` | `true` |
| `N8N_BASIC_AUTH_USER` | `<your-username>` |
| `N8N_BASIC_AUTH_PASSWORD` | `<strong-password>` |
| `N8N_HOST` | `${RENDER_EXTERNAL_HOSTNAME}` |
| `N8N_PORT` | `5678` |
| `N8N_PROTOCOL` | `https` |
| `MULTIAGENT_API_BASE_URL` | `https://<your-multiagent-api>` |

> Important: if n8n runs on Render, do not use `http://localhost:8000` for `MULTIAGENT_API_BASE_URL`. Use the public URL of the multiagent API service.

> ⚠️ **Never commit secrets.** Always set credentials through Render's environment variable UI.
>
> ℹ️ `N8N_ENCRYPTION_KEY` is generated automatically by `render.yaml`. Do not rotate it unless you are intentionally re-encrypting credentials.

### 3 — Persistence (Postgres by default, Disk optional)

n8n stores workflows, credentials, and settings under `/home/node/.n8n`.  
In this template, core n8n data is persisted in **Render Postgres** (defined in `render.yaml`).

Add a persistent disk only if your workflows need local filesystem persistence (for example, file-based temp/output patterns).

1. In the Web Service, go to **Disks** → **Add Disk**.
2. Set:
   - **Mount Path**: `/home/node/.n8n`
   - **Size**: 1 GB (minimum; adjust as needed)

> ℹ️ Persistent disks are not available on Render's free plan. If you stay on free tier, rely on Postgres for core n8n persistence.

### 4 — Deploy

Click **Deploy**. Render will apply `render.yaml`, provision or update the defined resources, and start the n8n service.  
Once healthy, access n8n at the URL shown in the Render dashboard.

### 5 — Import Starter Workflows

This repository includes bootstrap workflows in `workflows/`.

1. Open n8n UI.
2. Go to **Workflows**.
3. Use **Import from File** and import:
   - `workflows/whatsapp.json`
   - `workflows/planner.json`
   - `workflows/developer.json`
   - `workflows/mcp.json`
4. Save and activate each workflow.

For workflow-level setup and test payloads, see `workflows/README.md`.

---

## Security Best Practices

- **Always enable Basic Auth** (`N8N_BASIC_AUTH_ACTIVE=true`) before exposing n8n publicly.
- **Never commit secrets** — use Render's environment variable UI or a secrets manager.
- **Use Postgres as the primary persistence layer**; do not rely on the container filesystem.
- **Rotate credentials regularly** and use strong, unique passwords.
- Consider restricting access by IP using Render's built-in network policies if available on your plan.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Service fails healthcheck | n8n not yet ready or startup dependency issue | Check service logs, verify env vars, and confirm Postgres connectivity |
| `Cannot GET /` after deploy | Wrong port mapping | Confirm `N8N_PORT=5678` and that Render routes to port 5678 |
| Data lost after redeploy | Postgres not attached/misconfigured | Verify `DB_*` variables from `render.yaml` and database connectivity |
| 401 Unauthorized | Basic auth active but credentials wrong | Check `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` env vars |
| Render free tier spins down | Inactivity sleep (free tier) | Upgrade to Starter plan or use an external cron to keep the service alive |

---

## Repository Structure

```
n8n/
├── render.yaml      # Render IaC service definition
├── workflows/       # Starter workflow JSON files
├── .gitignore       # Files excluded from version control
└── README.md        # This file
```

---

## License

MIT

