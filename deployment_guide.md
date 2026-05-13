# Pulse — Deployment Guide

> **Before anything else — read this critical note.**

## ⚠️ Critical: SQLite → Cloud Database Required

Your local app uses **SQLite** (`file:./dev.db`). SQLite is a local file — it **cannot** be used on any serverless/cloud platform (Vercel, Render, Railway) because:
- Vercel has **no persistent filesystem** (serverless functions are stateless)
- Render free tier **resets the disk on every deploy**

You must migrate to a **cloud-hosted database** before deploying. This guide covers the three best free options:

| Platform | DB Option | Free Tier | Best For |
|---|---|---|---|
| **Vercel** | Turso (SQLite-compatible) | 9GB storage, 1B reads | ⭐ Easiest — recommended |
| **Render** | PostgreSQL (Render managed) | 1GB, 90 days free | Full-stack Node.js |
| **Railway** | PostgreSQL (built-in) | $5 credit/month | All-in-one simplicity |

---

## Option A — Vercel + Turso (Recommended ⭐)

Turso is a cloud SQLite database — **zero schema changes needed**, just swap the URL.

### Step 1: Set Up Turso Database (free)

1. Install Turso CLI:
   ```bash
   # Windows (PowerShell — run as admin)
   winget install --id=ChiselStrike.Turso -e
   # OR via npm
   npm install -g @turso/cli
   ```

2. Login and create your database:
   ```bash
   turso auth login
   turso db create pulse-db
   turso db show pulse-db     # note the URL
   turso db tokens create pulse-db   # copy the token
   ```

3. You'll get two values — save them:
   ```
   TURSO_DATABASE_URL=libsql://pulse-db-<your-name>.turso.io
   TURSO_AUTH_TOKEN=eyJ...
   ```

### Step 2: Update Prisma for Turso

1. Install the Turso adapter:
   ```bash
   cd "c:\Users\ATUL\Downloads\files (1)\pulse-app"
   npm install @libsql/client @prisma/adapter-libsql
   ```

2. Update `prisma/schema.prisma` — change just the datasource block:
   ```prisma
   datasource db {
     provider     = "sqlite"
     url          = env("TURSO_DATABASE_URL")
     relationMode = "prisma"
   }
   ```
   > [!NOTE]
   > Keep `provider = "sqlite"` — Turso is fully SQLite-compatible.

3. Update `lib/db.ts` to use the Turso adapter:
   ```typescript
   // lib/db.ts
   import { PrismaClient } from "@prisma/client";
   import { PrismaLibSQL } from "@prisma/adapter-libsql";
   import { createClient } from "@libsql/client";

   const globalForPrisma = globalThis as unknown as {
     prisma: PrismaClient | undefined;
   };

   function createPrismaClient() {
     // Use Turso in production, local SQLite in development
     if (process.env.TURSO_DATABASE_URL && process.env.TURSO_AUTH_TOKEN) {
       const libsql = createClient({
         url: process.env.TURSO_DATABASE_URL,
         authToken: process.env.TURSO_AUTH_TOKEN,
       });
       const adapter = new PrismaLibSQL(libsql);
       return new PrismaClient({ adapter, log: ["error"] });
     }
     return new PrismaClient({ log: ["error"] });
   }

   export const db = globalForPrisma.prisma ?? createPrismaClient();

   if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
   ```

4. Push the schema to Turso:
   ```bash
   # Set env temporarily and push
   $env:TURSO_DATABASE_URL="libsql://pulse-db-<your-name>.turso.io"
   $env:TURSO_AUTH_TOKEN="eyJ..."
   npx prisma db push
   ```

### Step 3: Push to GitHub

```bash
cd "c:\Users\ATUL\Downloads\files (1)\pulse-app"

# Make sure .gitignore has these (check it already does):
# .env
# prisma/dev.db
# .next/

git init
git add .
git commit -m "feat: initial Pulse AI app"

# Create a new repo on github.com, then:
git remote add origin https://github.com/<your-username>/pulse-app.git
git push -u origin main
```

### Step 4: Deploy to Vercel

