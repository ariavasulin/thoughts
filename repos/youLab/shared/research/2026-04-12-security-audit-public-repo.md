---
date: 2026-04-13T03:11:24Z
researcher: ARI
git_commit: b4c56efaf052b516310b9a047b9443bb5ed8b6be
branch: main
repository: YouLab
topic: "Security audit: secrets, data leaks, and vulnerabilities in public repo"
tags: [research, security, audit, secrets, pii, data-leak]
status: complete
last_updated: 2026-04-12
last_updated_by: ARI
---

# Research: Security Audit — Secrets, Data Leaks, and Vulnerabilities

**Date**: 2026-04-13T03:11:24Z
**Researcher**: ARI
**Git Commit**: b4c56efaf052b516310b9a047b9443bb5ed8b6be
**Branch**: main
**Repository**: YouLab

## Research Question
Are there any egregious security concerns or data leaks in this public repo?

## Summary

**No real secrets were found committed to git history.** The `.env` file containing live API keys is properly `.gitignore`d and was never committed. However, there are several findings that warrant attention, primarily around the **untracked `tmp/` directory** being at risk of accidental commit, **production infrastructure details** hardcoded in committed files, and **weak default credentials** in production compose files.

## Detailed Findings

### HIGH — `tmp/` Directory Not Gitignored (Risk of Accidental Commit)

The `tmp/` directory is **not in `.gitignore`** and shows as untracked in `git status`. It currently contains:

- `tmp/openwebui-data/webui.db` — A SQLite database that may contain user accounts, chat history, and API tokens
- `tmp/openwebui-data/uploads/` — User-uploaded files from a real session
- `tmp/openwebui-data/vector_db/` — Vector database shards
- `tmp/docker-compose.terminal-test.yml` — Contains hardcoded local filesystem path `/Users/ariasulin/Git/brain` (line 31)
- `tmp/pepr-ai-study-guide.md` — A personal interview prep document

**Risk:** A careless `git add .` or `git add -A` would commit all of this to the public repo, including the SQLite database with potential user data.

**Fix:** Add `tmp/` to `.gitignore`.

### HIGH — Real User UUID Hardcoded in Committed Docs

**File:** `docs/dev-testing.md:52,63`

```python
'ed6d1437-7b38-47a4-bd49-670267f0a7ce'
USER_ID="ed6d1437-7b38-47a4-bd49-670267f0a7ce"
```

This appears to be a real OpenWebUI user ID from a development or production instance, embedded in testing documentation. While UUIDs alone aren't directly exploitable, they could be used in API calls against the production instance if combined with other information.

### MEDIUM — Production Infrastructure Details in Committed Files

Production URLs and deployment details are hardcoded in committed source:

| Detail | File | Line |
|--------|------|------|
| `https://api.theyoulab.org` | `docker-compose.prod.yml` | 22 |
| `https://learn.theyoulab.org` | `docker-compose.prod.yml` | 78 |
| `https://theyoulab.org` | `src/ralph/server.py` | 153 |
| `https://openwebui.theyoulab.org` | `src/ralph/config.py` | 41 (comment) |
| `github.com/ariavasulin/open-webui.git#main` | `docker-compose.prod.yml` | 19 |
| `github.com/ariavasulin/YouLearn.git#demo-prod` | `docker-compose.prod.yml` | 70 |

This reveals the full production topology: domain names, personal GitHub forks, and branch names. Common for personal projects but worth noting for a public repo.

### MEDIUM — Weak Default Credentials in Production Compose

**File:** `docker-compose.prod.yml`

- **Line 94:** `DOLT_ROOT_PASSWORD: ${DOLT_ROOT_PASSWORD:-devpassword}` — If the env var is unset, the database gets `devpassword`
- **Line 31:** `WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:-please-change-me-in-production}` — If unset, JWT sessions can be forged with this known string

The same `devpassword` default is in `src/ralph/config.py:37` and `docker-compose.yml:12`.

**Fix:** Use `${VAR:?error message}` syntax in the prod compose file to fail loudly instead of silently using weak defaults.

### MEDIUM — `openwebui-content-sync/` Not Gitignored

This directory is also untracked (`??` in git status) and not in `.gitignore`. While its contents appear to be an open-source tool with only placeholder credentials, it should either be committed intentionally or added to `.gitignore`.

### LOW — Kubernetes Secret Manifest with Base64 Placeholders

**File:** `openwebui-content-sync/k8s/secrets.yaml`

Contains base64-encoded values that decode to placeholder strings (`your-confluence-api-key`, `your-slack-token`). Not real credentials, but the pattern of committing Kubernetes Secret manifests (even with placeholders) is a common anti-pattern that can lead to real secrets being committed later.

### LOW — Server Binds to 0.0.0.0

**File:** `src/ralph/server.py:427`

```python
uvicorn.run(app, host="0.0.0.0", port=8200)
```

Standard for Docker containers, but if run outside Docker, the server is accessible on all network interfaces. The production compose file correctly restricts this to `127.0.0.1:8200`.

## What Was NOT Found (Good Signs)

- **No real secrets in git history** — Searched for API keys, passwords, tokens, AWS credentials, SSH keys; none found
- **`.env` never committed** — Confirmed with `git log --all --name-only -- '.env'` returning no results
- **No AWS/GCP/Azure credentials**
- **No SSH keys or certificates**
- **No real Slack/GitHub tokens**
- **`.env.example` files use proper placeholders** (`sk-or-v1-your-key-here`, `your-secure-password-here`)
- **CORS is appropriately configured** for production
- **Production ports bound to localhost** in `docker-compose.prod.yml`
- **Container runs as non-root** (`useradd -m -u 1000 ralph` in Dockerfile)

## Recommended Actions (Priority Order)

1. **Add `tmp/` to `.gitignore`** — Prevents accidental commit of SQLite database, personal files, and local paths
2. **Remove or replace the real UUID in `docs/dev-testing.md`** — Use a clearly fake example UUID
3. **Use `${VAR:?must be set}` in `docker-compose.prod.yml`** for `DOLT_ROOT_PASSWORD` and `WEBUI_SECRET_KEY` — Fail loudly instead of using weak defaults
4. **Decide on `openwebui-content-sync/`** — Either commit it intentionally or add to `.gitignore`

## Code References

- `src/ralph/config.py:37` — Hardcoded `devpassword` default
- `src/ralph/config.py:41` — Production OpenWebUI URL in comment
- `src/ralph/server.py:149-154` — CORS allowlist with production domain
- `src/ralph/server.py:427` — Server binds to `0.0.0.0`
- `docker-compose.prod.yml:19,70` — Personal GitHub fork URLs
- `docker-compose.prod.yml:22,78` — Production API URLs
- `docker-compose.prod.yml:31` — Weak default `WEBUI_SECRET_KEY`
- `docker-compose.prod.yml:94` — Weak default `DOLT_ROOT_PASSWORD`
- `docs/dev-testing.md:52,63` — Real user UUID
- `tmp/docker-compose.terminal-test.yml:31` — Local filesystem path leak
- `openwebui-content-sync/k8s/secrets.yaml:11,24,37,50` — Placeholder base64 secrets

## Open Questions

- Is the UUID in `docs/dev-testing.md` from a production instance or a disposable dev environment?
- Should `openwebui-content-sync/` be its own repo or a committed subdirectory?
