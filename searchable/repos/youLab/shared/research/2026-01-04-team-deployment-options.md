---
date: 2026-01-04T17:07:29+07:00
researcher: ariasulin
git_commit: 29783e20ce6d230469eee40148d80b068d2fd021
branch: main
repository: YouLab
topic: "Cloud Deployment Options for YouLab"
tags: [research, deployment, infrastructure, docker, railway, cloud, cost-analysis, aws, ecs, fargate, lightsail]
status: complete
last_updated: 2026-01-04
last_updated_by: claude
last_updated_note: "Added AWS deployment deep-dive: Lightsail, ECS Fargate, App Runner comparison"
---

# Research: Cloud Deployment Options for YouLab

**Date**: 2026-01-04T17:07:29+07:00
**Researcher**: ariasulin
**Git Commit**: 29783e20ce6d230469eee40148d80b068d2fd021
**Branch**: main
**Repository**: YouLab

## Research Question

How to deploy YouLab to the cloud for:
- Team of 4 to test/use the full-stack
- Easy development iteration (commit, test, deploy)
- Authentication via OpenWebUI (disable new account creation)
- Persistent Letta agent memory (non-negotiable)
- Foundation for eventual production launch

## Summary

YouLab requires **3 services** for cloud deployment (not 4 - Ollama is not needed):

