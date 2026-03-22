---
date: 2026-02-09T14:29:44-08:00
researcher: claude
git_commit: c100f4c
branch: main
repository: YouLab
topic: "VPS Deployment of YouLab + YouLearn to theyoulab.org"
tags: [deployment, infrastructure, docker, cloudflare, vps, production]
status: in_progress
last_updated: 2026-02-09
last_updated_by: claude
type: implementation_strategy
---

# Handoff: VPS Deployment — theyoulab.org

## Task(s)

| Task | Status |
|------|--------|
| VPS base setup (Docker, clone repo) | Completed |
| Deploy stack (Ralph + Dolt + OpenWebUI) | Completed |
| Cloudflare Tunnel for theyoulab.org | Completed |
| Configure OpenWebUI admin + Ralph pipe | Completed |
| Add YouLearn backend service to compose | Completed |
| Install YouLearn pipe in OpenWebUI | Completed |
| Fix YouLearn PDF links (public URL) | Completed — code pushed, needs VPS pull + restart verification |
| Install cloudflared as systemd service | In Progress — command given to user, not confirmed |
| Harden VPS (UFW + fail2ban) | Not Started |
| Disable OpenWebUI signups | Not Started — signups were enabled for admin account creation, need to flip back |

Plan document: `thoughts/shared/plans/2026-02-09-vps-deployment.md`

## Critical References

- `thoughts/shared/plans/2026-02-09-vps-deployment.md` — the deployment plan (5 phases)
- `thoughts/shared/research/2026-01-28-vps-deployment-theyoulab-org.md` — VPS research doc with architecture details
- `docker-compose.prod.yml` — production compose with all 4 services

## Recent changes

- `docker-compose.prod.yml` — Added YouLearn service, fixed env var loading via `env_file`, set `YOULEARN_BACKEND_URL=https://learn.theyoulab.org`
- `.env.production.example` — Added YouLearn env vars section, changed default model to `deepseek/deepseek-v3.2`
- `Dockerfile:17` — Fixed to copy `README.md` alongside `pyproject.toml` (hatchling build requirement)
- `Dockerfile:16-22` — Reordered to copy source before `uv pip install .` (source needed at build time)
- `README.md` — Created minimal placeholder (was missing, required by pyproject.toml)

Commits on main (this session):
- `c100f4c` fix: Use public URL for YouLearn backend PDF links
- `e199f92` fix: Fix YouLearn env vars and default to deepseek-v3.2
- `5c1b419` feat: Add YouLearn backend service to production compose
- `7444682` fix: Copy source before pip install in Dockerfile
- `889f012` fix: Add README.md and fix Dockerfile to include it in build
- `4d16c1f` feat: Production deployment, workspace API, sync infrastructure, and tooling (merge of ralph/ralph-wiggum-mvp → main)

## Learnings

1. **Docker Compose `env_file` vs `environment`**: `env_file` loads vars into the container at runtime, but `${VAR}` interpolation in compose files reads from `.env` (root) or shell — NOT from `env_file`. The `environment:` section overrides `env_file` values for the same key. Solution: use `env_file` for all services and avoid `${VAR}` interpolation for vars only defined in `.env.production`.

2. **Hatchling build requires README.md**: If `pyproject.toml` has `readme = "README.md"`, the Dockerfile must COPY it into the build context before `uv pip install .`.

3. **Dockerfile source ordering**: `uv pip install .` with hatchling needs the actual source code present (not just `pyproject.toml`), since hatchling builds from source. Must copy `src/` before the install step.

4. **Docker container networking**: Pipes running inside OpenWebUI containers should use container names (e.g., `http://youlab-ralph:8200`) not `host.docker.internal` when services are on the same Docker network.

5. **Cloudflare Tunnel**: Can't create DNS routes if A/CNAME records already exist for that hostname — must delete existing records first in Cloudflare dashboard.

6. **VPS specs**: RackNerd San Jose, 8GB RAM, 144GB disk, Ubuntu 24.04. IP: `23.94.218.158`.

7. **Cloudflare Tunnel UUID**: `04cdbcb6-0830-428c-bdda-6a8d823c10ff`, credentials at `/root/.cloudflared/04cdbcb6-0830-428c-bdda-6a8d823c10ff.json`.

## Artifacts

- `thoughts/shared/plans/2026-02-09-vps-deployment.md` — deployment plan
- `docker-compose.prod.yml` — production compose (4 services: openwebui, ralph, youlearn, dolt)
- `.env.production.example` — env template with all required vars
- `Dockerfile` — Ralph server Dockerfile (fixed)
- YouLearn `demo-prod` branch pushed to `github.com/ariavasulin/YouLearn`

## Action Items & Next Steps

1. **Verify YouLearn PDF links work** — User needs to pull latest on VPS (`git checkout -- docker-compose.prod.yml && git pull origin main`), restart YouLearn (`docker compose -f docker-compose.prod.yml up -d youlearn`), and test that PDF links now use `https://learn.theyoulab.org/pdf/...`

2. **Confirm cloudflared systemd service** — User was given the command (`sudo cloudflared service install && sudo systemctl enable cloudflared && sudo systemctl start cloudflared`). Verify with `sudo systemctl status cloudflared`. If already running manually, stop the manual process first (Ctrl+C) before installing the service.

3. **Disable OpenWebUI signups** — On VPS: `sed -i 's/ENABLE_SIGNUP=true/ENABLE_SIGNUP=false/' docker-compose.prod.yml && docker compose -f docker-compose.prod.yml up -d openwebui`. NOTE: the compose file on VPS may already have `ENABLE_SIGNUP=false` if the git pull reset it — check before running.

4. **Harden VPS** — Run: `sudo ufw default deny incoming && sudo ufw default allow outgoing && sudo ufw allow ssh && sudo ufw enable && sudo apt install -y fail2ban && sudo systemctl enable fail2ban`

5. **Verify end-to-end** — Test both pipes work via `https://theyoulab.org`: Ralph (send a chat message) and YouLearn (ask "show me my youbook" and verify PDF link loads)

## Other Notes

- **VPS SSH**: `ssh root@23.94.218.158` (user has Termius configured)
- **Repo on VPS**: `/opt/YouLab`
- **VPS `.env.production`**: Already has Ralph and YouLearn env vars set. Model is `deepseek/deepseek-v3.2`.
- **Cloudflare tunnel config**: `/root/.cloudflared/config.yml` routes `theyoulab.org` → `localhost:3000` and `learn.theyoulab.org` → `localhost:10000`
- **Ralph pipe valve**: Set to `http://youlab-ralph:8200` in OpenWebUI admin
- **YouLearn pipe valve**: Set to `http://youlab-youlearn:10000` in OpenWebUI admin
- **YouLearn branch**: The compose builds from `demo-prod` branch of `ariavasulin/YouLearn` — this branch has NOT been merged to main yet
- **OpenWebUI fork**: Built from `main` branch of `ariavasulin/open-webui` — contains custom YouLab branding, memory blocks UI, You page
- **Launched background session**: A security/performance review of the OpenWebUI fork was launched via `humanlayer launch` (session ID: `35dc76aa-1fbb-4b4b-ba0b-74f85275267b`) — check CodeLayer for results
