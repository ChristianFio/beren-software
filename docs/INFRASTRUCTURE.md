# Infrastructure overview

> Source of truth for `berensofware.com`. Updated 2026-04-27.
> See also: [DEPLOY.md](./DEPLOY.md) and [EMAIL.md](./EMAIL.md).

## Architecture (one diagram, no surprises)

```
                          ┌─────────────────────┐
                          │  Cloudflare DNS     │
                          │  elle.ns + rommy.ns │
                          │  Zone: Active       │
                          └──────────┬──────────┘
                                     │
                ┌────────────────────┼─────────────────────┐
                │                    │                     │
                ▼                    ▼                     ▼
         ┌──────────────┐    ┌──────────────┐      ┌──────────────┐
         │   A @ →      │    │ MX route1/   │      │ TXT SPF +    │
         │ 76.76.21.21  │    │ 2/3.mx.cf    │      │ DKIM cf2024  │
         │ (Vercel)     │    └──────┬───────┘      └──────────────┘
         └──────┬───────┘           │
                │                   │
                ▼                   ▼
       ┌────────────────┐   ┌──────────────────────┐
       │ Vercel project │   │ Cloudflare Email     │
       │ beren-software │   │ Routing (free)       │
       │ Auto-deploy    │   │ info@    →           │
       │ via GH Actions │   │ support@ → berensoftware@gmail.com
       │ (push to main) │   │ catch-all → drop     │
       └────────────────┘   └──────────────────────┘
```

## Domain & DNS

| Field | Value |
|---|---|
| Domain | `berensofware.com` (one `t` — careful, `berensoftware.com` exists and is unrelated) |
| Registrar | Vercel ($11.25/year, renewed automatically) |
| Expiration | 17 April 2027 |
| Authoritative DNS | Cloudflare (`elle.ns.cloudflare.com`, `rommy.ns.cloudflare.com`) |
| Cloudflare zone ID | `ace7f2d09ad3421ab99de669618e7394` |
| Cloudflare account ID | `03eb1f70d324e573bde2ca54eefe0fb9` |
| Cloudflare account email | `fiorillo.christian@gmail.com` |
| Vercel registrar API | `PATCH /v6/domains/berensofware.com` with `customNameservers` |

### Active DNS records (Cloudflare)

| Type | Name | Content | Notes |
|---|---|---|---|
| A | `@` | `76.76.21.21` | Vercel anycast, DNS only (no proxy) |
| CNAME | `www` | `cname.vercel-dns.com` | DNS only |
| CAA | `@` | issue `pki.goog` / `sectigo.com` / `letsencrypt.org` | SSL cert authorities |
| MX | `@` | `route1.mx.cloudflare.net` (prio 49), `route2…` (85), `route3…` (71) | Cloudflare Email Routing |
| TXT | `@` | `v=spf1 include:_spf.mx.cloudflare.net ~all` | SPF for routing |
| TXT | `cf2024-1._domainkey` | `v=DKIM1; …` | Auto-managed by Cloudflare |

**Do not add** `*` wildcard A or `_domainconnect` CNAME — they were Vercel-import artefacts and have been removed. Adding them back will not break anything but is unnecessary.

## Hosting (Vercel)

| Field | Value |
|---|---|
| Project | `beren-software` |
| Project ID | `prj_k2wAYjPSojN3EZNgfrHlZwrGLLm3` |
| Org/Team ID | `team_ovKLOZXaTiC8fn6qvIigFjHH` |
| Production URL | https://berensofware.com (+ `www.`) |
| Vercel preview URL | `beren-software.vercel.app` |
| Plan | Free / Hobby |
| Git connection | **Not connected** (Vercel GitHub App lacks repo permissions); deploys driven by CI |

## CI/CD (GitHub Actions)

| Field | Value |
|---|---|
| Repo | `ChristianFio/beren-software` |
| Workflow | [`.github/workflows/vercel-deploy.yml`](../.github/workflows/vercel-deploy.yml) |
| Trigger | `push` to `main` + `workflow_dispatch` (manual) |
| Steps | checkout → Node 22 → install vercel CLI → `vercel pull` → `vercel build --prod` → `vercel deploy --prebuilt --prod` |
| Concurrency | `group: vercel-prod`, in-flight runs cancelled on new commit |

