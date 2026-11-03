# ROADMAP — open-webui
*Updated: 2026-11-03*

## Milestones

| # | Milestone | Goal | Status |
|---|---|---|---|
| M0 | Initial deployment | Container running on docker-core, Ollama backend connected | ✅ Done |
| M1 | Documentation | README with ops runbook on GitHub | ✅ Done (2026-04-23) |

## Backlog

| ID | Item | Decision | Date |
|---|---|---|---|
| B-01 | Cloudflare tunnel for external access | Not decided — evaluate when needed | 2026-04-23 |
| B-02 | Pin to a versioned image tag instead of `:main` | Good idea for stability; deferred until `:main` causes a regression | 2026-04-23 |
| B-03 | docker-compose file for reproducible redeploy | Useful; low priority since container rarely needs to be recreated | 2026-04-23 |
