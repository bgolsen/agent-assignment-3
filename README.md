# Assignment 3: n8n for Visual Workflow Intelligence

**UCLA Extension - Agentic AI Course**

---

## Overview

Week 3 pairs planning/reasoning theory with **n8n**, a visual workflow tool you'll use to wire together LLMs, triggers, and APIs without writing a full agent framework.

This starter spins up a self-hosted n8n instance on your machine using Docker. No cloud account, no Postgres — just SQLite and a Docker volume.

| Component | Choice |
|-----------|--------|
| Runtime | Docker (single container via Compose) |
| Database | SQLite (n8n default) |
| Persistence | Named Docker volume `n8n_data` |
| Port | `5678` |
| Image | `docker.n8n.io/n8nio/n8n:latest` |

---

## Prerequisites

1. **Docker Desktop** — install from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) (Mac/Windows) or use Docker Engine on Linux. Make sure it's running before the next step.
2. `openssl` — preinstalled on macOS/Linux. On Windows use Git Bash or WSL.

Verify:

```bash
docker --version
docker compose version
```

---

## Getting Started

### 1. Configure environment

```bash
cp .env.example .env
```

Then generate an encryption key and paste it into `.env` as `N8N_ENCRYPTION_KEY`:

```bash
openssl rand -hex 32
```

> ⚠️ Keep this key stable. If it changes, every credential you've saved in n8n becomes unreadable.

Optionally add `OPENAI_API_KEY` and/or `ANTHROPIC_API_KEY` so the AI nodes work out of the box.

### 2. Start n8n

```bash
docker compose up -d
```

First run pulls the image (~400 MB) and may take a minute.

### 3. Open the editor

Visit **[http://localhost:5678](http://localhost:5678)**.

On first visit n8n asks you to create a local owner account (email + password). This account is stored inside the container's SQLite DB — it's separate from anything online.

### 4. Open the sample workflow

Any JSON file in `workflows/` is auto-imported on first container boot. The starter ships with `workflows/topic-planner.json` — a workflow that takes a topic and asks Claude to produce a 3-step research plan.

After signing in, open the **Topic Planner (Claude)** workflow from the sidebar. To run it you need an Anthropic key. Two paths:

**A. Header Auth credential (recommended)**

Create the credential:
1. Left sidebar → **Credentials** → **+ Add Credential** (top right).
2. Search for and pick **Header Auth** (under "Generic Credential Type").
3. Fill in:
   - **Credential Name:** `Anthropic API Key`
   - **Name:** `x-api-key`
   - **Value:** `sk-ant-...your-real-key...`
4. **Save**.

Wire it into the workflow:
1. Open the **Topic Planner (Claude)** workflow → click the **Anthropic Messages** node.
2. **Authentication** dropdown → switch from *None* to **Generic Credential Type**.
3. **Generic Auth Type** → **Header Auth**.
4. **Credential for Header Auth** → select `Anthropic API Key`.
5. Scroll to **Header Parameters** and **delete the `x-api-key` row** (the credential supplies it; leaving the row will overwrite it with an empty value). Keep the `anthropic-version` and `content-type` rows.
6. **Save**, then click **Test workflow**. The **Extract Plan** node's `plan` field is the model's numbered list.

The credential lives in the SQLite DB inside `n8n_data`, encrypted with your `N8N_ENCRYPTION_KEY` — it survives container restarts and isn't in `.env`.

**B. Quick env var (less portable)**

Skip the credential — put the key in `.env` and the workflow uses it as-shipped:

```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." >> .env
docker compose up -d            # re-applies env to the container
```

The workflow's `{{ $env.ANTHROPIC_API_KEY }}` expression resolves it. Env access in expressions is enabled in this compose file (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) — fine for a single-user local dev box, not what you'd want on a shared host.

---

To re-import a workflow after editing the JSON: `docker exec n8n rm /home/node/.n8n/.imported_<filename>` and restart.

---

## Where Your Data Lives

Everything n8n writes — workflows, credentials, execution history, the SQLite DB — lives inside the named Docker volume `n8n_data`, mounted at `/home/node/.n8n` in the container.

Inspect it:

```bash
docker volume inspect n8n_data
```

Because it's a named volume (not a bind mount), it survives container rebuilds and image upgrades.

---

## Common Commands

| Action | Command |
|--------|---------|
| Start in background | `docker compose up -d` |
| Stop (keep data) | `docker compose down` |
| View logs | `docker compose logs -f n8n` |
| Restart | `docker compose restart n8n` |
| Pull a newer n8n image | `docker compose pull && docker compose up -d` |
| **Reset everything (deletes workflows!)** | `docker compose down -v` |

`-v` on `down` removes the named volume — use it only when you want a clean slate.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `port is already allocated` | Something else is on 5678. Stop it, or change the host side: `"5679:5678"` in `docker-compose.yml`. |
| Editor loads but shows "encryption key changed" warning | Your `N8N_ENCRYPTION_KEY` doesn't match what was used when credentials were saved. Restore the original key, or delete credentials and re-enter them. |
| AI nodes can't see your API key | Confirm the key is in `.env`, then `docker compose up -d` to re-apply env vars (a plain `restart` won't reload `.env`). |
| `Cannot connect to the Docker daemon` | Docker Desktop isn't running — open it and wait for the whale icon. |

---

## What NOT to Commit

`.env` is gitignored. It holds your encryption key and API keys — keep it local. If you accidentally commit it, rotate the keys.

---

## Resources

- [n8n docs](https://docs.n8n.io/)
- [n8n self-hosting reference](https://docs.n8n.io/hosting/)
- [Workflow templates](https://n8n.io/workflows/)
- [AI / LangChain nodes](https://docs.n8n.io/advanced-ai/)
