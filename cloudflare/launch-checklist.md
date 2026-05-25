---
date: 2026-05-25
---

# Cloudflare launch checklist

Dashboard items to verify when shipping a Cloudflare-hosted site.
Most of these are not API-settable on free accounts; check via the
dashboard.

## SSL / TLS

- [ ] **SSL/TLS mode: Full (strict).** Flexible leaves the CF→origin
      leg in cleartext and is a common footgun.
- [ ] **Edge Certificates → Always Use HTTPS: ON.**
- [ ] HSTS: prefer setting `Strict-Transport-Security` in your
      `_headers` file rather than the dashboard toggle, to keep one
      source of truth. `preload` requires submitting to
      [hstspreload.org](https://hstspreload.org).

## Account hygiene

- [ ] **Account 2FA enforced.** CF account compromise = total
      takeover (DNS + serving). Higher impact than GitHub repo
      compromise. Worth a hardware key.

## Bot protection

- [ ] **Bot Fight Mode: ON** (Security → Bots, free tier).
- [ ] **Super Bot Fight Mode: OFF.** Paid feature that aggressively
      blocks crawlers — also blocks search engines and breaks
      discoverability of your site.

## Workers Builds / Pages auto-deploy

If the site auto-deploys from GitHub via Workers Builds:

- [ ] Confirm production deploy is gated on `main` (or your chosen
      branch) only, not every branch.
- [ ] Verify fork PRs don't expose env vars or secrets to preview
      builds. Settings → Builds → environment variables. Treat
      preview builds as if a stranger can read whatever's in scope.
- [ ] If you've set up a custom domain, confirm the production
      worker (not a preview) is the one bound to it.

## Caching: `_headers` patterns

For static SPAs deployed via Workers Static Assets:

- **HTML is forced to `max-age=0, must-revalidate`** regardless of
  what you put in `_headers`. This is correct: content-hashed assets
  rely on a fresh HTML pointing at the right hash.
- **Hashed assets (`/assets/*`)**: cache aggressively. Single biggest
  egress saving for a traffic spike:
  ```
  /assets/*
    Cache-Control: public, max-age=31536000, immutable
  ```
- **Service worker (`/sw.js`)**: never cache, so updates ship promptly:
  ```
  /sw.js
    Cache-Control: no-cache
  ```
- **Brand assets (favicons, OG images)**: medium-long cache; new ones
  ship by changing the filename or waiting out the cache:
  ```
  /favicon.ico
    Cache-Control: public, max-age=604800
  ```

## Free-tier limits

Workers Static Assets serves files for free (unlimited requests,
unlimited bandwidth) when there's no Worker code in the deployment.
Only Worker *invocations* count against the 100k/day quota. A
pure-SPA deploy with no `main` in `wrangler.jsonc` is essentially
uncapped.

If you later add Worker code:

- Free plan: 100k requests/day; throttles past that until the daily
  window resets.
- Paid plan ($5/mo): 10M requests included, $0.50 per additional
  million. Scales with cost, doesn't block.

## Rules / leftover state

- [ ] Page Rules: none leftover from earlier setup that override
      your `_headers`. Rules → Page Rules.
- [ ] Cache Rules: same. Rules → Cache Rules.
- [ ] Transform Rules: same.

## Domain transfer gotcha

If you ever transfer a GitHub repo to a new owner, the CF Workers
Builds dashboard config (build command + root directory + deploy
command) **gets wiped**. Note your current config somewhere private
before transferring. Typical config:

- Root directory: `packages/website` (or wherever your `wrangler.jsonc` lives)
- Build command: `bash cf-build.sh` (a script in your repo)
- Deploy command: `npx wrangler deploy`

## ToS sanity

Cloudflare §2.8 historically blocked using their bandwidth as a
generic file / video CDN on free plans. Doesn't apply to a regular
SPA serving its own assets. If your content is primarily large
binaries or video, host them elsewhere (R2, S3) and serve via your
CF site.
