# Deploy runbook

> See also: [INFRASTRUCTURE.md](./INFRASTRUCTURE.md).

The site is a single static directory deployed to Vercel. Auto-deploy runs on every push to `main`.

## Pipeline

```
git push origin main
        │
        ▼
GitHub Actions: .github/workflows/vercel-deploy.yml
        │  • Node 22
        │  • npm install -g vercel@latest
        │  • vercel pull --yes --environment=production
        │  • vercel build --prod
        │  • vercel deploy --prebuilt --prod
        ▼
Vercel project beren-software (prj_k2wAYjPSojN3EZNgfrHlZwrGLLm3)
        │
        ▼
https://berensofware.com  (+ www.berensofware.com)
```

Workflow runs ~25-35 seconds. `concurrency: vercel-prod, cancel-in-progress: true` means new commits cancel any in-flight build.

## Why this and not the Vercel GitHub App?

The Vercel GitHub App didn't have repository permissions for `ChristianFio/beren-software` when the project was created, so `vercel git connect` returned _Failed to connect... access_. Rather than fight the OAuth flow, the workflow drives the deploy directly via the Vercel CLI using a token stored as a GitHub repo secret.

If you ever want to switch back to native Git integration:

1. https://vercel.com → project `beren-software` → Settings → Git → "Connect Git repository"
2. Authorize the Vercel App for the `beren-software` repo on GitHub
3. Once connected, the GitHub Actions workflow becomes redundant and can be removed (or kept disabled as a fallback)

## Repository secrets

| Name | Set with | Notes |
|---|---|---|
| `VERCEL_TOKEN` | `gh secret set VERCEL_TOKEN --body "<token>" --repo ChristianFio/beren-software` | Currently a Vercel CLI session token; rotate to a scoped CI token (https://vercel.com/account/tokens) for least-privilege |
| `VERCEL_ORG_ID` | already set: `team_ovKLOZXaTiC8fn6qvIigFjHH` | Stable, no rotation needed |
| `VERCEL_PROJECT_ID` | already set: `prj_k2wAYjPSojN3EZNgfrHlZwrGLLm3` | Stable |

List with `gh secret list --repo ChristianFio/beren-software`.

## Local dev / preview

The site is plain HTML — no build step. Open `index.html` directly or:

```bash
python -m http.server 8000
# → http://localhost:8000
```

For a production-equivalent preview without pushing to main, force a Vercel preview deploy from CLI:

```bash
vercel deploy
```

(no `--prod` flag → preview URL only, never touches `berensofware.com`)

## Manual deploy / hotfix

If GitHub Actions is unavailable:

```bash
cd path/to/beren-software
vercel --prod --yes
```

(needs Vercel CLI authenticated with an account that has access to the `beren-software` project)

## Rollback

Vercel keeps every previous production deployment for free.

```bash
vercel rollback --token "$VERCEL_TOKEN"
# pick from the interactive list

# or, by URL:
vercel rollback https://beren-software-<hash>.vercel.app --token "$VERCEL_TOKEN"
```

The site at `berensofware.com` swings back to the chosen deployment within seconds. No DNS change involved.

## Domain config inside Vercel project

| Domain | Status |
|---|---|
| `berensofware.com` | Production, primary |
| `www.berensofware.com` | Production, redirects to apex |
| `beren-software.vercel.app` | Preview, default |
| `beren-software-fiorillochristian-3190s-projects.vercel.app` | Preview, team-scoped |

DNS lives on Cloudflare (see [INFRASTRUCTURE.md](./INFRASTRUCTURE.md)) — Vercel's own DNS records are not used for these domains.
