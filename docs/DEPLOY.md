# Deploying AMS Cloud Panel

Three paths, from quickest to most controlled.

## A. Zero-configuration free deploy

```bash
npm install
npx wrangler login
npx wrangler deploy
```

Wrangler can provision missing resources on the first deploy. If a namespace
with the configured title already exists (for example, after a partially
completed first deploy or when deploying through Workers Builds), Wrangler may
not be able to adopt it automatically. In that case, bind the existing
resources explicitly: run `npx wrangler kv namespace list` and
`npx wrangler d1 list`, then set the KV namespace `id` and D1 `database_id` in
`wrangler.toml` for the matching resources. Do not create another namespace
with the same title. Open `https://hosting-panel-cms.<subdomain>.workers.dev/_panel`
and complete the installer.

Before you deploy, decide whether to lock the installer:

```bash
npx wrangler secret put PANEL_SETUP_TOKEN   # paste a long random string
```

Without it, whoever opens the fresh Worker first becomes the owner. With it, the
installer requires the token.

## B. Deploy from the Cloudflare dashboard (no CLI)

1. Create a D1 database named `ams_cloud_panel` and a KV namespace.
2. Create a Worker → *Connect to Git* or paste the bundle, and add the source.
3. Bindings: `DB` → the D1 database, `KV` → the namespace.
4. Variables: `PANEL_SETUP_TOKEN` (secret), optionally `PANEL_PATH`,
   `PANEL_HOST`, `PANEL_NAME`.
5. Deploy, then open `/_panel`.

If you upload a prebundled Worker instead of the source, build it with:

```bash
npx wrangler deploy --dry-run --outdir dist
# dist/index.js is the deployable module
```

## C. Local only

```bash
npm run local     # http://localhost:8787/_panel, storage in local/data/
```

Node 22.18+ runs the TypeScript directly. The local runner binds `0.0.0.0` so it
also works inside a container or a remote preview.

## Attaching customer sites

**Cloudflare-hosted domains (best):** add each zone to the same account, then
give the Worker a *Custom Domain* or a route (`example.com/*`). DNS is created
for you and traffic is proxied.

**Registrar elsewhere:** point a CNAME at the Worker hostname
(`ams-cloud-panel.<subdomain>.workers.dev`). Apex domains need CNAME flattening
or an A record from a provider that supports it.

**One Worker, many sites:** routing is by `Host` header, so nothing else is
needed. Add the hostname in *Domains*, press *Check DNS*, and add the suggested
`_ams-verify.<host>` TXT record to prove ownership.

## Serving content

| Mode | Setup | Best for |
| --- | --- | --- |
| Proxy | Site → *Origin URL* (e.g. `https://origin.example.net`) | An existing Node CMS, gradually moving to the edge |
| Bundle | *Deployments* → upload a zip of the built site | Static sites, docs, landing pages, SPAs |

Bundles are read out of KV on demand; a zip of a typical static site is well
inside the 24 MB cap. Use `node scripts/make-bundle.mjs ./dist out.zip` to build
one from a folder.

## Scheduled work

`wrangler.toml` registers `*/10 * * * *`. Every run:

- re-checks every live site's health URL and stores the result,
- prunes check history to the newest 200 rows per site,
- deletes audit rows older than 90 days and login rows older than 30,
- clears expired sessions.

## Updating the panel

```bash
git pull
npm install
npx wrangler deploy
```

Schema changes are additive and applied automatically on the first request
(`src/db/index.ts` → `runAdditiveMigrations`). Existing data is never dropped.

## Rolling back the panel itself

`npx wrangler rollback` returns the Worker to the previous version. Panel data
lives in D1/KV, so it is untouched.

## Cost

Everything above is inside the Workers **Free** plan: 100k requests/day, D1 with
5 GB storage, KV with 100k reads and 1,000 writes/day. The panel writes to KV
only on uploads, backups and deployments — never per request. Move to Workers
Paid ($5/mo) only when request volume or KV write counts demand it; nothing in
the code needs changing.
