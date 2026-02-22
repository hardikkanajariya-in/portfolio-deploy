# Portfolio Deploy (Vercel)

This repository deploys the private `portfolio` repo to Vercel free hosting using GitHub Actions.

## Architecture

```
┌─────────────────────────┐         ┌──────────────────────────┐         ┌─────────┐
│  Private Repo           │         │  Public Repo             │         │         │
│  (portfolio)            │ ──────> │  (portfolio-deploy)      │ ──────> │ Vercel  │
│                         │ trigger │                          │ deploy  │         │
│  Push to main           │         │  Clone → Build → Deploy  │         │         │
└─────────────────────────┘         └──────────────────────────┘         └─────────┘
      Uses: PUBLIC_REPO_PAT              Uses: PRIVATE_REPO_PAT
                                               VERCEL_TOKEN
                                               VERCEL_ORG_ID
                                               VERCEL_PROJECT_ID
                                               DATABASE_URL
                                               DEMO_DATABASE_URL
                                               JWT_SECRET
                                               CRON_SECRET

Workflows:
  deploy.yml       → Build & deploy (production / preview / demo)
  demo-reset.yml   → Scheduled every 8h: reset demo DB + seed + redeploy
```

## Setup Guide

### Step 1: Create this public repo on GitHub

```bash
cd portfolio-deploy
git init
git add .
git commit -m "Initial commit: Vercel deploy workflow"
git branch -M main
git remote add origin https://github.com/hardikkanajariya-in/portfolio-deploy.git
git push -u origin main
```

### Step 2: Create a Vercel Project