1. Go to **[vercel.com](https://vercel.com)** → Sign in with GitHub
2. Click **"Add New Project"** → Import your `pulse-app` repo
3. Vercel auto-detects Next.js — no framework config needed
4. Under **"Environment Variables"**, add all of these:

   | Key | Value |
   |---|---|
   | `TURSO_DATABASE_URL` | `libsql://pulse-db-<your-name>.turso.io` |
   | `TURSO_AUTH_TOKEN` | `eyJ...` (your Turso token) |
   | `AUTH_SECRET` | `pulse-ai-super-secret-nextauth-key-2025-xk9mzp` |
   | `NEXTAUTH_URL` | `https://your-app.vercel.app` (your Vercel URL) |
   | `GEMINI_API_KEY` | `AIzaSyCFOEnUqqkrpuCJZYqn3_yHZMcWvK-GaLo` |

5. Set the **Build Command** (in Project Settings → Build & Output):
   ```
   prisma generate && next build
   ```

6. Click **Deploy** ✅

> [!IMPORTANT]
> After first deploy, run the seed script once from your local machine pointing at Turso:
> ```bash
> $env:TURSO_DATABASE_URL="libsql://..."
> $env:TURSO_AUTH_TOKEN="eyJ..."
> npx ts-node --skip-project prisma/seed.ts
> ```

---

## Option B — Render + PostgreSQL

Render offers a managed PostgreSQL database with a free 90-day trial.

### Step 1: Migrate Schema to PostgreSQL

1. Update `prisma/schema.prisma`:
   ```prisma
   datasource db {
     provider = "postgresql"
     url      = env("DATABASE_URL")
   }
   ```

2. Update `lib/db.ts` — revert to the simple version:
   ```typescript
   import { PrismaClient } from "@prisma/client";
   const globalForPrisma = globalThis as unknown as { prisma: PrismaClient | undefined };
   export const db = globalForPrisma.prisma ?? new PrismaClient({ log: ["error"] });
   if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
   ```

3. Re-generate the Prisma client:
   ```bash
   npx prisma generate
   ```

4. Push to GitHub (same as Option A, Step 3).

### Step 2: Create Render Services

1. Go to **[render.com](https://render.com)** → Sign up/in

2. **Create PostgreSQL database first:**
   - New → PostgreSQL
   - Name: `pulse-db`
   - Region: choose closest
   - Plan: **Free** (90 days)
   - Copy the **Internal Database URL** shown after creation

3. **Create Web Service:**
   - New → Web Service → Connect your GitHub repo
   - Runtime: **Node**
   - Build Command:
     ```
     npm install && npx prisma generate && npx prisma db push && npm run build
     ```
   - Start Command:
     ```
     npm start
     ```

4. **Environment Variables** in Render dashboard:

   | Key | Value |
   |---|---|
   | `DATABASE_URL` | (paste Internal DB URL from step 2) |
   | `AUTH_SECRET` | `pulse-ai-super-secret-nextauth-key-2025-xk9mzp` |
   | `NEXTAUTH_URL` | `https://pulse-app.onrender.com` (your Render URL) |
   | `GEMINI_API_KEY` | `AIzaSyCFOEnUqqkrpuCJZYqn3_yHZMcWvK-GaLo` |
   | `NODE_ENV` | `production` |

5. Click **Create Web Service** → Deploy ✅

> [!WARNING]
> Render free tier **spins down after 15 min of inactivity** — the first request after sleep takes ~30s to respond (cold start).

---

## Option C — Railway (Simplest All-in-One)

Railway provisions everything from one dashboard with `$5` free credit/month.

### Steps

1. Go to **[railway.app](https://railway.app)** → Login with GitHub

2. New Project → **Deploy from GitHub repo** → select `pulse-app`

3. Click **"Add Service"** → **Database** → **PostgreSQL**
   - Railway auto-injects `DATABASE_URL` into your app service

4. Update `prisma/schema.prisma` to use PostgreSQL (same as Option B, Step 1)

5. In your app service → **Variables**, add:

   | Key | Value |
   |---|---|
   | `AUTH_SECRET` | `pulse-ai-super-secret-nextauth-key-2025-xk9mzp` |
   | `NEXTAUTH_URL` | `https://pulse-app.up.railway.app` |
   | `GEMINI_API_KEY` | `AIzaSyCFOEnUqqkrpuCJZYqn3_yHZMcWvK-GaLo` |

6. In app service → **Settings** → **Deploy**:
   - Build Command: `npm install && npx prisma generate && npx prisma db push && npm run build`
   - Start Command: `npm start`

7. Click **Deploy** ✅

---

## Pre-Deployment Checklist

- [ ] `.env` is in `.gitignore` (never commit secrets)
- [ ] `prisma/dev.db` is in `.gitignore`
- [ ] Database schema is pushed to the cloud DB (`prisma db push`)
- [ ] `NEXTAUTH_URL` matches your actual deployment URL exactly
- [ ] `AUTH_SECRET` is set (Auth.js v5 won't work without it)
- [ ] `GEMINI_API_KEY` is added to the platform's env vars
- [ ] Build command includes `prisma generate` before `next build`

---

## Environment Variables Quick Reference

```env
# Required on ALL platforms
AUTH_SECRET=pulse-ai-super-secret-nextauth-key-2025-xk9mzp
NEXTAUTH_URL=https://YOUR-DEPLOYMENT-URL.com
GEMINI_API_KEY=AIzaSyCFOEnUqqkrpuCJZYqn3_yHZMcWvK-GaLo

# Vercel + Turso only
TURSO_DATABASE_URL=libsql://pulse-db-yourname.turso.io
TURSO_AUTH_TOKEN=eyJ...

# Render / Railway (PostgreSQL)
DATABASE_URL=postgresql://user:pass@host:5432/dbname
```

---

## Recommendation Summary

```
🥇 Vercel + Turso    → Best DX, global CDN, instant deploys, stays SQLite
🥈 Railway           → Easiest setup, everything in one place, $5 free credit
🥉 Render            → Good for Node.js, PostgreSQL free 90 days
```

For a Next.js 16 App Router project like Pulse, **Vercel + Turso** is the most natural fit — zero cold starts, global edge network, and Turso's LibSQL is 100% SQLite-compatible so your Prisma schema barely changes.
