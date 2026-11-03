# Open WebUI
*Updated: 2026-11-03*

Persistent ChatGPT-style web interface running on `docker-core` (LXC 1110, 10.0.100.18).
Connects to local LLM backends — normally Ollama on PC1, or the T2-03 distributed
llama-server when that cluster is running. Access at `http://10.0.100.18:3000`.

---

## Quick Start

```bash
# Open in browser
http://10.0.100.18:3000

# Check if the container is running
ssh root@10.0.100.18 "docker ps --filter name=open-webui"

# Start if stopped
ssh root@10.0.100.18 "docker start open-webui"

# Restart (picks up no config change — image is stateless)
ssh root@10.0.100.18 "docker restart open-webui"
```

Container has `restart: unless-stopped` — it comes back automatically when docker-core
reboots. You should only ever need `docker start open-webui` after a manual `docker stop`.

---

## Architecture

```
Browser
  └── http://10.0.100.18:3000
        │
        ▼
docker-core (LXC 1110, 10.0.100.18)
  └── open-webui container (:3000 → internal :8080)
        │  volume: open-webui-data (users, chat history, settings)
        │
        ▼ OpenAI-compatible API
  ┌─────────────────────────────────────────────────┐
  │ Default: Ollama on PC1 (10.0.100.100:11434)     │
  │ Alt:     T2-03 llama-server  (:8080/v1)         │
  └─────────────────────────────────────────────────┘
```

Open WebUI is backend-agnostic. It talks OpenAI-compatible API. Switch the backend URL
in **Settings → Connections** without restarting the container.

---

## Components

### Container — `open-webui`

| Property | Value |
|---|---|
| Image | `ghcr.io/open-webui/open-webui:main` |
| Host port | `3000` |
| Internal port | `8080` |
| Restart policy | `unless-stopped` |
| Data volume | `open-webui-data` → `/app/backend/data` |
| Host | docker-core (`10.0.100.18`) |

### Backends

| Backend | URL | When to use |
|---|---|---|
| Ollama on PC1 | `http://10.0.100.100:11434` | Default — Ollama is always running on PC1 |
| T2-03 llama-server | `http://10.0.100.100:8080/v1` | When distributed cluster is up (see t2-03 project) |

### Data Volume

`open-webui-data` holds all persistent state: user accounts, chat history, saved prompts,
model settings. It survives container updates. Never delete this volume.

---

## Reference Tables

### Environment Variables (active)

| Variable | Value | Purpose |
|---|---|---|
| `OLLAMA_BASE_URL` | `http://10.0.100.100:11434` | Ollama backend |
| `OPENAI_API_BASE_URL` | `http://10.0.100.100:11434/v1` | OpenAI-compat layer for same |
| `OPENAI_API_KEY` | `none` | Required field, no auth on local backends |
| `ENABLE_OLLAMA_API` | `true` | Exposes Ollama model list in UI |
| `PORT` | `8080` | Internal container port |

### Ports

| Port | Host | Purpose |
|---|---|---|
| 3000 | docker-core | External access (browser) |
| 8080 | container | Internal web server |

---

## Mini-Tutorials

### 1. Switch to the T2-03 Distributed Cluster

**Situation:** T2-03 cluster is running (llama-server on PC1:8080) and you want Open WebUI
to use it instead of Ollama.

1. Make sure the cluster is up: `bash /opt/projects/t2-03-distributed-inference/scripts/health-check.sh`
2. Open `http://10.0.100.18:3000` → top-right avatar → **Settings**
3. Go to **Connections → OpenAI API**
4. Change Base URL to `http://10.0.100.100:8080/v1`, API Key: `none`, Save
5. New chat → model selector should show `nemotron-3-nano-30b` (or whatever is loaded)

**Tip:** The change is per-user in Open WebUI. It persists in `open-webui-data` — no need
to repeat it next session. Switch back to Ollama the same way when the cluster is stopped.

---

### 2. Update the Container to Latest Image

**Situation:** Open WebUI has released new features and you want to pull the latest build.

```bash
# Pull latest image
ssh root@10.0.100.18 "docker pull ghcr.io/open-webui/open-webui:main"

# Stop current container
ssh root@10.0.100.18 "docker stop open-webui"

# Remove old container (volume is safe — it's named, not inline)
ssh root@10.0.100.18 "docker rm open-webui"

# Recreate with same config
ssh root@10.0.100.18 "docker run -d \
  --name open-webui \
  --restart unless-stopped \
  -p 3000:8080 \
  -v open-webui-data:/app/backend/data \
  -e OLLAMA_BASE_URL=http://10.0.100.100:11434 \
  -e OPENAI_API_BASE_URL=http://10.0.100.100:11434/v1 \
  -e OPENAI_API_KEY=none \
  -e ENABLE_OLLAMA_API=true \
  ghcr.io/open-webui/open-webui:main"

# Verify
ssh root@10.0.100.18 "docker ps --filter name=open-webui"
```

**Tip:** Chat history and user settings are in the volume — they survive the `rm`/`run` cycle.

---

## Decision Tree

```
Can't reach http://10.0.100.18:3000 ?
│
├── docker-core LXC stopped?
│     └── Start LXC 1110 in Proxmox → Docker autostart brings container up
│
├── Container stopped?
│     └── ssh root@10.0.100.18 "docker start open-webui"
│
├── Container running but no models showing?
│     ├── Ollama backend: is Ollama running on PC1? (check PC1 tray icon)
│     └── llama-server backend: run health-check.sh in t2-03 project
│
└── Want to switch backend?
      └── Settings → Connections → change OpenAI Base URL (no restart needed)
```

---

## Troubleshooting

### Container not running after docker-core reboot

The restart policy is `unless-stopped`, not `always`. If you previously ran `docker stop open-webui`
manually, Docker will NOT restart it on next boot. Fix:

```bash
ssh root@10.0.100.18 "docker start open-webui"
# To make it restart on boot from now on:
ssh root@10.0.100.18 "docker update --restart always open-webui"
```

### No models in the model selector

- **Ollama path:** Ollama must be running on PC1 (`10.0.100.100:11434`). Check the Ollama
  tray icon on PC1, or `curl http://10.0.100.100:11434/api/tags` from gemini-ops.
- **llama-server path:** Cluster must be started first. See t2-03 project Quick Start.

### Chat history or settings lost

The `open-webui-data` volume was deleted. This is unrecoverable. To check:

```bash
ssh root@10.0.100.18 "docker volume ls | grep open-webui"
```

Avoid `docker system prune -a` on docker-core — it deletes named volumes with `--volumes`.

### Container in a crash loop

```bash
ssh root@10.0.100.18 "docker logs open-webui --tail 50"
```

Most common cause: upstream image regression. Pin to a known-good digest or roll back
to a previous image tag.

---

## File Map

| Path | Purpose | Edit when |
|---|---|---|
| `README.md` | This file — ops reference | Container config changes, new backend added |
| `ROADMAP.md` | Backlog and improvement ideas | Accepting a new improvement |
| `LESSONS.md` | Non-obvious issues resolved | After debugging something unexpected |
| `STATUS.md` | Current state snapshot | After any significant change |
