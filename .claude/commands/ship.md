---
description: Security-scan, push to GitHub, deploy Pages via Actions, and update the README and About section
argument-hint: [repo-url] (omit to use the existing origin)
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion
---

Ship this project to GitHub: scan it for anything that must not go public, push it,
deploy it as a GitHub Pages site via GitHub Actions, and fill in the README and the
repo's About section.

Target repo URL (may be empty — if so, use the existing `origin` remote): `$1`

Work through the phases in order. **Phase 1 is a hard gate: nothing leaves the machine
until it passes.** Report what you find at each phase rather than silently continuing.

---

## Environment facts (verified on this machine — don't rediscover them)

- **`gh` CLI is NOT installed.** Use `git` plus `curl` against `https://api.github.com`.
- **Push over SSH.** The key at `~/.ssh/id_ed25519` is registered and works:
  `ssh -T git@github.com` returns `Hi jonbowden!`. Prefer SSH remotes
  (`git@github.com:USER/REPO.git`) over HTTPS.
- **The token in `~/.git-credentials` is a fine-grained PAT with no admin rights**, scoped
  to a fixed allowlist of repos. It **will** return `403 Resource not accessible` or `404
  Not Found` for: creating repos, changing visibility, enabling Pages, and editing the
  About section. This is expected — fall back to the browser steps given below rather than
  retrying. Never print the token; read it into a shell variable only:
  ```bash
  TOKEN=$(grep 'github.com' ~/.git-credentials | head -1 | sed -E 's#^https://[^:]+:([^@]+)@.*#\1#')
  ```
- **Verifying a rendered page:** Chrome is at
  `/mnt/c/Program Files/Google/Chrome/Application/chrome.exe` and needs Windows-style
  paths (`file:///C:/...`). Use `--headless --disable-gpu --virtual-time-budget=8000
  --dump-dom` or `--screenshot='C:\path\out.png'`.

---

## Phase 1 — Security scan (blocking)

Publishing to GitHub is irreversible in practice: once content is public it can be cloned,
cached and indexed within minutes, and making the repo private afterwards does not retract
it. Scan **both the working tree and the full git history** — a secret removed in a later
commit is still public in an earlier one.

Run each of these from the repo root. `.git` is excluded because packfiles produce noise.

```bash
# 1a. Credential-shaped strings in tracked files
grep -rInE --exclude-dir=.git \
  '(gh[pousr]_[A-Za-z0-9]{16,}|github_pat_[A-Za-z0-9_]{20,})' . || echo "  clean: github tokens"
grep -rInE --exclude-dir=.git '(AKIA|ASIA)[0-9A-Z]{16}' . || echo "  clean: aws keys"
grep -rInE --exclude-dir=.git 'sk-(ant-)?[A-Za-z0-9_-]{20,}' . || echo "  clean: llm api keys"
grep -rInE --exclude-dir=.git 'AIza[0-9A-Za-z_-]{35}' . || echo "  clean: google api keys"
grep -rInE --exclude-dir=.git 'xox[baprs]-[A-Za-z0-9-]{10,}' . || echo "  clean: slack tokens"
grep -rInE --exclude-dir=.git -- '-----BEGIN [A-Z ]*PRIVATE KEY-----' . || echo "  clean: private keys"
grep -rInE --exclude-dir=.git \
  '(password|passwd|secret|api[_-]?key|access[_-]?token|client[_-]?secret)[[:space:]]*[:=][[:space:]]*["'"'"'][^"'"'"']{6,}' . \
  || echo "  clean: assigned secrets"
grep -rInE --exclude-dir=.git \
  '(mongodb(\+srv)?|postgres(ql)?|mysql|redis)://[^/[:space:]]*:[^@[:space:]]+@' . \
  || echo "  clean: connection strings with credentials"

# 1b. Files that should never be committed
find . -path ./.git -prune -o \
  \( -name '.env*' -o -name '*.pem' -o -name '*.key' -o -name '*.pfx' -o -name '*.p12' \
     -o -name 'id_rsa*' -o -name 'id_ed25519' -o -name '.git-credentials' \
     -o -name 'credentials.json' -o -name 'service-account*.json' \) -print

# 1c. Same sweep across all history, not just HEAD
git log -p --all 2>/dev/null | grep -nE \
  '(gh[pousr]_[A-Za-z0-9]{16,}|github_pat_|AKIA[0-9A-Z]{16}|sk-(ant-)?[A-Za-z0-9_-]{20,}|-----BEGIN [A-Z ]*PRIVATE KEY-----)' \
  || echo "  clean: git history"

# 1d. Personal / internal data — review each hit by eye, do not auto-clear
grep -rInE --exclude-dir=.git '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' . | grep -v 'noreply@' || echo "  no emails"
grep -rInE --exclude-dir=.git '\b(10|192\.168|172\.(1[6-9]|2[0-9]|3[01]))\.[0-9]+\.[0-9]+\b' . || echo "  no private IPs"

# 1e. What is actually about to be published
git status --porcelain
git ls-files
```

Then judge, don't just pattern-match:

- **Real names, customer data, or internal hostnames in seed/demo data?** Demo fixtures
  should use invented names. Flag anything that looks like it came from a real system.
- **Personal email addresses?** Commits should be authored with the GitHub noreply address
  (`ID+user@users.noreply.github.com`), not a personal one — check
  `git log -1 --format='%ae'`. On a public repo the author email is permanently visible.
