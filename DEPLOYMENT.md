# Deployment Guide

Complete, step-by-step walkthrough to take this repository from `git clone` to four live submission links:

1. GitHub Repository (frontend + backend)
2. Docker Hub backend image
3. Vercel frontend URL
4. Render backend API URL

All commands are written for **Windows PowerShell**. Run them from the repo root unless a step says otherwise.

---

## 0. Prerequisites

| Tool | Check it's installed |
|---|---|
| Git | `git --version` |
| Node.js 20+ | `node --version` |
| Docker Desktop | `docker --version` |
| GitHub account | — |
| Docker Hub account | — |
| Render account | — |
| Vercel account | — |
| Neon account | — |

---

## 1. Neon PostgreSQL connection setup

1. Sign in at [neon.tech](https://neon.tech) → **Create a project** → name it `inventory-management`.
2. In your project's **Dashboard → Connection Details**, select:
   - Driver: `Postgres`
   - Pooled connection: **on** (recommended — Render's free tier benefits from Neon's connection pooler)
3. Copy the connection string. It looks like:

   ```
   postgresql://neondb_owner:AbC123XyZ@ep-cool-leaf-12345678-pooler.us-east-2.aws.neon.tech/neondb?sslmode=require
   ```

4. Paste it as `DATABASE_URL` in `backend\.env` (local) — you'll paste it again into Render's environment variables in Step 5.
5. Apply the schema (creates Better Auth's `user`/`session`/`account`/`verification` tables plus `products`/`categories`):

   ```powershell
   cd backend
   npm install
   npm run db:migrate
   cd ..
   ```

   You should see `✅ Migration complete. Tables are ready.` Alternatively, Better Auth can generate/verify its own tables:

   ```powershell
   cd backend
   npx @better-auth/cli generate
   cd ..
   ```

> Neon's free tier auto-suspends idle databases — the first request after idle time may take a couple of seconds while it wakes up. This is expected.

---

## 2. Push the code to GitHub

```powershell
cd inventory-management-system
git init
git add .
git commit -m "Initial commit: Inventory Management System"
git branch -M main
git remote add origin https://github.com/<your-github-username>/inventory-management-system.git
git push -u origin main
```

Replace `<your-github-username>` with your actual GitHub username. Create the empty repository on GitHub first (no README/license, since this repo already has them) at `https://github.com/new`.

---

## 3. Production environment variables

Set these wherever each service runs. **Never commit real `.env` files** — only `.env.example` is tracked in Git.

### Backend (Render)

| Variable | Example value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | |
| `PORT` | `4000` | Render injects its own `PORT`; this is the fallback used by `server.js` |
| `DATABASE_URL` | `postgresql://...sslmode=require` | From Neon, Step 1 |
| `BETTER_AUTH_SECRET` | 32+ char random string | Generate below |
| `BETTER_AUTH_URL` | `https://inventory-management-api.onrender.com` | Your Render URL, set **after** first deploy |
| `FRONTEND_URL` | `https://inventory-management.vercel.app` | Used for CORS + Better Auth trusted origins; comma-separate multiple origins |

Generate `BETTER_AUTH_SECRET` in PowerShell:

```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

### Frontend (Vercel)

| Variable | Example value | Notes |
|---|---|---|
| `VITE_API_URL` | `https://inventory-management-api.onrender.com` | Your Render backend URL |

---

## 4. Docker Hub setup instructions

1. Create an account at [hub.docker.com](https://hub.docker.com) if you don't have one.
2. Create an **Access Token** (used instead of your password for CLI/CI login): **Account Settings → Security → New Access Token** → name it `inventory-ci` → copy the token now, it's shown once.
3. Create the repository (or let `docker push` auto-create it on first push): **Repositories → Create Repository** → name `inventory-management-backend` → Visibility: Public.

Log in from PowerShell:

```powershell
docker login -u <your-dockerhub-username>
```

Paste the access token when prompted for a password.

---

## 5. Docker build commands

Run from the repo root.

```powershell
# Build the backend image
docker build -t <your-dockerhub-username>/inventory-management-backend:latest .\backend

# (Optional) also tag with a version for traceability
docker build -t <your-dockerhub-username>/inventory-management-backend:v1.0.0 .\backend
```

Quick local smoke test before pushing:

```powershell
docker run --rm -p 4000:4000 --env-file .\backend\.env `
  <your-dockerhub-username>/inventory-management-backend:latest
```

Then in another PowerShell window:

```powershell
curl http://localhost:4000/api/health
```

You should get back `{"status":"ok", ... "database":"connected"}`. Stop the container with `Ctrl+C`.

---

## 6. Docker push commands

```powershell
docker push <your-dockerhub-username>/inventory-management-backend:latest
docker push <your-dockerhub-username>/inventory-management-backend:v1.0.0
```

Verify it's live at:
`https://hub.docker.com/r/<your-dockerhub-username>/inventory-management-backend`

---

## 7. Render deployment steps

You can deploy either via the Render Blueprint (`render.yaml`, fastest) or manually. Both are documented.

### Option A — Blueprint (recommended)

1. Render Dashboard → **New + → Blueprint**.
2. Connect your GitHub account and select `inventory-management-system`.
3. Render reads `render.yaml` and proposes the `inventory-management-api` service. Click **Apply**.
4. When prompted, fill in the environment variables marked `sync: false` (`DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `FRONTEND_URL`) using the table from Step 3.
5. Click **Create Web Service**. Render builds `backend/Dockerfile` and deploys it.

### Option B — Manual web service from a Docker Hub image

1. Render Dashboard → **New + → Web Service**.
2. Choose **Deploy an existing image from a registry**.
3. Image URL: `docker.io/<your-dockerhub-username>/inventory-management-backend:latest`.
4. Name: `inventory-management-api`. Region: closest to your users. Instance type: Free.
5. **Environment** tab → add all variables from the Step 3 table.
6. **Health Check Path**: `/api/health`.
7. Click **Create Web Service**.

### After first deploy

1. Copy the assigned URL, e.g. `https://inventory-management-api.onrender.com`.
2. Go back into the service's **Environment** tab and set `BETTER_AUTH_URL` to that exact URL (it can't be known before the first deploy).
3. Save — this triggers a redeploy automatically.
4. Get a **Deploy Hook** URL for CI (used in Step 9): **Settings → Deploy Hook → Create Deploy Hook** → copy the URL.

### Verify

```powershell
curl https://inventory-management-api.onrender.com/api/health
```

> Render's free tier spins services down after 15 minutes of inactivity. The next request will take ~30–60 seconds to "wake up" — this is expected free-tier behavior, not a bug.

---

## 8. Vercel deployment steps

1. Vercel Dashboard → **Add New → Project**.
2. Import `inventory-management-system` from GitHub.
3. **Root Directory**: click **Edit** and set it to `frontend`.
4. Framework Preset: Vercel should auto-detect **Vite** (confirmed by `frontend/vercel.json`).
5. Build Command: `npm run build` · Output Directory: `dist` (pre-filled from `vercel.json`).
6. **Environment Variables**: add `VITE_API_URL` = your Render backend URL (e.g. `https://inventory-management-api.onrender.com`).
7. Click **Deploy**.
8. Once deployed, Vercel assigns a URL like `https://inventory-management.vercel.app` (rename the project beforehand in **Project Settings → General → Project Name** if you want this exact name).

### After first deploy

Go back to **Render → Environment** and set `FRONTEND_URL` to your final Vercel URL (and redeploy) so CORS and Better Auth's `trustedOrigins` allow requests from it.

### Verify

Open the Vercel URL, register an account, sign in, and add a product — confirms frontend ↔ backend ↔ Neon are all connected end to end.

---

## 9. GitHub Actions — automatic deployment

The workflow at `.github/workflows/deploy.yml` runs on every push to `main`:

- **backend-build-test** → installs backend deps, sanity-checks `server.js`
- **backend-docker** → builds the backend image and pushes `:latest` + `:<commit-sha>` to Docker Hub, then calls the Render deploy hook
- **frontend-build-test** → installs frontend deps and runs `npm run build`
- **vercel-deploy** → builds and deploys the frontend via the Vercel CLI

### Required GitHub Secrets

Go to your repo → **Settings → Secrets and variables → Actions → New repository secret**, and add:

| Secret | Where to get it |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | The access token from Step 4 |
| `RENDER_DEPLOY_HOOK_URL` | From Render → service **Settings → Deploy Hook** (Step 7) |
| `VITE_API_URL` | Your Render backend URL, e.g. `https://inventory-management-api.onrender.com` |
| `VERCEL_TOKEN` | Vercel → **Account Settings → Tokens → Create** |
| `VERCEL_ORG_ID` | From `.vercel/project.json` after running `vercel link` locally once (see below), or Vercel → Project → Settings → General |
| `VERCEL_PROJECT_ID` | Same as above |

To generate `.vercel/project.json` locally (one-time):

```powershell
npm install --global vercel
cd frontend
vercel link
cd ..
```

This creates `frontend\.vercel\project.json` containing `orgId` and `projectId` — copy those values into the GitHub secrets above. (`.vercel/` is already git-ignored.)

> If you'd rather not maintain `VERCEL_ORG_ID` / `VERCEL_PROJECT_ID` as secrets, Vercel's native GitHub integration (enabled by default when you import the repo in Step 8) will auto-deploy on every push without any extra setup — the `vercel-deploy` job is an optional, fully scripted alternative and can be removed from the workflow if you rely on the native integration instead.

### Trigger it

```powershell
git add .
git commit -m "Trigger deployment"
git push origin main
```

Watch progress under the repo's **Actions** tab.

---

## 10. Final submission checklist

- [ ] **GitHub Repository** pushed and public (or shared with reviewers): `https://github.com/<username>/inventory-management-system`
- [ ] `backend/.env` and `frontend/.env` are **not** committed (only `.env.example` files are)
- [ ] Neon database created, schema migrated (`npm run db:migrate` ran successfully)
- [ ] Backend Docker image built and pushed: `https://hub.docker.com/r/<username>/inventory-management-backend`
- [ ] Render service deployed and `/api/health` returns `{"status":"ok","database":"connected"}`
- [ ] Render env vars set: `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `FRONTEND_URL`, `NODE_ENV`
- [ ] Vercel project deployed: `https://inventory-management.vercel.app`
- [ ] Vercel env var set: `VITE_API_URL` pointing at the Render URL
- [ ] CORS verified: signing up / signing in from the Vercel URL works without console errors
- [ ] GitHub Actions secrets configured and the workflow run is green on the **Actions** tab
- [ ] Full user flow tested end-to-end on the **live URLs**: register → sign in → add product → edit product → delete product → sign out
- [ ] README.md and DEPLOYMENT.md links updated with your real GitHub/Docker Hub/Render/Vercel URLs

**Submission package:**

```
GitHub Repository:  https://github.com/<username>/inventory-management-system
Docker Hub Image:   https://hub.docker.com/r/<username>/inventory-management-backend
Frontend URL:       https://inventory-management.vercel.app
Backend URL:        https://inventory-management-api.onrender.com
```

---

## 11. Troubleshooting guide

**`docker build` fails with "no such file or directory"**
Run the build command from the repo root, not from inside `backend/` — the path argument (`.\backend`) already points Docker at the right context.

**Backend container starts then immediately exits**
Check logs: `docker logs <container-id>`. Almost always a missing/invalid `DATABASE_URL` — the app exits intentionally (`process.exit(1)`) if it isn't set, so the database is never silently unreachable in production.

**`/api/health` returns `"database":"disconnected"`**
- Confirm `DATABASE_URL` includes `?sslmode=require`.
- Confirm the Neon project isn't paused/deleted, and the password in the connection string hasn't been rotated.
- On Render, re-check the **Environment** tab for typos (a trailing space is a common cause).

**CORS errors in the browser console (`No 'Access-Control-Allow-Origin' header`)**
`FRONTEND_URL` on Render must exactly match the Vercel origin, including `https://` and no trailing slash. After changing it, Render must finish redeploying before it takes effect.

**Sign-up/sign-in returns 404 or hangs**
Confirm `app.all("/api/auth/*", toNodeHandler(auth))` is mounted in `backend/src/app.js` **before** `express.json()`. If `express.json()` runs first, Better Auth can't read the raw request body.

**Session doesn't persist / user gets logged out on refresh**
The browser must be allowed to send cross-site cookies: the frontend `fetch` calls already use `credentials: "include"` (see `frontend/src/lib/api.js` and Better Auth's client), and the backend `cors()` config already sets `credentials: true`. If you changed either, that's the first place to check. Also confirm `BETTER_AUTH_URL` matches the backend's real public URL exactly.

**Render free-tier service is slow on first request**
Expected — free instances spin down after 15 minutes idle and take ~30–60s to wake on the next request. Not an error.

**Vercel build fails with "Module not found"**
Make sure **Root Directory** is set to `frontend` in Vercel project settings (Step 8.3) — Vercel otherwise tries to build from the repo root, where there's no `package.json` for the frontend app alone.

**Vercel deploys but the page is blank / shows raw 404 on refresh of `/inventory`**
This is the classic SPA-routing issue. It's already handled by the `rewrites` rule in `frontend/vercel.json`; confirm that file exists and Vercel picked it up (check the deployment's **Source** tab).

**GitHub Action fails at `docker/login-action`**
`DOCKERHUB_TOKEN` must be an access token (Step 4), not your Docker Hub account password — Docker Hub disabled plain-password CI logins.

**GitHub Action fails at the Render deploy hook step**
`RENDER_DEPLOY_HOOK_URL` is service-specific and looks like `https://api.render.com/deploy/srv-xxxxx?key=yyyyy` — regenerate it from the Render dashboard if it was rotated or the service was recreated.

**`npm audit` flags a moderate/high vulnerability in `esbuild`/`vite` after `npm install`**
This is a known dev-server-only advisory (a malicious site could read responses from your local `vite dev` server). It does not affect the production build or the deployed app. Fix by upgrading to Vite's next major version if desired; not required for deployment.

**`npm ci` fails in Docker build with "lock file out of date"**
Your `package.json` and `package-lock.json` are out of sync. Run `npm install` locally to refresh the lockfile, commit it, and rebuild.
