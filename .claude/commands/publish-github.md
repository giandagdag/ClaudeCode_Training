---
description: Security-scan, push to GitHub, write README, set About, enable GitHub Pages, and add CI/CD
argument-hint: <owner/repo | https://github.com/owner/repo(.git)> [branch]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*), Bash(gh:*), Bash(curl:*), PowerShell
---

# Publish this project to GitHub

Repo info from the user: `$ARGUMENTS`

Do the steps below in order. **Step 1 (security scan) gates everything else.** It runs before any
push, even though the user listed it last, because a secret that has been pushed counts as leaked.
Follow `CLAUDE.md`'s hard constraints throughout. Never remove features or change app behaviour
to make a check pass. Report a failing check and stop.

## 0. Resolve inputs and preflight

1. Parse `$ARGUMENTS` into `OWNER`, `REPO`, and an optional `BRANCH` (default `main`). Accept
   `owner/repo`, `https://github.com/owner/repo`, or the same URL ending in `.git`. If
   `$ARGUMENTS` is empty, ask the user for the repo with AskUserQuestion. Do not guess it from the
   existing remote without confirming.
2. Check `gh auth status` and `git --version`. If `gh` isn't authenticated, stop and tell the user
   to run `gh auth login`.
3. Check whether the repo exists (`gh repo view OWNER/REPO`). If it doesn't, ask whether to create
   it as **public** or **private**. Note that GitHub Pages on a private repo requires a paid plan.
4. If the folder isn't a git repo, run `git init -b BRANCH`. Set or update `origin` to
   `https://github.com/OWNER/REPO.git`. If `origin` already points somewhere else, show both URLs
   and confirm before you change it.
5. Derive `PAGES_URL = https://OWNER.github.io/REPO/` (lower-case the owner). If `REPO` is
   `OWNER.github.io`, use `https://OWNER.github.io/`.

## 1. Security scan (blocking gate)