1. Go to [vercel.com](https://vercel.com) and sign in
2. Click **"Add New Project"**
3. Select **"Import Git Repository"** — but since the repo is private, we'll use CLI instead:

```bash
# Install Vercel CLI
npm install -g vercel

# Login to Vercel
vercel login

# Link the project (run this in any temp folder)
mkdir temp-vercel && cd temp-vercel
vercel link
# Follow prompts: create new project, name it "portfolio" or whatever you want
```

4. After linking, check `.vercel/project.json` for:
   - `orgId` → This is your **VERCEL_ORG_ID**
   - `projectId` → This is your **VERCEL_PROJECT_ID**

### Step 3: Generate Tokens

#### GitHub PAT (for cloning private repo)
1. Go to [GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens](https://github.com/settings/tokens?type=beta)
2. Create a new token with:
   - **Repository access**: Select `portfolio` (private repo)
   - **Permissions**: Contents → Read-only
3. Copy the token → This is your **PRIVATE_REPO_PAT**

#### GitHub PAT (for triggering deploy from private repo)
1. Create another fine-grained token with:
   - **Repository access**: Select `portfolio-deploy` (public repo)
   - **Permissions**: Contents → Read & Write
2. Copy the token → This is your **PUBLIC_REPO_PAT** (add this to the **private** repo's secrets)

#### Vercel Token
1. Go to [Vercel Account Settings → Tokens](https://vercel.com/account/tokens)
2. Create a new token, name it "github-deploy"
3. Copy the token → This is your **VERCEL_TOKEN**

### Step 4: Add Secrets

#### In the PUBLIC repo (`portfolio-deploy`) → Settings → Secrets → Actions:

| Secret | Description |
|--------|-------------|
| `PRIVATE_REPO_PAT` | GitHub PAT with read access to private repo |
| `VERCEL_TOKEN` | Vercel API token |
| `VERCEL_ORG_ID` | From `.vercel/project.json` → `orgId` |
| `VERCEL_PROJECT_ID` | From `.vercel/project.json` → `projectId` |
| `DATABASE_URL` | PostgreSQL connection string (production) |
| `DEMO_DATABASE_URL` | PostgreSQL connection string (demo — separate DB!) |
| `JWT_SECRET` | Your JWT secret |
| `CRON_SECRET` | Your cron secret |

#### In the PRIVATE repo (`portfolio`) → Settings → Secrets → Actions:

| Secret | Description |
|--------|-------------|
| `PUBLIC_REPO_PAT` | GitHub PAT with write access to public repo (for triggering deploy) |

### Step 5: Configure Vercel Environment Variables

Go to your Vercel project → Settings → Environment Variables and add:

| Variable | Value |
|----------|-------|
| `DATABASE_URL` | Your PostgreSQL connection string |
| `JWT_SECRET` | Your JWT secret |
| `CRON_SECRET` | Your cron secret |

### Step 6: Set Up Database

You need **two separate PostgreSQL databases** — one for production and one for demo.

#### Recommended Free Providers
- **[Neon](https://neon.tech)** — Free tier, serverless Postgres (recommended). Create 2 databases in one project.
- **[Supabase](https://supabase.com)** — Free tier Postgres (1 project per org, create 2 orgs for 2 DBs)
- **[Railway](https://railway.app)** — Free trial Postgres

#### Database Setup

1. Create **two databases** (e.g., `portfolio_prod` and `portfolio_demo`)
2. Get the connection strings for both
3. Add `DATABASE_URL` (production) and `DEMO_DATABASE_URL` (demo) to the public repo secrets

> **Important**: The demo database gets **completely wiped and re-seeded every 8 hours** by the `demo-reset.yml` workflow. Never use the same database for production and demo!

#### Initial Database Setup (first time only)

After adding secrets, manually run:

1. **Production**: Run **"Deploy to Vercel"** workflow → branch: `main`, environment: `production`
   - This runs `prisma migrate deploy` to create tables
   - The app's `/install` page will handle initial setup (first user creation)

2. **Demo**: Run **"Reset Demo Database"** workflow → branch: `develop`
   - This runs `prisma migrate reset --force` + seeders
   - Seeds all demo data (users, blog posts, projects, invoices, etc.)

## Demo Application

### How Demo Works

The demo application uses a **separate database** (`DEMO_DATABASE_URL`) and is automatically reset every 8 hours:

```
Every 8 hours (GitHub Actions cron):
  1. Clone private repo (develop branch)
  2. prisma migrate reset --force  →  Wipes all data
  3. tsx prisma/seeders/index.ts   →  Seeds 37 seeders with demo data
  4. vercel build + deploy         →  Redeploy with fresh state
```

### Demo Seeders (auto-run)

Your seeder (`prisma/seeders/index.ts`) runs in 10 phases:
1. **Core** — SiteConfig, email templates, users, media
2. **Content** — Blog posts, portfolio, products, services, skills, timeline
3. **Freelance** — Clients, projects, tasks, milestones, invoices, contracts
4. **E-commerce** — Coupons, orders, licenses
5. **CRM** — Leads
6. **Affiliate** — Affiliates, applications, post likes
7. **Time Tracking** — Tracker sessions, time entries, coding summaries
8. **Finance** — Expenses
9. **System** — Cron jobs, activity logs, notifications, blocked entities
10. **Misc** — FAQs, testimonials, subscribers, social links, legal templates, pages

The seeder detects `DEMO_MODE=true` env var and seeds all phases (vs production which only seeds SiteConfig).

### Manual Demo Reset

1. Go to **Actions** → **Reset Demo Database** → **Run workflow**
2. Options:
   - **branch**: Which branch to use (default: `develop`)
   - **seed_only**: If `true`, only resets DB + seeds without rebuilding/redeploying (faster, ~2 min vs ~8 min)

### Deploy Demo Without Reset

Use the main **Deploy to Vercel** workflow with environment: `demo` and `fresh_database: false` to redeploy without wiping the database.

## Usage

### Automatic Deploy (Production)
Push to `main` branch in the private repo → automatically triggers Vercel production deploy.

### Manual Deploy
1. Go to this repo → **Actions** tab
2. Click **"Deploy to Vercel"**
3. Click **"Run workflow"**
4. Select branch, environment (`production` / `preview` / `demo`), and whether to reset DB
5. Click **"Run workflow"**

### Deploy from Private Repo
Go to the private repo → **Actions** → **Trigger Vercel Deploy** → select environment and options.

## Vercel Free Tier Limits

| Resource | Limit |
|----------|-------|
| Bandwidth | 100 GB/month |
| Serverless Function Duration | 10 seconds |
| Builds | 6,000 minutes/month |
| Cron Jobs | Minimum 1/day (hourly set in vercel.json) |
| Edge Middleware | Supported |
| Image Optimization | 1,000 images/month |

## Important Notes

1. **File uploads won't persist** — Vercel has an ephemeral filesystem. Uploaded files will be lost on redeployment. For production use, integrate external storage (S3, Cloudflare R2, etc.)
2. **Custom server.ts is ignored** — Vercel uses its own serverless infrastructure. The `server.ts` file is not used.
3. **WebSocket not supported** — Vercel doesn't support persistent WebSocket connections. Use a separate service for real-time features.
4. **Cron minimum interval** — Free tier cron jobs run at minimum every 1 day. The `vercel.json` is set to hourly but will only work on paid plans.

## Troubleshooting

### Build fails with Prisma error
Make sure `DATABASE_URL` is set in both Vercel environment variables AND GitHub secrets.

### Deploy succeeds but app shows error
Check Vercel dashboard → Deployments → Function Logs for runtime errors.

### Trigger from private repo not working
- Verify `PUBLIC_REPO_PAT` token has `Contents: Read & Write` permission on this repo
- Check the private repo's Actions tab for error details
