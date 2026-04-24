# STATUS — open-webui
*Updated: 2026-04-23*

## What this is
Open WebUI is a persistent ChatGPT-style web interface running as a Docker container on
`docker-core` (LXC 1110, 10.0.100.18:3000). It connects to local LLM backends via
OpenAI-compatible API — normally Ollama on PC1, or the T2-03 llama-server when that
cluster is active.

## Where it stands
Deployed and running. M0 (initial deployment) and M1 (documentation) both complete.

## Last session
2026-04-23 — Audited live container state (up 3 days, healthy, restart: unless-stopped).
Created project repo and full ops README. Pushed to GitHub.

## Next action
No active work. Open if needed: update backend URL or follow a backlog item in ROADMAP.md.

## Watch out for
- Restart policy is `unless-stopped` — if you ever manually `docker stop` it, Docker will
  NOT auto-restart it on next boot. Run `docker start open-webui` manually.
- `open-webui-data` volume holds all chat history and user settings. Never run
  `docker system prune --volumes` on docker-core.
- Image tag is `:main` (rolling). A bad upstream release will break the container on next
  pull. See ROADMAP B-02 if this becomes a problem.