Scan every file that will be committed (`git ls-files` plus untracked files that aren't ignored)
**and** the existing git history (`git log -p --all`). Use Grep/Read, not ad-hoc network tools.

**A. Secrets and credentials.** Flag anything that matches the following:
- Private keys: `-----BEGIN [A-Z ]*PRIVATE KEY-----`
- Cloud and API tokens: `AKIA[0-9A-Z]{16}`, `ghp_|gho_|ghs_|ghu_|github_pat_`, `xox[abpr]-`,
  `sk-[A-Za-z0-9]{20,}`, `sk-ant-`, `AIza[0-9A-Za-z_-]{35}`, Stripe `sk_live_`/`rk_live_`
- Generic assignments: `(api[_-]?key|secret|token|passw(or)?d|client[_-]?secret)\s*[:=]\s*['"][^'"]{8,}`
- Credentials embedded in URLs: `https?://[^/\s:@]+:[^/\s@]+@`
- Sensitive files: `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.sqlite`, `*.log`,
  `.claude/settings.local.json`

**B. Personal data.** Flag these:
- Real email addresses anywhere, including the local git `user.email`. The only allowed ones are
  `YOUR_EMAIL@example.com`, `noreply@anthropic.com`, and `*@users.noreply.github.com`.
- A real email inside `FORMSUBMIT_ENDPOINT`. On a public repo and Pages site, anyone can read
  it. Recommend FormSubmit's random-string alias (`https://formsubmit.co/ajax/<alias>`) instead.
- Phone numbers, real names of customers, or account numbers in seed data.

**C. Outbound network calls in `index.html`.** The only allowed destination is FormSubmit's AJAX
endpoint (`formsubmit.co`). Grep for `fetch(`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`,
`EventSource`, `import(`, `<script src`, `<link href`, `<img src`, `<iframe`, `url(http`,
`@import`, and any `http://` or `https://` literal. Every hit must be FormSubmit, an `xmlns`/SVG
namespace, or a plain non-loading link in text. Also make sure `notifyNewTask` still refuses to
call the network while the placeholder email is present.

**D. Brief constraints.** Confirm there's no `localStorage`, `sessionStorage`, `indexedDB`,
`document.cookie`, `!important`, CDN, web font, or real UOB branding.

**E. Ignore file.** Create or extend `.gitignore` with `.env*`, `*.pem`, `*.key`, `*.log`,
`.DS_Store`, `Thumbs.db`, `node_modules/`, `_site/`, and `.claude/settings.local.json`.

**Result.** Print a table with one row per check (A–E): PASS, or FAIL with `file:line`. If any check
FAILs, **stop**. Show the findings and ask the user how to proceed. If a secret is in history, tell
the user to rotate it and offer to rewrite history; never force-push without explicit approval.
Don't print a full secret value. Show only the first 4 characters.

## 2. README.md (create or edit)

Read the existing `README.md` and keep anything the user wrote that's still accurate. Then make
sure it contains the following:

1. Title and a one-line description (a demo Kanban board for an IT PMO; no real bank branding).
2. **Badges** (shields.io, static images, no tracking scripts):
   - HTML5, CSS3, and JavaScript (vanilla) tech badges
   - `No dependencies` and `No build step`
   - CI status: `https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg`
   - Pages deploy status, linking to `PAGES_URL`
   - License badge, only if a LICENSE file exists. Don't invent one; ask the user if they want one.
3. **Live demo** link to `PAGES_URL`, followed by the screenshot from step 2a:
   `![IT PMO Kanban board](docs/screenshot.png)`.
4. **Features**: board columns, drag/drop plus the keyboard "Move ▸" menu, filters, overdue
   badges, the Add Task modal with an email notification, and no persistence (resets on refresh).
5. **Architecture**, based on `CLAUDE.md`:
   - A Mermaid diagram of the one-way flow: user action → mutate `state` → `renderBoard()` →
     `renderCard()` → DOM, plus the `handleSubmit` → `validateForm` → `addTask` → `notifyNewTask`
     (FormSubmit) path.
   - A short explanation of `state = { tasks, filters, ui }`, event delegation on `#board`,
     `escapeHtml()`, task ID format, and overdue logic.
   - A project-structure tree (include `docs/screenshot.png`).
6. **How to install / run**: `git clone`, then open `index.html` in a browser. No server, npm, or
   build is needed. Include optional email setup: replace the placeholder in `FORMSUBMIT_ENDPOINT`
   with a FormSubmit alias, plus a warning not to commit a personal email.
7. **CI/CD**: what `ci.yml` checks and how Pages deploys.
8. **Security and privacy**: the only network call is FormSubmit, there's no storage, and all
   user input is escaped.

### 2a. Screenshot for the README (Playwright MCP)

Capture the app with the Playwright MCP server (configured in `.mcp.json`) and save it to
`docs/screenshot.png`:

1. `browser_resize` to 1440×900.
2. `browser_navigate` to `file:///<absolute path to>/index.html` (forward slashes). If the server
   refuses `file://` URLs and `PAGES_URL` already returns 200, navigate to `PAGES_URL` instead.
3. `browser_wait_for` the board columns (e.g. text "Backlog") so the seed cards have rendered.
4. `browser_take_screenshot` (viewport, PNG) with `filename` set to the absolute path of
   `docs/screenshot.png`, then `browser_close`. If the MCP saves elsewhere, move the file there.
5. Read the image back and check it shows the seeded board: no open modal, toast, or error, and
   no real email or personal data on screen. Take it on a fresh load so only seed data appears.

If the Playwright MCP server is unavailable, tell the user, then fall back to the Playwright CLI
using an installed browser (no browser download):
`npx -y playwright@latest screenshot --channel msedge --viewport-size "1440,900" --wait-for-selector ".column" "file:///<path>/index.html" docs/screenshot.png`
(use `--channel chrome` if Edge isn't installed). If both fail, skip the screenshot, leave the
README without it, and list it under "needs the user's action" in the final report.

The image lives only in the repo for the README. It isn't part of the app, so it doesn't break
the "no image files" constraint, and Pages still deploys only `index.html`.

## 3. CI/CD with GitHub Actions

Read any existing workflows in `.github/workflows/` first and merge into them instead of duplicating.

Create `.github/workflows/ci.yml` (name: `CI`). It triggers on `push` to any branch,
`pull_request`, and `workflow_dispatch`, with `permissions: contents: read`. Add these jobs:

- **checks** (ubuntu-latest), shell steps only, no npm:
  - `index.html` exists and isn't empty.
  - A grep guard fails the build on `localStorage|sessionStorage|indexedDB|document\.cookie`, on
    `!important`, or on `<script[^>]+src=` / `<link[^>]+href=["']https?:`.
  - A network-allowlist guard: extract every `https?://` URL from `index.html` and fail on any
    host other than `formsubmit.co` and `www.w3.org`.
  - A guard that fails if `FORMSUBMIT_ENDPOINT` contains an `@` that isn't
    `YOUR_EMAIL@example.com`.
- **secret-scan**: `actions/checkout@v4` with `fetch-depth: 0`, then `gitleaks/gitleaks-action@v2`
  with `GITHUB_TOKEN`. If gitleaks needs a license for an org-owned repo, use
  `trufflesecurity/trufflehog@v3.88.0` with `--only-verified` instead.

Update `.github/workflows/pages.yml` so that deployment happens only after CI passes. Either
trigger it with `workflow_run` on `CI` completed on `BRANCH` with
`if: github.event.workflow_run.conclusion == 'success'`, or move the deploy into `ci.yml` as a job
with `needs: [checks, secret-scan]` and `if: github.ref == 'refs/heads/BRANCH'`. Prefer the
single-file approach and delete `pages.yml` if you do. Keep `permissions` minimal: `pages: write`
and `id-token: write` only on the deploy job. Keep staging to `_site/` with only `index.html`, so
nothing else in the repo is published. Pin actions to major versions (`checkout@v4`,
`configure-pages@v5`, `upload-pages-artifact@v3`, `deploy-pages@v4`).

## 4. Commit and push

1. Run `git status` and show the user the list of files that will be committed. Never use
   `git add -A` blindly. Add only the intended files.
2. Commit with a clear message ending in the attribution lines from the current system reminder
   (if any).
3. `git push -u origin BRANCH`. If the push is rejected because the remote has commits, run
   `git pull --rebase` and retry. **Never** `--force` unless the user explicitly says so.

## 5. Enable GitHub Pages (Actions source)

- Check `gh api repos/OWNER/REPO/pages`. If it returns 404, create it with
  `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`. If it exists with
  `build_type: legacy`, switch it with `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`.
- Watch the run: `gh run list --repo OWNER/REPO --limit 3`, then `gh run watch <id> --exit-status`.
  If the first deploy ran before Pages was enabled and failed, re-run it with
  `gh workflow run ci.yml --repo OWNER/REPO --ref BRANCH`.
- Verify the site with `curl -sS -o /dev/null -w "%{http_code}" PAGES_URL`. Expect 200; allow a
  few minutes for propagation.

## 6. GitHub About section (description, website, topics)

```
gh repo edit OWNER/REPO \
  --description "Single-file vanilla HTML/CSS/JS Kanban board demo for an IT PMO. No dependencies, no build." \
  --homepage "PAGES_URL" \
  --add-topic kanban --add-topic vanilla-javascript --add-topic html --add-topic css \
  --add-topic github-pages --add-topic project-management --add-topic demo
```

Keep the description at 350 characters or fewer, with no real bank branding. If the user already
has a description, show the old and new versions and edit rather than overwrite. Confirm with
`gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`.

## 7. Final report

Give a short summary that includes the following:
- The security scan table (A–E) and any warnings, such as the FormSubmit email exposure.
- Commit SHA and branch pushed.
- CI run result with a link, and the Pages URL with its HTTP status.
- The About section as set.
- Anything skipped or needing the user's action (for example, LICENSE, FormSubmit alias
  activation email, Pages first-deploy delay).
