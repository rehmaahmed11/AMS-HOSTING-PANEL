# AMS Cloud Panel — a hosting panel that runs on Cloudflare's free Worker plan

A multi-site hosting control panel + lightweight CMS, built as a single Cloudflare
Worker. No servers, no R2 bucket, no credit card: **D1 + KV only**, both of which
the free plan includes.

It provisions sites, hands out credentials, verifies DNS, backs up and
restores, updates and rolls back bundles, and health-checks the fleet — all
from a proper panel with an audit trail, roles and an API.

```
src/
├── src/
│   ├── index.ts            Worker entry: install gate → panel dispatch → site serving
│   ├── types.ts            Erasable TypeScript types shared by every runtime
│   ├── db/
│   │   ├── schema.sql      Portable D1/SQLite schema (13 tables)
│   │   ├── schema.generated.ts  generated from schema.sql
│   │   └── index.ts        query helpers (D1 + local SQLite shim compatible)
│   ├── lib/
│   │   ├── auth.ts         sessions, PBKDF2 passwords, CSRF, throttling, API keys
│   │   ├── crypto.ts       WebCrypto only (works on Workers and Node)
│   │   ├── kv.ts           KV helpers + panel settings + counters
│   │   ├── files.ts        KV-backed file store with free-plan size caps
│   │   ├── zip.ts          zero-dependency ZIP reader (stored + deflate-raw)
│   │   ├── deploy.ts       ZIP writer, bundle index/extract, backups
│   │   ├── sites.ts        site registry & validation
│   │   ├── dns.ts          DNS-over-HTTPS resolution + ownership verification
│   │   ├── monitor.ts      health checks, uptime maths, retention pruning
│   │   ├── router.ts       tiny path router with :params
│   │   └── http.ts         escaping, HTML templating, cookie helpers
│   ├── ui/                 design system (CSS, icons, layout, components)
│   └── routes/             panel pages + JSON API
├── local/                  Node preview runtime (sqlite + folder KV)
├── scripts/
│   ├── bundle.mjs          `npm run build`: regenerate schema + bundle check into dist/
│   ├── build-schema.mjs    schema.sql → schema.generated.ts
│   └── make-bundle.mjs     zip a folder into an uploadable site bundle
├── wrangler.toml
└── package.json
```

## Run it locally (no Cloudflare account)

```bash
npm install
npm run local            # http://localhost:8787/_panel
```

Node 22.18+ runs the TypeScript directly through its built-in type stripping.
Storage is `node:sqlite` (a D1 look-alike in `local/d1.mjs`) and a folder for KV
(`local/kv.mjs`), so the *same* worker source serves the preview. Data lands in
`local/data/` — delete that folder to start over.

## Deploy to Cloudflare's free plan

```bash
npm install
npx wrangler login
npx wrangler secret put PANEL_SETUP_TOKEN   # optional, strongly recommended
npx wrangler deploy
```

Wrangler may create the D1 database and KV namespace on the first run. If a
resource already exists but Wrangler tries to provision it again, use
`npx wrangler kv namespace list` and `npx wrangler d1 list` to find its ID,
then set `id` (KV) and `database_id` (D1) in `wrangler.toml` to bind the
existing resources. Open `https://<your-worker>.workers.dev/_panel` and
complete the installer.

Tested bundle: **254 KiB uploaded, 64 KiB gzipped** — well inside the free
Worker size limit.

### Configuration (optional Worker variables)

| Variable | Default | Purpose |
| --- | --- | --- |
| `PANEL_PATH` | `/_panel` | Where the panel lives. Change it to hide the panel at a non-obvious path. |
| `PANEL_HOST` | *(any host)* | Serve the panel only on this hostname, e.g. `panel.example.com`. |
| `PANEL_SETUP_TOKEN` | *(unset)* | Required by the installer when set. Use a secret, not a plain var. |
| `PANEL_NAME` | `AMS Cloud Panel` | Brand shown in the sidebar and login screen. |
| `KDF_ITERATIONS` | `100000` | Lower to `50000` if you ever hit the free plan's 10 ms CPU cap at login. |
| `INSECURE_COOKIES` | *(unset)* | Set to `1` only for local http previews. |