There is also a legacy [`.github/workflows/pages.yml`](../.github/workflows/pages.yml) that publishes to GitHub Pages (`christianfio.github.io/beren-software/`). Harmless but unused as a public URL.

### Repository secrets

| Secret | Purpose |
|---|---|
| `VERCEL_TOKEN` | Auth for Vercel CLI in CI. Currently a session token; rotate to a project-scoped CI token at https://vercel.com/account/tokens |
| `VERCEL_ORG_ID` | `team_ovKLOZXaTiC8fn6qvIigFjHH` |
| `VERCEL_PROJECT_ID` | `prj_k2wAYjPSojN3EZNgfrHlZwrGLLm3` |

Rotate with `gh secret set VERCEL_TOKEN --body "<new>" --repo ChristianFio/beren-software`.

## Email

See [EMAIL.md](./EMAIL.md) for full detail. Quick summary:

- `info@berensofware.com` → `berensoftware@gmail.com`
- `support@berensofware.com` → `berensoftware@gmail.com`
- Anything else → dropped (catch-all DROP rule)
- Inbound only — outbound (Send-as from Gmail) requires SMTP relay (not yet configured)

## Cloudflare API access

A scoped API token `beren-software-deploy` exists with these permissions, all scoped to `berensofware.com` zone (and account-level for routing addresses):

- `Zone → DNS → Edit`
- `Zone → Email Routing Rules → Edit`
- `Account → Email Routing Addresses → Edit`

It does **not** have `Email Routing Settings:Edit` — adding/removing routing itself still requires dashboard access. Recreate the token from `dash.cloudflare.com/profile/api-tokens` if you need those permissions.

The token value is **not stored in the repo**. It currently lives only in Cloudflare's view; if you lose it, create a new one and rotate.

## Common operations

### "I want to redeploy now"

```
git commit --allow-empty -m "redeploy" && git push
```

Or trigger the workflow manually from GitHub Actions UI (`workflow_dispatch`).

### "I want to add another email alias on the domain"

Two ways:

**Via dashboard:**
1. https://dash.cloudflare.com → berensofware.com → Email → Email Routing → Routing rules
2. Create address rule

**Via API** (with the scoped token):
```bash
curl -X POST \
  https://api.cloudflare.com/client/v4/zones/ace7f2d09ad3421ab99de669618e7394/email/routing/rules \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "newsletter",
    "enabled": true,
    "matchers": [{"type":"literal","field":"to","value":"newsletter@berensofware.com"}],
    "actions":  [{"type":"forward","value":["berensoftware@gmail.com"]}]
  }'
```

### "I want to add another DNS record"

```bash
curl -X POST \
  https://api.cloudflare.com/client/v4/zones/ace7f2d09ad3421ab99de669618e7394/dns_records \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"TXT","name":"_dmarc","content":"v=DMARC1; p=none; rua=mailto:berensoftware@gmail.com","ttl":1}'
```

### "I want to add a subdomain pointing to a different host"

Add a `CNAME` (or `A`) record on Cloudflare. If the subdomain serves a Vercel app, point it to the Vercel project's CNAME and add the domain in the Vercel project settings.

### "I want to undo the Cloudflare migration and put DNS back on Vercel"

```bash
# (assuming Vercel CLI authenticated)
curl -X PATCH \
  "https://api.vercel.com/v6/domains/berensofware.com?teamId=team_ovKLOZXaTiC8fn6qvIigFjHH" \
  -H "Authorization: Bearer $VERCEL_SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"customNameservers":[]}'
```

This restores the default `ns1/ns2.vercel-dns.com`. You'll lose Email Routing (and need to re-add MX/SPF for whichever provider you choose).

## Things that are **not** wired up

- ❌ **Outbound SMTP** for `info@`/`support@` (cannot send "as" yet)
- ❌ **DMARC record** (recommended `p=none` to start collecting reports)
- ❌ **Vercel Git integration** (Vercel App doesn't have repo permissions; deploys via CI)
- ❌ **Production secrets management** beyond GitHub repo secrets
- ❌ **Analytics** / tracking (deliberate — no profiling cookies, see Cookie Policy)
- ❌ **Web3Forms access key** in the contact form (still placeholder `REPLACE_WITH_WEB3FORMS_KEY`); form falls back to `mailto:` until configured