- **Does the project imitate a real organisation?** If the code carries a real company's
  name, branding or internal-system framing, say so plainly before publishing publicly.
  A disclaimer in the source reduces but does not remove the risk.
- **Anything the user described as internal or confidential** stays unpublished regardless
  of what the greps say.

**Report every finding and stop.** Ask the user how to proceed with `AskUserQuestion` —
offering to remove the item, to keep it and continue, or to abort. Only continue on an
explicit answer. If the scan is completely clean, say so in one line and continue.

---

## Phase 2 — Push

1. Determine the remote:
   - If `$1` was given, normalise it to SSH form (`git@github.com:USER/REPO.git`) and set it:
     `git remote add origin <url>` or `git remote set-url origin <url>`.
   - Otherwise use the existing `git remote get-url origin`. If there is no origin and no
     argument, stop and ask for the URL.
2. If this is not a git repo yet: `git init -b main`, and write a `.gitignore` that covers
   build output, `.env*`, and any temp/verification artifacts this project generates.
3. Set the commit identity to the GitHub noreply address **for this repo only**, so a
   personal address is never baked into public history:
   ```bash
   git config user.email "$(git log -1 --format='%ae' 2>/dev/null | grep noreply || echo 'ID+USER@users.noreply.github.com')"
   ```
   If the numeric ID is unknown, get it from an existing repo's commits via the API, or ask.
4. Stage, commit with a message describing what actually changed, and push:
   `git push -u origin main`.
5. **Confirm the push landed** — compare `git rev-parse HEAD` against
   `git ls-remote origin main`. Do not report success on the basis of exit code alone.

If the push fails with `403 Write access to repository not granted`, the HTTPS token
doesn't cover this repo — switch the remote to SSH and retry.

---

## Phase 3 — Deploy Pages via GitHub Actions

Use a workflow rather than the "deploy from a branch" setting: `actions/configure-pages`
can enable Pages using the workflow's own `GITHUB_TOKEN`, which sidesteps the PAT's missing
admin rights.

Create `.github/workflows/pages.yml` (or edit it if it exists — preserve any customisation):

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
        with:
          enablement: true
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .
      - id: deployment
        uses: actions/deploy-pages@v4
```

Adjust `path:` if the site is not served from the repo root (e.g. `./dist` after a build —
add the build step before `upload-pages-artifact`).

Commit and push it, then poll until the site actually serves:

```bash
for i in $(seq 1 10); do
  CODE=$(curl -sS -o /dev/null -w '%{http_code}' https://USER.github.io/REPO/)
  echo "attempt $i: HTTP $CODE"; [ "$CODE" = "200" ] && break; sleep 20
done
```

Then verify it is serving the **right** file, not a stale or placeholder build: compare the
live `md5sum` against the local one, and render it headlessly to confirm it works when
served over `https://` rather than `file://`.

If `enablement: true` is refused (some org policies block it), fall back to telling the user:
`https://github.com/USER/REPO/settings/pages` → Source: **GitHub Actions** → Save, then
re-run the workflow.

---

## Phase 4 — README

Create or update `README.md`. If one exists, edit it rather than overwriting — keep any
content the user wrote.

Derive the content from what the code actually does; do not invent features, badges,
roadmaps, licences or contribution guidelines that don't exist. Aim for:

- One-line description of what it is, and a link to the live Pages URL
- How to run it locally (the genuine steps for this project)
- Any configuration the user must change themselves, named by file and line
- Constraints a contributor would otherwise violate — if the project has a `CLAUDE.md`,
  its constraints section is the source for this
- A note on anything deliberately unusual (e.g. state that resets on refresh by design)

If the project imitates or is themed around a real organisation, carry the disclaimer into
the README, since that is the first thing a visitor reads.

---

## Phase 5 — About section

Set the repo description, homepage and topics in one API call:

```bash
TOKEN=$(grep 'github.com' ~/.git-credentials | head -1 | sed -E 's#^https://[^:]+:([^@]+)@.*#\1#')
curl -sS -w '\nHTTP %{http_code}\n' -X PATCH \
  -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/USER/REPO \
  -d '{"description":"<one line>","homepage":"https://USER.github.io/REPO/","has_issues":true}'
```

Topics are a separate call:

```bash
curl -sS -X PUT -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/USER/REPO/topics -d '{"names":["topic-a","topic-b"]}'
```

**Expect this to fail with 403/404** on the current token. When it does, don't retry —
give the user the exact manual steps:

> `https://github.com/USER/REPO` → the **⚙︎** next to **About** (top right) →
> set **Description**, tick **Use your GitHub Pages website** (or paste the URL into
> **Website**), add topics → **Save changes**

Then confirm with an unauthenticated read, which works on public repos:

```bash
curl -sS https://api.github.com/repos/USER/REPO | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print('description:',d.get('description')); print('homepage:',d.get('homepage')); print('has_pages:',d.get('has_pages'))"
```

---

## Final report

Give the user, in this order:

1. **Security scan result** — what was checked, what was found, what was done about it
2. **Live URL** and **repo URL**, both verified reachable rather than assumed
3. **Commit pushed** (short SHA) and confirmation local matches remote
4. **Anything still requiring manual action**, with the exact URL and clicks
5. **Anything deliberately left undone**, and why

Never report a step as done when it was refused by a permission wall — say it was refused
and hand over the manual steps.