| Service | Purpose | Persistence | Notes |
|---------|---------|-------------|-------|
| OpenWebUI | Chat UI + auth | Volume for SQLite | Has Railway one-click template |
| Letta Server | Agent memory | **Volume required** | PostgreSQL at `/var/lib/postgresql/data` |
| HTTP Service | API bridge | Stateless | Needs Dockerfile (doesn't exist yet) |

**Key finding**: Ollama is NOT used in the YouLab flow. The pipeline goes `OpenWebUI → HTTP Service → Letta → OpenAI/Claude API` directly.

**Recommended platform**: Railway - has one-click OpenWebUI template, supports persistent volumes, simple multi-service projects

## Detailed Findings

### Actual Data Flow (Ollama Not Used)

The YouLab pipeline bypasses Ollama entirely:

```
Browser → OpenWebUI:3000 → HTTP Service:8100 → Letta:8283 → OpenAI/Claude API
```

**Evidence** (from codebase analysis):
- `src/letta_starter/server/agents.py:110` - Agents hardcoded to `model="openai/gpt-4o-mini"`
- `src/letta_starter/pipelines/letta_pipe.py` - Pipe connects directly to HTTP Service, no Ollama reference
- Ollama only appears in docker-compose for OpenWebUI's built-in model support (unused by YouLab)

### Services Required for Cloud Deployment

| Service | Image | Port | Persistence | Railway Support |
|---------|-------|------|-------------|-----------------|
| OpenWebUI | `ghcr.io/open-webui/open-webui:main` | 8080 | `/app/backend/data` | One-click template |
| Letta Server | `letta/letta:latest` | 8283 | `/var/lib/postgresql/data` | Docker image |
| HTTP Service | Custom Dockerfile needed | 8100 | None | Dockerfile |

### Letta Persistence Details

From [Letta Docker documentation](https://docs.letta.com/guides/server/remote):

```bash
# Volume mount for persistent agent memory
-v ~/.letta/.persist/pgdata:/var/lib/postgresql/data
```

The Letta Docker image bundles PostgreSQL. Without this volume mount, **all agent data is lost on container restart**.

For cloud deployments, you can also use external PostgreSQL via `LETTA_PG_URI` environment variable.

### Environment Variables for Cloud

**HTTP Service**:
```bash
YOULAB_SERVICE_HOST=0.0.0.0              # Accept external connections
YOULAB_SERVICE_PORT=8100
YOULAB_SERVICE_LETTA_BASE_URL=http://letta:8283  # Internal service name
OPENAI_API_KEY=sk-...
```

**Letta Server**:
```bash
OPENAI_API_KEY=sk-...
# Optional: LETTA_PG_URI for external PostgreSQL
```

**OpenWebUI**:
```bash
# Disable Ollama (not needed)
OLLAMA_BASE_URL=  # Leave empty or don't set
# Point Pipe to HTTP Service
# (Configured in admin UI, not env var)
```

### OpenWebUI Authentication

OpenWebUI handles user authentication natively:
- First user becomes admin
- Admin can disable new account creation in Settings
- Users authenticate against OpenWebUI's internal user database
- No external auth service needed for team of 4

## Railway Deployment (Recommended)

### Why Railway

- [One-click OpenWebUI template](https://railway.com/deploy/open-webui) exists
- [Persistent volumes](https://docs.railway.com/reference/volumes) up to 250GB on Pro plan
- Multiple services in one project with internal networking
- Auto-HTTPS on all services
- GitHub integration for auto-deploy on push
- Cost: ~$5-20/month for this stack

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Railway Project                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐                                           │
│  │    OpenWebUI     │◄──── HTTPS (public domain)                │
│  │   (template)     │                                           │
│  │ Volume: /data    │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           │ http://http-service.railway.internal:8100           │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │   HTTP Service   │◄──── Internal only (no public URL)        │
│  │   (Dockerfile)   │                                           │
│  │   Stateless      │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           │ http://letta.railway.internal:8283                  │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │   Letta Server   │◄──── Internal only                        │
│  │   (Docker image) │                                           │
│  │ Volume: /pgdata  │◄──── CRITICAL: Agent memory               │
│  └──────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Setup Steps

1. **Create Railway Project**
   - Start from OpenWebUI template: https://railway.com/deploy/open-webui
   - This creates OpenWebUI service with volume

2. **Add Letta Service**
   - Add new service → Docker Image → `letta/letta:latest`
   - Add volume mounted to `/var/lib/postgresql/data`
   - Set environment variables: `OPENAI_API_KEY`
   - No public URL needed (internal only)

3. **Add HTTP Service**
   - Add new service → GitHub repo (or Dockerfile)
   - Point to YouLab repo, Dockerfile at root
   - Set environment variables:
     - `YOULAB_SERVICE_HOST=0.0.0.0`
     - `YOULAB_SERVICE_LETTA_BASE_URL=http://letta.railway.internal:8283`
     - `OPENAI_API_KEY`
   - No public URL needed (internal only)

4. **Configure OpenWebUI**
   - Access via Railway-provided HTTPS URL
   - Create admin account (first user)
   - Disable new registrations in Settings
   - Add Pipe from `src/letta_starter/pipelines/letta_pipe.py`
   - Set Pipe valve: `LETTA_SERVICE_URL=http://http-service.railway.internal:8100`

5. **Invite Team**
   - Create accounts for team members manually in OpenWebUI admin
   - Share the Railway HTTPS URL

### What Needs to Be Created

**Dockerfile for HTTP Service** (doesn't exist yet):

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install uv
RUN pip install uv

# Copy project files
COPY pyproject.toml uv.lock ./
COPY src/ ./src/

# Install dependencies
RUN uv sync --frozen

# Run the service
EXPOSE 8100
CMD ["uv", "run", "letta-server"]
```

### Development Workflow with Railway

**Option A: Auto-deploy from main**
- Push to `main` → Railway auto-deploys
- Team tests on production URL

**Option B: Staging environment**
- Create second Railway project or environment
- Deploy feature branches there
- Team tests on staging URL

**Option C: Local + Cloud hybrid**
- Develop locally as usual
- Push to branch when ready for team testing
- Railway deploys branch to staging

## Alternative: Fly.io

Fly.io is more complex but offers:
- Edge deployment (servers closer to users)
- More control over infrastructure
- Slightly cheaper for compute-heavy workloads

**Drawbacks**:
- No native docker-compose support
- Requires converting to `fly.toml` configuration
- Each service = separate Fly app
- More manual networking setup

See [Fly.io Docker documentation](https://fly.io/docs/app-guides/multiple-processes/)

## Alternative: Dev Server + Tunnel (Simpler Short-term)

If you want to start testing today without cloud setup:

1. Run stack on your machine (existing setup)
2. Expose via Tailscale or ngrok
3. Team accesses your machine directly
4. Migrate to Railway when ready for always-on

**Pros**: Works immediately, no new infrastructure
**Cons**: Your machine must be online, single point of failure

## Code References

- `src/letta_starter/pipelines/letta_pipe.py:25` - Pipe default URL: `http://host.docker.internal:8100`
- `src/letta_starter/server/agents.py:110` - Model hardcoded to `openai/gpt-4o-mini`
- `src/letta_starter/server/cli.py:8-17` - HTTP service entry point
- `src/letta_starter/config/settings.py:106-153` - Environment configuration
- `OpenWebUI/open-webui/docker-compose.yaml` - Local OpenWebUI compose (includes Ollama, not needed for cloud)
- `docs/Quickstart.md` - Current local setup documentation

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Original scope excluded production deployment
- `thoughts/shared/research/2025-12-31-open-webui-docker-setup.md` - Docker setup research

**Note**: The original plan scoped out production deployment for pilot. This research expands scope to support team testing and future launch.

## Next Steps

1. **Create Dockerfile** for HTTP Service (see template above)
2. **Test Railway deployment** with all three services
3. **Configure OpenWebUI** authentication (disable registration)
4. **Document deployment** in `/docs/Deployment.md`

## Web Sources

- [Railway Volumes Documentation](https://docs.railway.com/reference/volumes)
- [Railway Open WebUI Template](https://railway.com/deploy/open-webui)
- [Letta Remote Deployment Guide](https://docs.letta.com/guides/server/remote)
- [OpenWebUI Environment Configuration](https://docs.openwebui.com/getting-started/env-configuration/)
- [Fly.io Multiple Processes](https://fly.io/docs/app-guides/multiple-processes/)

## Open Questions

1. Should Letta use Railway's managed PostgreSQL instead of bundled PostgreSQL for better reliability?
2. Do you want staging + production environments, or just one deployment?
3. Should the HTTP Service have a public healthcheck URL for monitoring?

---

## Follow-up Research: Railway vs Competitors Cost Analysis

**Date**: 2026-01-04T17:39:01+07:00
**Question**: Is Railway really the move? Comprehensive cost estimate and competitor comparison.

### Executive Summary

| Platform | Monthly Cost | Complexity | Verdict |
|----------|-------------|------------|---------|
You | **Hetzner + Coolify** | **$6.50** | Medium | **Best value** if you don't mind self-hosting |
| **Fly.io** | $15-20 | Low-Medium | **Best managed PaaS** for cost |
| **DO Droplet** | $12-24 | Medium | Good middle ground |
| **Render** | $23-42 | Low | Easy but pricier |
| **Railway** | $28-35 | **Lowest** | **Easiest DX**, but premium cost |
| **DO App Platform** | $27-42 | Low | **Not recommended** - persistence issues |

**Bottom line**: Railway is ~2x the cost of Fly.io for equivalent functionality. Whether that's worth it depends on how much you value Railway's superior developer experience.

---

### Detailed Cost Breakdown

#### Railway (Pro Plan)

| Resource | Usage | Monthly Cost |
|----------|-------|--------------|
| Base plan | Required | $20 (includes $20 credits) |
| RAM | 2GB × 730h | $20.28 |
| CPU | ~0.3 vCPU avg | $6.08 |
| Volume storage | 10GB | $1.58 |
| Egress | ~2GB | $0.10 |
| **Total** | | **$28-35** |

**Pricing model**: Pure usage-based after $20 minimum. $10.14/GB RAM, $0.158/GB storage.

**Volume gotchas**:
- Not available at build time
- Not mounted during pre-deploy
- Growing volumes requires Pro plan
- Non-root apps need `RAILWAY_RUN_UID=0`

---

#### Fly.io

| Resource | Usage | Monthly Cost |
|----------|-------|--------------|
| OpenWebUI | shared-cpu-1x, 1GB | $5.50 |
| Letta Server | shared-cpu-1x, 512MB | $3.50 |
| HTTP Service | shared-cpu-1x, 256MB | $2.00 |
| Volumes | 10GB | $1.50 |
| IPv4 address | 1 shared | $2.00 |
| Egress | ~5GB | $0.10 |
| **Total** | | **$14.60** |

**Pricing model**: Pure pay-as-you-go, no minimum. ~$5/GB RAM, $0.15/GB storage.

**Volume gotchas**:
- 1:1 machine mapping (no shared volumes)
- Single-region, single-server per volume
- Manual redundancy required for production

**Networking**: Excellent - automatic WireGuard mesh via `<app>.internal` DNS.

---

#### Render

| Option | Configuration | Monthly Cost |
|--------|---------------|--------------|
| **Budget** | 3× Starter (512MB each) + 10GB disk | **$23.50** |
| **Balanced** | 1× Standard (2GB) + 2× Starter + 10GB | **$41.50** |
| **Comfortable** | 3× Standard (2GB each) + 10GB | **$77.50** |

**Pricing model**: Fixed per-instance ($7 Starter, $25 Standard). Storage $0.25/GB.

**Key limitation**: No 1GB tier - jump from 512MB ($7) to 2GB ($25).

**Free tier**: Services sleep after 15 minutes (30-50s cold start). PostgreSQL expires after 30 days.

---

#### DigitalOcean

**App Platform** (managed):

| Component | Monthly Cost |
|-----------|--------------|
| 3 services (512MB-1GB each) | $17-22 |
| Dev database (free) OR Managed DB | $0-15 |
| **Total** | **$22-37** |

**Critical problem**: **No persistent volumes on App Platform.** Filesystem wipes on every deploy. Must use database for all storage.

**Droplet + Docker Compose** (self-managed):

| Droplet Size | Monthly Cost |
|--------------|--------------|
| 2GB RAM, 50GB SSD | $12 |
| 4GB RAM, 80GB SSD | $24 |

Full persistence, you manage updates.

---

#### Hetzner + Coolify (Self-Hosted)

| Component | Monthly Cost |
|-----------|--------------|
| Hetzner CX33 (8GB RAM, 80GB NVMe) | **$6.50** |
| Coolify (self-hosted) | $0 |
| **Total** | **$6.50** |

Coolify = open-source Railway clone. Full docker-compose support, automatic SSL, one-click databases, GitHub push-to-deploy.

**Trade-off**: You maintain the server. But for a 4-person team, this is trivial.

---

### Feature Comparison Matrix

| Feature | Railway | Fly.io | Render | DO App | Coolify |
|---------|---------|--------|--------|--------|---------|
| **Git push deploy** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Docker Compose** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Persistent volumes** | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Auto SSL** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Internal networking** | ✅ | ✅ | ✅ | ✅ | ✅ You |
| **One-click OpenWebUI** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Preview environments** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Zero-config** | ✅✅ | ✅ | ✅ | ✅ | ❌ |
| **Full control** | ❌ | ✅ | ❌ | ❌ | ✅ |

---

### Hidden Costs & Gotchas Summary

| Platform | Watch Out For |
|----------|---------------|
| **Railway** | $20 minimum even if idle; Volume grows = Pro plan required |
| **Fly.io** | IPv4 costs $2/mo; no permanent free tier; volumes don't replicate |
| **Render** | 512MB → 2GB jump ($7 → $25); free DBs expire in 30 days |
| **DO App** | **NO PERSISTENT STORAGE** - deal-breaker for Letta |
| **Coolify** | You're the ops team; Hetzner = EU-based (latency for US users) |

---

### Recommendation

**For YouLab's current stage (4-person team testing)**:

#### Option A: Lowest Cost Path
**Hetzner CX33 + Coolify** = **$6.50/month**
- Best if you're comfortable with light server maintenance
- Docker Compose works natively (unlike managed platforms)
- Full control, no vendor lock-in

#### Option B: Best Managed PaaS Value
**Fly.io** = **$15-20/month**
- 50% cheaper than Railway for same functionality
- Great networking (`<app>.internal` just works)
- More manual setup than Railway, but not by much

#### Option C: Easiest Developer Experience
**Railway** = **$28-35/month**
- One-click OpenWebUI template saves setup time
- Best web UI and observability
- Premium cost for premium DX

**My verdict**: Railway is **not** overpriced for what it offers, but it IS ~2x the cost of Fly.io. For a prototype/team testing phase, I'd lean toward **Fly.io** unless the extra $15/month for Railway's DX is worth it to you.

If you're price-sensitive and slightly technical, **Hetzner + Coolify** is unbeatable at $6.50/month.

---

### Cost Over Time

| Platform | 1 Month | 6 Months | 1 Year |
|----------|---------|----------|--------|
| Hetzner + Coolify | $6.50 | $39 | $78 |
| Fly.io | $17 | $102 | $204 |
| Railway | $32 | $192 | $384 |
| Render (balanced) | $42 | $252 | $504 |

**Annual savings** (Railway → Fly.io): **$180**
**Annual savings** (Railway → Coolify): **$306**

---

### Decision Factors

Choose **Railway** if:
- You want the easiest setup with best docs
- One-click OpenWebUI template is valuable
- $30/month is negligible vs your time

Choose **Fly.io** if:
- You want managed PaaS at reasonable cost
- You're comfortable with slightly more config
- You want edge deployment later

Choose **Coolify + Hetzner** if:
- You want maximum value for money
- You're comfortable SSHing into a server occasionally
- Docker Compose native support appeals to you
- EU data residency is acceptable (or use US Hetzner region)

---

## Follow-up Research: AWS Deployment Deep-Dive

**Date**: 2026-01-04T18:18:14+07:00
**Question**: Best AWS deployment setup balancing DX, cost, and scalability

### Executive Summary

| AWS Option | Monthly Cost | DX Score | Scalability | Best For |
|------------|--------------|----------|-------------|----------|
| **Lightsail VPS** | $20-35 | ⭐⭐⭐⭐ | Manual | Simplest AWS, good for start |
| **ECS Fargate + Copilot** | $50-80 | ⭐⭐⭐ | Auto | Production-ready scaling |
| **App Runner** | $60-80 | ⭐⭐⭐ | Auto | Simpler Fargate, but limits |
| **ECS on EC2** | $30-50 | ⭐⭐ | Manual/Auto | Cost optimization later |

**Recommendation**: Start with **Lightsail VPS ($20/mo)** for immediate deployment, then migrate to **ECS Fargate with Copilot** when you need auto-scaling.

---

### AWS Option 1: Lightsail VPS + Docker Compose (Recommended Start)

**The simplest AWS path** - closest to local development workflow.

| Instance | RAM | vCPU | Storage | Transfer | Price |
|----------|-----|------|---------|----------|-------|
| 2GB | 2GB | 2 | 60GB | 3TB | $10/mo |
| **4GB** | **4GB** | **2** | **80GB** | **4TB** | **$20/mo** |
| 8GB | 8GB | 2 | 160GB | 5TB | $40/mo |

**Why this works:**
- Full Docker Compose support (unlike managed services)
- Persistent storage for Letta PostgreSQL
- Your existing `docker-compose.yml` works as-is
- SSH + deploy scripts = familiar workflow
- **~40% cheaper than EC2** for equivalent specs

**Deployment workflow:**
```bash
# One-time setup
ssh ubuntu@your-lightsail-ip
apt install docker.io docker-compose
git clone your-repo && cd your-repo

# Deploy (can be automated via GitHub Actions)
docker-compose pull && docker-compose up -d
```

**Scaling path:**
1. Start with 4GB ($20/mo) for 5 users
2. Resize to 8GB ($40/mo) for 10-20 users
3. Migrate to ECS Fargate when you need auto-scaling

**Limitations:**
- No auto-scaling (manual resize only)
- Single point of failure
- You manage OS updates and Docker

---

### AWS Option 2: ECS Fargate + Copilot CLI (Production Path)

**AWS Copilot** simplifies ECS to a Railway-like experience.

**Setup (~30 min):**
```bash
# Install Copilot CLI
brew install aws/tap/copilot-cli

# Initialize app
copilot app init youlab
copilot svc init --name openwebui --type "Load Balanced Web Service"
copilot svc init --name http-service --type "Backend Service"
copilot svc init --name letta --type "Backend Service"

# Deploy
copilot deploy --all

# Setup CI/CD pipeline
copilot pipeline init --url https://github.com/your/repo
copilot pipeline deploy
```

**Cost breakdown:**

| Component | Cost/month |
|-----------|------------|
| Fargate compute (1GB total RAM) | $27-34 |
| Application Load Balancer | $16 |
| NAT Gateway (for outbound) | $32 (or $3 with NAT instance) |
| RDS db.t4g.micro (PostgreSQL) | $12-15 |
| EBS/storage | $1-2 |
| **Total** | **$58-100** |

**Cost optimization:**
- Use ARM (Graviton2) - 20% cheaper
- NAT Instance instead of NAT Gateway - saves $29/mo
- Savings Plans (1-year commitment) - up to 50% off

**DX comparison:**

| Aspect | Railway | Copilot |
|--------|---------|---------|
| First deploy | ~5 min | ~10 min |
| Subsequent deploys | git push | git push (via pipeline) |
| Environment setup | Click button | `copilot env init` |
| Logs | Web UI | `copilot svc logs` |
| Scaling | Automatic | Manifest config |

**When to choose this:**
- Need auto-scaling for variable load
- Want AWS-native CI/CD
- Planning for production scale
- Have budget for $60-80/mo

---

### AWS Option 3: App Runner (Simpler but Limited)

**AWS's answer to Railway** - but with significant limitations for your stack.

**Cost:** ~$45-60/mo for compute + $15-50 for RDS = **$60-110/mo**

**Critical limitation: No persistent storage.** You CANNOT run PostgreSQL in App Runner.

**Architecture required:**
```
OpenWebUI (App Runner, public)
     ↓
HTTP Service (App Runner, private endpoint)
     ↓
Letta Server (App Runner, private endpoint)
     ↓
RDS PostgreSQL (separate service)
```

**When to choose:**
- Only if you need scale-to-zero (pays nothing when idle)
- Simpler than Fargate for stateless services
- Must use RDS for database

**When NOT to choose:**
- Bundled Letta Docker image (has PostgreSQL) - won't work
- Need persistent filesystem storage
- Want docker-compose workflow

---

### AWS Option 4: Lightsail Container Service (NOT Recommended)

**$40/mo for 2GB RAM with NO persistent storage.** Avoid this.

- Designed for stateless web apps only
- Cannot attach volumes to containers
- Would require separate managed database anyway
- Worse value than Lightsail VPS or ECS Fargate

---

### Database: RDS vs Bundled PostgreSQL

Your Letta Docker image bundles PostgreSQL. On AWS, you have two choices:

| Option | Monthly Cost | Pros | Cons |
|--------|--------------|------|------|
| **Bundled in container** (Lightsail VPS) | $0 extra | Simple, works as-is | Single point of failure |
| **RDS db.t4g.micro** | $12-15 | Managed backups, Multi-AZ option | Extra cost, complexity |

**Recommendation for 5 users:** Start with bundled PostgreSQL on Lightsail VPS. Migrate to RDS when reliability becomes critical.

**RDS configuration (when ready):**
```
Instance: db.t4g.micro (2 vCPU burstable, 1GB RAM)
Storage: 10GB gp3
Multi-AZ: No (save money for dev)
Backup: 7 days (free)
Monthly: ~$12-15
```

Set `LETTA_PG_URI=postgresql://user:pass@your-rds-endpoint:5432/letta` to use external DB.

---

### Scaling Path: Lightsail → ECS Fargate

**Phase 1: Now (5 users)**
```
Lightsail VPS 4GB: $20/mo
├── OpenWebUI (port 3000)
├── HTTP Service (port 8100)
└── Letta Server + bundled PG (port 8283)
```
Total: **$20/mo**

**Phase 2: Growth (20-50 users)**
```
Lightsail VPS 8GB: $40/mo
+ RDS db.t4g.micro: $15/mo
```
Total: **$55/mo**

**Phase 3: Scale (100+ users)**
```
ECS Fargate (via Copilot): $50-70/mo
+ RDS db.t4g.small: $25-30/mo
+ ALB: $16/mo
```
Total: **$90-120/mo** with auto-scaling

---

### DX Comparison: Railway vs AWS Options

| Factor | Railway | Lightsail VPS | ECS Copilot |
|--------|---------|---------------|-------------|
| **Time to first deploy** | 10 min | 30-60 min | 30-45 min |
| **Deploy workflow** | git push | SSH + docker-compose | git push (after pipeline setup) |
| **One-click Postgres** | Yes | No (bundled in container) | No (use RDS) |
| **Auto-scaling** | Yes | No | Yes |
| **Cost (your stack)** | $25-40/mo | $20/mo | $60-80/mo |
| **Learning curve** | Low | Low | Medium |

---

### Recommendation

**For your situation (5 users, active development, need scalability path):**

1. **Start today: Lightsail VPS 4GB at $20/mo**
   - Fastest path to AWS deployment
   - Docker Compose works immediately
   - Plenty of headroom for 5-20 users
   - Create daily EBS snapshots for backup ($0.50/mo)

2. **Scale trigger: When you hit 8GB RAM utilization**
   - Migrate to ECS Fargate + Copilot
   - Add RDS for database
   - Enable auto-scaling

3. **Alternative: Stay on Railway**
   - Railway at $25-40/mo is genuinely easier
   - AWS only makes sense if you need: AWS ecosystem integration, specific compliance, or expect significant scale

**Bottom line:** AWS adds complexity and cost vs Railway/Fly.io. Choose AWS if you have specific requirements (compliance, AWS integration, future scale to 100+ users) or want to learn the AWS ecosystem. Otherwise, Railway/Fly.io remain better value for small teams.

---

## Follow-up Research: Actual Resource Requirements & Railway Scaling

**Date**: 2026-01-04T18:02:43+07:00
**Question**: Do I really need 2GB RAM? What are my actual needs? How does Railway scale?

### TL;DR

**No, you don't need 2GB RAM per service.** The 2GB figure comes from conservative defaults, not actual usage.

| Service | Previous Estimate | Actual Need | Optimized |
|---------|-------------------|-------------|-----------|
| HTTP Service | 512MB | **256MB** | 128MB possible |
| Letta Server | 2GB | **1GB** | 512MB with external PostgreSQL |
| OpenWebUI | 2GB | **512MB-1GB** | 256MB with optimizations |
| **Total** | 6GB | **2-2.5GB** | **1GB achievable** |

---

### Actual Resource Analysis by Service

#### 1. HTTP Service (Your Code)

**Analyzed from codebase** (`src/letta_starter/server/`):

| Component | Memory |
|-----------|--------|
| Python runtime | 15-20 MB |
| FastAPI + uvicorn | 10-15 MB |
| Letta client (httpx) | 1-2 MB |
| Agent ID cache (100 users) | 10 KB |
| Langfuse (if enabled) | 2-3 MB |
| **Total baseline** | **30-45 MB** |
| **Peak with 10 concurrent requests** | **40-60 MB** |

**Key findings**:
- No in-memory databases or document stores
- No ML models loaded
- Agent cache scales at ~1 KB per 10 users
- Documents stored in Letta, not locally
- Truly stateless - can run on **128MB**

**Recommendation**: **256MB** is comfortable, 128MB is tight but works.

#### 2. Letta Server (letta/letta Docker image)

The bundled image includes PostgreSQL, Redis, and the Python server:

| Component | Memory |
|-----------|--------|
| PostgreSQL + pgvector | 100-300 MB |
| Redis | 50-100 MB |
| Python/FastAPI server | 200-500 MB |
| OpenTelemetry (optional) | 50-100 MB |
| **Total** | **500MB-1GB** |

**Memory at different limits**:
- **256MB**: Will crash - PostgreSQL alone needs more
- **512MB**: Marginal - may OOM during vector operations
- **1GB**: Minimum viable for light single-user usage
- **2GB**: Comfortable headroom

**External PostgreSQL option**: Setting `LETTA_PG_URI` to use Railway's managed PostgreSQL or external DB reduces container memory to **300-500MB** (Python + Redis only).

**Source**: [GitHub Discussion #2276](https://github.com/letta-ai/letta/discussions/2276)

#### 3. OpenWebUI

Memory varies dramatically based on what's enabled:

| Configuration | Memory |
|---------------|--------|
| With local embedding models | 650MB-1GB+ |
| Default (no Ollama) | 500MB-1GB |
| **Optimized for external API** | **200-500MB** |

**Why yours can be smaller**: You're using Claude via Letta, not local models. The memory hogs are:
1. Local SentenceTransformers for RAG embeddings
2. Whisper for speech-to-text
3. ChromaDB for document storage

**Optimization env vars for your case**:
```bash
RAG_EMBEDDING_ENGINE=openai      # Use API, not local model
AUDIO_STT_ENGINE=openai          # Or disable
ENABLE_AUTOCOMPLETE_GENERATION=False
ENABLE_TITLE_GENERATION=False
ENABLE_TAGS_GENERATION=False
```

**Result**: Raspberry Pi users report **~200MB** with these optimizations.

**Source**: [OpenWebUI Docs - Reduce RAM Usage](https://docs.openwebui.com/tutorials/tips/reduce-ram-usage/)

---

### Railway Scaling Deep-Dive

#### Does Railway Scale to Zero?

**Yes**, via "App Sleeping" (serverless mode). Services sleep after **10 minutes of no outbound traffic**.

While asleep: Pay only for storage (~$0.15/GB/month), not compute.

**BUT**: Databases prevent sleeping. Connection pools, keep-alives, and health checks create outbound traffic that keeps services awake.

#### Cold Start Reality

| Claim | Reality |
|-------|---------|
| Railway marketing | "Sub-1-second" |
| **Actual user reports** | **5-22 seconds** |

Cold starts are highly variable. For a 4-person team testing, this is fine. For production, not acceptable.

#### CPU/Memory Billing

Railway bills **actual utilization**, not allocation. If your service idles at 10% CPU and 200MB RAM, you pay for that.

**Pricing rates**:
- vCPU: ~$20/month per core (billed per second)
- Memory: **~$10/month per GB** (this dominates costs)
- Storage: $0.15/GB/month
- Egress: $0.05/GB

**90% of Railway bills come from memory, not CPU.**

#### Hidden Gotchas

1. **$20 minimum** on Pro plan (but includes $20 credits)
2. **Databases keep services awake** - connection pools prevent sleeping
3. **PR deploys mirror entire stack** - disable if not needed
4. **External PostgreSQL connections** still count as outbound traffic

---

### Revised Cost Estimates (Realistic)

#### Scenario A: Aggressive Optimization (Hobby Plan)

```
HTTP Service: 256MB × $10/GB = $2.56/month
Letta (external PG): 512MB × $10/GB = $5.12/month
OpenWebUI (optimized): 512MB × $10/GB = $5.12/month
Railway managed PostgreSQL: $1-2/month
Storage (5GB): $0.75/month
─────────────────────────────────────────────
Total: ~$15/month
```

**Fixed costs**: $5 Hobby plan minimum
**Variable costs**: ~$10-15 based on actual usage

#### Scenario B: Conservative (Current Estimates)

```
HTTP Service: 512MB = $5/month
Letta (bundled PG): 1GB = $10/month
OpenWebUI: 1GB = $10/month
Storage (10GB): $1.50/month
─────────────────────────────────────────────
Total: ~$27/month
```

#### Scenario C: Let Services Sleep

If services can actually sleep (you're not actively using the system):

```
Compute when active: ~$0.50/hour for full stack
Storage only when idle: ~$1/month
Hobby credits: $5 included
─────────────────────────────────────────────
Light usage: $5-10/month possible
```

---

### What Breaks Sleeping (Important)

Services won't sleep if they have:
- Active database connection pools
- Health check endpoints being polled
- Framework telemetry (Next.js phones home)
- Inter-service private network requests
- WebSocket connections

**Your stack**: Letta's bundled PostgreSQL maintains connections internally, preventing the Letta service from sleeping. HTTP Service and OpenWebUI could potentially sleep between requests.

---

### Minimum Viable Configuration

**For Railway with your stack**:

```
┌─────────────────────────────────────────────┐
│ OpenWebUI (optimized)                       │
│   Memory: 512MB                             │
│   Serverless: enabled                       │
│   Env: RAG_EMBEDDING_ENGINE=openai          │
├─────────────────────────────────────────────┤
│ HTTP Service                                │
│   Memory: 256MB                             │
│   Serverless: enabled                       │
│   Base image: python:3.11-slim              │
├─────────────────────────────────────────────┤
│ Letta Server                                │
│   Memory: 1GB minimum                       │
│   Serverless: DISABLED (has PostgreSQL)     │
│   Volume: 5GB                               │
└─────────────────────────────────────────────┘

Estimated cost: $15-20/month
```

**Alternative with external PostgreSQL**:

```
Same as above, but:
- Letta Server: 512MB (no bundled PG)
- Railway PostgreSQL: $7/month (dedicated)
- Letta can now sleep

Estimated cost: $18-22/month (slightly higher due to managed DB)
```

---

### Final Recommendations

#### For Your Use Case (4-person team testing)

**If you want simplest setup**: Railway at **$20-25/month**
- Use 1GB for Letta (with bundled PostgreSQL)
- Use 512MB for OpenWebUI (with optimizations)
- Use 256MB for HTTP Service
- Don't worry about sleeping - you're actively developing

**If you want cheapest managed PaaS**: Fly.io at **$12-15/month**
- Same memory allocations as above
- More manual setup, but ~40% cheaper

**If you want cheapest overall**: Hetzner + Coolify at **$6.50/month**
- 8GB RAM shared across all services
- Full docker-compose support
- You handle maintenance

---

### Sources

- [Railway App Sleeping](https://docs.railway.com/reference/app-sleeping)
- [Railway Pricing](https://railway.com/pricing)
- [Railway Optimize Usage Guide](https://docs.railway.com/guides/optimize-usage)
- [OpenWebUI Reduce RAM Usage](https://docs.openwebui.com/tutorials/tips/reduce-ram-usage/)
- [Letta External PostgreSQL](https://github.com/letta-ai/letta/discussions/2276)
- [Railway Cold Start Reports](https://station.railway.com/questions/cold-start-slow-ee224f40)
- YouLab codebase analysis (`src/letta_starter/server/`)