## What the panel does

**Sites** — unlimited registrations, each with its own admin slug, `DB_PASSWORD`,
origin URL, health URL, repo reference, notes, status (live / paused /
maintenance) and one-click credential rotation.

**Domains** — primary plus unlimited aliases, live DNS-over-HTTPS checks, and a
TXT ownership token (`_ams-verify.<host>`) so you can prove a domain is yours
before trusting it.

**Serving** — two modes, chosen per site:
- *proxy*: forward to an existing Node CMS origin (the AMS CMS `Hono` server),
- *bundle*: upload a zip once; the Worker serves it straight out of KV with SPA
  fallback, per-type caching, and immutable caching for hashed assets.

**Deployments** — versioned bundles with instant activate, rollback to the
previous release, and SHA-256 fingerprints.

**Backups** — one click snapshots config + env vars + domains + pages + the
active deployment reference into a zip in KV. Downloadable, restorable in place.
There's also a whole-panel JSON export.

**Monitoring** — health checks on demand and every 10 minutes via the cron
trigger, 200 checks retained per site, latency sparklines, 24 h uptime, and a
per-site history page.

**Content** — pages (Markdown) and media uploads, both stored in D1/KV and
scoped per site, so the panel is a small CMS in its own right.

**Team & security** — owner / admin / viewer roles, PBKDF2-SHA256 passwords with
per-user salts, session rows in D1 (revocable), per-session CSRF tokens, login
throttling by email and IP, and a 90-day audit log.

**API** — `Bearer ams_…` keys with `read` / `write` scopes against
`/_panel/api/overview`, `/api/sites`, `/api/sites/:id/check`,
`/api/monitor/feed`. There is also `/_panel/healthz` for uptime monitors.

## Design highlights

Built for the free plan:

1. **Host-based multi-tenancy in one Worker.** No per-site process, no Nginx:
   the Host header picks the site, the panel path is reserved.
2. **Static hosting from KV.** R2 needs a card; a deployment zip in KV plus an
   in-Worker ZIP reader gives you the same result for typical sites.
3. **Ownership verification, not just resolution.** DNS resolving is not proof of
   control; the TXT token is.
4. **Rollback as a first-class action** rather than a file rename.
5. **Free-plan guardrails in the UI** — 24 MB upload caps, 1,000 KV writes/day
   counter-style warnings, check-history trimming, audit pruning.
6. **An audit trail with IP + actor** so a shared panel is still accountable.
7. **API keys** so CI can trigger a health check or read the fleet without a
   browser session.
8. **A preview banner** on proxied and bundled sites so you always know you are
   looking at a draft, not production.

## Free-plan limits this design respects

| Resource | Free plan | How the panel stays inside it |
| --- | --- | --- |
| KV value size | 25 MB | Uploads capped at 24 MB with a clear error |
| KV writes | 1,000/day | One write per upload/backup/deploy; none per request |
| D1 storage | 5 GB | Check history trimmed to 200 rows/site, audit pruned at 90 days |
| Worker CPU | 10 ms/request | Indexed lookups, no per-request KV writes, hashing only at login |
| Worker size | 10 MB (3 MB gzip) | ~254 KiB / 64 KiB gzip |

## Tests

```bash
npm test          # node --test tests/*.test.mjs
npm run typecheck # tsc --noEmit
```

The unit tests cover the ZIP reader/writer round-trip, password hashing, site
validation, DNS verdicts, and the router. The local server is the integration
test: `npm run local`, install, create a site, upload a bundle, hit the host.

## More documentation

- [`docs/DEPLOY.md`](docs/DEPLOY.md) — three deploy paths, custom domains, cron work, updates, cost.
- The panel's own **Docs** page renders the same deployment notes at runtime, with your real hostnames filled in, and can download them as Markdown.
