---
date: 2026-05-25
source: nicermd CF Workers setup
---

# Cloudflare preview deployments

For a CF Worker connected to a GitHub repo via Workers Git
integration, every push to a non-production branch can auto-deploy
to a stable preview URL. This is the lighter-weight alternative to
spinning up a staging environment or subdomain.

## URL pattern

```
https://<branch-with-slashes-replaced-by-hyphens>-<worker-name>.<account-subdomain>.workers.dev
```

For a hypothetical worker `myapp` under an account subdomain `acme`:

```
feature/auth   →  https://feature-auth-myapp.acme.workers.dev
fix/login      →  https://fix-login-myapp.acme.workers.dev
main           →  whatever custom domain you've bound to production
```

- Slashes in branch names get replaced with hyphens.
- The worker name and account subdomain are fixed per project; find
  yours at the top of `Workers & Pages → <worker> → Settings`.
- Each branch's URL is stable across pushes — re-pushing the same
  branch overwrites the build at the same URL.

## Dashboard config

In **Workers & Pages → `<worker>` → Settings → Build**:

- **Production branch**: `main` (only this triggers the custom-domain deploy)
- **Preview branches**: All non-production branches (gets you the
  per-branch URLs above)
- **Build pull requests from forks**: OFF by default. Safer
  principle (don't auto-deploy strangers' code). Flip on once you
  have external contributors AND have confirmed no env vars /
  secrets are in any build scope.

## Build command setup (monorepo with `packages/website`)

For a pnpm workspace where the worker lives in a subdirectory:

| Field | Value |
|-------|-------|
| Root directory | `packages/website` |
| Build command | `bash cf-build.sh` |
| Deploy command | `npx wrangler deploy` |

Where `packages/website/cf-build.sh` climbs to workspace root:

```bash
#!/usr/bin/env bash
set -euo pipefail
corepack enable
cd "$(dirname "$0")/../.."
pnpm install --frozen-lockfile
pnpm -r build
```

Why a script and not inline: the CF dashboard's build-command field
runs in `dash`, which splits multi-line `&&` chains and chokes on
lines beginning with `&&`. Single-line `bash cf-build.sh` keeps the
multi-step build readable and lives in version control.

## Gotcha: build config does NOT survive GitHub repo transfer

If you transfer a repo between owners (e.g. personal account →
org), the CF GitHub App auth carries over (push events still fire)
but the **build configuration is wiped** back to defaults. Symptom:
deploy fails with "Missing entry-point to Worker script or to
assets directory" because wrangler runs from repo root and never
sees the monorepo structure.

**Mitigation**: note the three Build fields above somewhere private
(e.g. an iplaybook entry) so you can re-enter them after a
transfer. Re-deploy fires automatically once the config is saved.

## Verifying a deploy went through

For a Vite-built SPA where bundle filenames carry a content hash:

```bash
curl -s https://<your-host>/ | grep -oE 'index-[A-Za-z0-9_-]+\.js'
```

Returns the hashed bundle filename. A successful new deploy changes
this hash; polling for it is the cleanest signal that a push went
all the way through (CF→build→deploy→CDN propagation).

Works equally on a custom production domain or a preview URL.
