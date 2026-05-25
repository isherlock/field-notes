---
date: 2026-05-25
---

# GitHub public-repo launch checklist

Before announcing a public repo, sweep these settings. GitHub ships
public repos with weak defaults — most need fixing one by one.

Replace `<owner>/<repo>` and `<owner>` placeholders throughout.

## Settings

### Security: secret scanning + push protection

```bash
gh api repos/<owner>/<repo> --method PATCH \
  -f 'security_and_analysis[secret_scanning][status]=enabled' \
  -f 'security_and_analysis[secret_scanning_push_protection][status]=enabled'
```

Note: `secret_scanning_non_provider_patterns` may be plan-gated and
silently refuse to enable via REST. Provider patterns + push
protection cover the main threat; toggle non-provider from
`https://github.com/<owner>/<repo>/settings/security_analysis` if
available.

### Repo defaults

```bash
gh api repos/<owner>/<repo> --method PATCH \
  -F delete_branch_on_merge=true \
  -F allow_squash_merge=true \
  -F allow_merge_commit=false \
  -F allow_rebase_merge=false \
  -F allow_update_branch=true \
  -F has_discussions=true \
  -F has_wiki=false
```

### Branch protection on `main`

For solo repos — required PR + named status check, no required reviewers.
First, list the available check names:

```bash
gh api repos/<owner>/<repo>/commits/main/check-runs --jq '.check_runs[].name' | sort -u
```

Then apply protection (replace `build` with the actual CI check name):

```bash
cat > /tmp/protection.json <<'EOF'
{
  "required_status_checks": {"strict": true, "contexts": ["build"]},
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "required_conversation_resolution": false
}
EOF
gh api repos/<owner>/<repo>/branches/main/protection --method PUT --input /tmp/protection.json
rm /tmp/protection.json
```

### Tag protection ruleset

Block deletion + update on release tags so a compromise can't move
tags to a different commit. Admin can still create new tags.

```bash
cat > /tmp/ruleset.json <<'EOF'
{
  "name": "Protect release tags",
  "target": "tag",
  "enforcement": "active",
  "bypass_actors": [
    {"actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always"}
  ],
  "conditions": {
    "ref_name": {"include": ["refs/tags/v*"], "exclude": []}
  },
  "rules": [
    {"type": "deletion"},
    {"type": "update"}
  ]
}
EOF
gh api repos/<owner>/<repo>/rulesets --method POST --input /tmp/ruleset.json
rm /tmp/ruleset.json
```

Adjust the tag pattern (`refs/tags/v*`) to match your release tag
convention (`tauri-v*`, `chrome-ext-v*`, etc.).

## Files to add

- `.github/CODEOWNERS` — `* @<owner>` so PRs auto-request review
- `SECURITY.md` — private-vulnerability-reporting link + threat model
- `CONTRIBUTING.md`
- `CODE_OF_CONDUCT.md`
- `LICENSE` — MIT is a safe default for most public OSS
- `.github/ISSUE_TEMPLATE/{bug_report.yml,feature_request.yml,config.yml}` — YAML issue forms beat .md templates
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/dependabot.yml` — npm + actions at minimum

## Verify

```bash
gh api repos/<owner>/<repo> --jq '{
  allow_squash_merge,
  allow_merge_commit,
  allow_rebase_merge,
  delete_branch_on_merge,
  allow_update_branch,
  has_discussions,
  has_wiki,
  security_and_analysis
}'

gh api repos/<owner>/<repo>/branches/main/protection --jq '{
  required_status_checks,
  allow_force_pushes,
  allow_deletions,
  enforce_admins
}'

gh api repos/<owner>/<repo>/rulesets --jq '.[] | {name, target, enforcement}'
```

## What I deliberately don't enable on solo repos

- `web_commit_signoff_required` (DCO sign-off) — friction without
  meaningful gain on a solo repo. Worth it for projects expecting
  enterprise contributors.
- `required_signatures` (GPG-signed commits) — same trade-off.
- Required approving reviewers > 0 — would block self-merging.

## Workflow implications

- Direct `git push origin main` no longer works once branch
  protection is on. PR-only flow becomes mandatory.
- Release-tag pushes need an admin context if you set tag protection.
- The named CI check becomes the gate — if you rename your CI job,
  also update `required_status_checks.contexts`.
