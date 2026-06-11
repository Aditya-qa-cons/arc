---
name: arc-setup
description: One-time arc workflow setup — creates local config, verifies gh and gcloud auth, creates GitHub labels. Run on a new machine or when /arc-start reports missing config.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) -> **File -> Open Folder** -> select the repo folder, then retry `/arc-setup`.

# arc Workflow Setup

Run once on a new machine (or re-run any time to verify). Every step is idempotent — it checks before acting, so running it twice is safe.

## Step 0 â€" Verify project context

Before anything else, confirm Claude Code is running inside the correct repo root (not a worktree, not an unrelated folder).

```bash
# Check if workflow-config.json exists in the current directory
cat .claude/workflow-config.json 2>/dev/null | head -1
```

**If the file is found:** print `âœ… Project context â€" workflow-config.json found` and continue to Step 1.

**If the file is NOT found**, run an automatic search:

```powershell
# Search from the user's home directory for workflow-config.json inside a .claude folder
Get-ChildItem -Path $env:USERPROFILE -Filter "workflow-config.json" -Recurse -Depth 5 -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -like "*\.claude\*" } |
  Select-Object -First 1 |
  ForEach-Object { Split-Path $_.DirectoryName -Parent }
```

- **If a path is found:** open it in Windows Explorer so the user can drag it into Claude Code:
  ```powershell
  Start-Process explorer.exe "[FOUND_PATH]"
  ```
  Then stop and tell the user:
  ```
  âš ï¸  You're not inside the project folder.
  We found the repo at: [FOUND_PATH]
  We've opened it in Explorer â€" drag that folder into Claude Code (or use File â†' Open Folder),
  then re-run /arc-setup.
  ```

- **If no path is found:** stop and tell the user:
  ```
  âŒ  Could not find the repo on this machine.
  Clone it first:  git clone https://github.com/[repo].git
  Then open the cloned folder in Claude Code and re-run /arc-setup.
  ```
  (Use the `repo` value from workflow-config.json once it is available â€" for now use the literal GitHub URL format.)

**Worktree guard â€" also check if we're inside a git worktree instead of the main repo:**

```bash
git rev-parse --git-dir 2>/dev/null
```

If the output contains `worktrees` (e.g. `.git/worktrees/xxx`), we are inside a worktree. Get the main repo root:

```bash
# Strip /.git/worktrees/xxx to get the main repo path
git rev-parse --git-common-dir | sed 's|/\.git.*||' | sed 's|\\\.git.*||'
```

Open that path in Explorer and tell the user:
```
âš ï¸  You've opened a git worktree, not the repo root.
The main repo is at: [MAIN_REPO_PATH]
We've opened it in Explorer â€" drag that folder into Claude Code (or use File â†' Open Folder),
then re-run /arc-setup.
```

## Step 1 â€" Read project config

First, resolve the main repo root so all file paths work correctly from any worktree:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

Keep `MAIN_ROOT` in context — use it as a prefix for every `.claude/` file operation in all subsequent steps.

```bash
cat "$MAIN_ROOT/.claude/workflow-config.json"
```

If the file is missing, stop and tell the user: "`.claude/workflow-config.json` is missing. Run `/arc-init` to generate it first, then re-run `/arc-setup`."

Extract and keep in context for later steps:
- `repo` — e.g. `your-org/your-repo`
- `labels` object — all label names
- `dev_url` — e.g. `https://dev.example.com`

**Check for stale skills â€" confirm local repo is up to date:**

```bash
git fetch origin 2>/dev/null
git status -sb | head -3
```

If the output shows `behind` (e.g. `## develop...origin/develop [behind 3]`), print a warning:

```
âš ï¸  Your local repo is behind the remote â€" skills and config may be outdated.
    Run: git pull
    Then re-run /arc-setup to ensure you have the latest workflow.
```

If up to date or ahead, print: `âœ… Repo â€" up to date with remote`

## Step 2 â€" Create local config

Check if `.claude/workflow-config.local.json` already exists (using MAIN_ROOT from Step 1):

```bash
cat "$MAIN_ROOT/.claude/workflow-config.local.json" 2>/dev/null
```

**If it exists and has `gh_account` set**, show the current value and ask:

```
Current local config:
  gh_account: [current value]

Is this correct, or do you want to update it? (yes / new-handle)
```

If the user confirms it's correct, proceed. If they provide a new handle, rewrite the file preserving the JSON structure (keep `schema_version`) and confirm: "Updated `.claude/workflow-config.local.json` with `gh_account: [new handle]`."

**If it doesn't exist (or is missing `gh_account`)**, create it with a placeholder and ask the user to confirm their account name before continuing:

```bash
cat > "$MAIN_ROOT/.claude/workflow-config.local.json" << 'EOF'
{
  "schema_version": "1.0",
  "gh_account": "[your-github-handle]"
}
EOF
```

Ask the user: "What is your GitHub account handle? (e.g. `Aditya-qa-cons`) I'll update `.claude/workflow-config.local.json` with the correct value."

Once they confirm, replace `[your-github-handle]` with the actual account name. Confirm: "Created `.claude/workflow-config.local.json` with `gh_account: [confirmed account]`. This file is gitignored and stays local to this machine."

## Step 3 — Verify gh auth

```bash
gh auth status
```

Check which account is active. The active account should match `gh_account` from the local config.

**If correct account is active:** print `✅ gh auth — [gh_account] active`

**If wrong account is active:**
```bash
gh auth switch --user [gh_account]
gh auth status
```

If switching fails ("no accounts matched"), tell the user: "Run `gh auth login` and authenticate as `[gh_account]`, then re-run `/arc-setup`."

**Verify repo access:**
```bash
gh repo view [repo] --json name,defaultBranchRef --jq '{repo: .name, default_branch: .defaultBranchRef.name}'
```

If this fails, stop: "Cannot access `[repo]` — check that `[gh_account]` has repo access."

## Step 4 — Configure git remote to avoid multi-account credential prompts

On machines with more than one GitHub account stored in Git Credential Manager, `git push` and `git pull` may show an interactive account-selection dialog. This breaks autonomous operation (po-run can't click dialogs).

Fix: embed `gh_account` in the remote URL so GCM resolves credentials automatically without prompting.

```bash
# Check the current remote URL
git remote get-url origin
```

If the URL does **not** already contain `[gh_account]@` (e.g. it is `https://github.com/...`), update it:

```bash
git remote set-url origin https://[gh_account]@github.com/[repo].git
```

Then verify the push still works:

```bash
git ls-remote --heads origin HEAD
```

If it succeeds, print: `✅ git remote — origin set to https://[gh_account]@github.com/[repo].git`

If the URL already contains `[gh_account]@`, print: `✅ git remote — already configured correctly`

> **Note**: this must be repeated for any git worktrees created later, since each worktree's remote is independent. The `superpowers:using-git-worktrees` skill handles new worktrees — but if you have existing worktrees from before running po-setup, run `git remote set-url origin https://[gh_account]@github.com/[repo].git` inside each one manually.

### 4b — Activate shared git hooks

The repo ships a `hooks/pre-push` script that blocks direct pushes to `main` or `develop`. Activate it:

```bash
git config core.hooksPath hooks
git config core.hooksPath  # verify — should print "hooks"
```

If the output is `hooks`, print: `✅ git hooks — hooks/pre-push active (blocks direct pushes to main/develop)`

If the command fails (e.g. git version too old), print a warning and continue:
```
⚠️  Could not set core.hooksPath — git version may be too old (requires git 2.9+).
    Run manually: git config core.hooksPath hooks
```

## Step 5 — Verify gcloud auth

First check if `gcloud` is installed at all:

```bash
command -v gcloud >/dev/null 2>&1
```

**If gcloud is not found**, skip the auth check entirely and print a neutral note (don't stop):

```
ℹ️  gcloud not installed — skipped. Only needed for GCP deployments (Firestore index deploys, secrets).
    If you need it: https://cloud.google.com/sdk/docs/install
    Teammates with GCP access (e.g. Ben) handle deployments — this is safe to skip.
```

**If gcloud is found**, read `gcloud.config_name` from `.claude/workflow-config.json`. If that key is missing or the `gcloud` block is absent, skip this check (non-GCP project).

```bash
GCLOUD_CONFIG=$(python3 -c 'import json; c=json.load(open(".claude/workflow-config.json")); print(c.get("gcloud",{}).get("config_name",""))' 2>/dev/null)
[ -n "$GCLOUD_CONFIG" ] && CLOUDSDK_ACTIVE_CONFIG_NAME="$GCLOUD_CONFIG" gcloud config get account
```

If the command prints an email address, the config is configured and authenticated. Print: `✅ gcloud — [output] ($GCLOUD_CONFIG config)`

If the command returns an empty string or errors, print a warning (don't stop):

```
⚠️  gcloud installed but '[config_name]' config not found or not authenticated.
    To fix: CLOUDSDK_ACTIVE_CONFIG_NAME=[config_name] gcloud auth login
    Ask your GCP admin to help set up the gcloud config.
    (Setup can continue — gcloud is only needed for Firestore index deploys.)
```

## Step 6 — Create GitHub labels

Required labels and their colors:

| Label | Color | Description |
|---|---|---|
| `Merged - Check` | `#0075ca` | Fix merged, awaiting QA on dev |
| `Merged - Redo` | `#e4e669` | QA failed, needs re-fix |
| `QA Passed` | `#0e8a16` | QA verified on dev |
| `p0-critical` | `#b60205` | Blocker — production down |
| `priority-high` | `#d93f0b` | High priority fix |
| `po-run:active` | `#BFD4F2` | Claimed by an active po-run session — do not pick up |

Fetch **all** existing labels in a single call (use `--limit 1000`, the GitHub CLI maximum, to avoid pagination truncation):

```bash
EXISTING_LABELS=$(gh label list --repo [repo] --json name --jq '.[].name' --limit 1000)
```

For each required label, check against `$EXISTING_LABELS` and create only if missing:

```bash
# Check (-F for fixed strings so label names with special chars match literally)
echo "$EXISTING_LABELS" | grep -Fqx "LABEL_NAME"

# Create if missing
gh label create "LABEL_NAME" --repo [repo] \
  --color "COLOR_HEX_WITHOUT_HASH" --description "DESCRIPTION"
```

After processing all labels, print:
```
✅ GitHub labels — all required labels present
```

Or if any were created:
```
✅ GitHub labels — created: [list of new ones]
```

## Step 7 — Verify .gitignore entries

Check that these entries are present in `.gitignore`:

```bash
grep -E "qa-screenshots|workflow-config.local.json|run-log.jsonl|last-work-summary.json" "$MAIN_ROOT/.gitignore"
```

If any are missing, add the missing line(s) to `.gitignore`:

```bash
echo ".claude/qa-screenshots/" >> "$MAIN_ROOT/.gitignore"
echo ".claude/workflow-config.local.json" >> "$MAIN_ROOT/.gitignore"
echo ".claude/run-log.jsonl" >> "$MAIN_ROOT/.gitignore"
echo ".claude/last-work-summary.json" >> "$MAIN_ROOT/.gitignore"
```

Note: `run-log-*.jsonl` (per-operator logs) are intentionally NOT gitignored — they are committed to the repo so all teammates' runs are visible for analysis.

Print: `✅ .gitignore — local config, screenshots, and session files excluded`

## Step 8 — Check Playwright + Chromium

`/arc-verify-issue` uses `pw-page.mjs` and `pw-act.mjs` to run headless browser verification. These scripts require Playwright and the Chromium browser binary to be installed in `frontend/node_modules`.

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
node -e "
  const {createRequire} = require('module');
  const r = createRequire(require('path').join('$MAIN_ROOT', 'frontend/package.json'));
  const {chromium} = r('playwright');
  chromium.launch({headless:true})
    .then(b => b.close())
    .then(() => console.log('ok'))
    .catch(e => { console.error(e.message); process.exit(1); });
" 2>/dev/null && echo "pw-ok" || echo "pw-missing"
```

**If output contains `pw-ok`**, print:
```
✅ Playwright + Chromium — available
```

**If output contains `pw-missing`**, prompt the user before installing:

```
❌ Playwright Chromium browser not installed — required for /arc-verify-issue.

Install now? This downloads ~200MB (Chromium binary, one-time).
  Y — install automatically
  N — skip (install manually later: cd frontend && npx playwright install chromium)
```

If the user says **Y** (or yes/y), run:

```bash
cd [MAIN_ROOT]/frontend && npx playwright install chromium
```

Then re-run the Playwright check from above to confirm it succeeded. If it now prints `pw-ok`, print: `✅ Playwright + Chromium — installed and verified`

If the install fails, print the error output and tell the user:
```
❌ Playwright install failed — see error above.
    Try manually: cd frontend && npx playwright install chromium
    Setup continues — /arc-verify-issue will not work until this is resolved.
```

If the user says **N**, print:
```
⚠️  Playwright Chromium skipped — /arc-verify-issue will not work until installed.
    When ready: cd frontend && npx playwright install chromium
```

Do not stop setup — other checks are independent.

## Step 8a — Check Chrome MCP (optional, manual debugging only)

Chrome MCP is **not required** for `/arc-verify-issue` — that skill uses Playwright scripts for automated QA. Chrome MCP is useful for manual visual debugging of the dev environment when you want to inspect the UI interactively.

Try to auto-detect a live Chrome MCP connection by calling the `list_connected_browsers` tool from the `mcp__Claude_in_Chrome__*` server (load via ToolSearch if deferred: `select:mcp__Claude_in_Chrome__list_connected_browsers`).

**If the tool call succeeds and returns at least one browser**, print:
```
✅ Chrome MCP — connected ([N] browser(s) detected) — useful for manual debugging
```

**If the tool call succeeds but returns an empty list**, print:
```
ℹ️  Chrome MCP extension installed but no tab connected (optional — not needed for /arc-verify-issue).
```

**If the tool call fails or the MCP server is unavailable**, print:
```
ℹ️  Chrome MCP not detected (optional — only needed for manual visual debugging, not for /arc-verify-issue).
```

## Step 9 — Check .env.test.local (warning only)

Use `MAIN_ROOT` from Step 1 (not `git rev-parse --show-toplevel`, which returns the worktree path when run from a worktree):

```bash
ls "$MAIN_ROOT/frontend/.env.test.local" 2>/dev/null && echo "found" || echo "missing"
```

**If found:** print `✅ frontend/.env.test.local — present`

**If missing:** print a warning (don't stop):
```
⚠️  frontend/.env.test.local not found.
    This file is needed for /arc-verify-issue (QA credential lookups).
    Ask a teammate for the file — it contains test user passwords.
    (Setup continues — you can add it later.)
```

## Step 10 â€" Verify Node.js is installed

```bash
node --version 2>/dev/null && npm --version 2>/dev/null
```

**If both return version strings:** print `âœ… Node.js â€" [version] / npm [version]`

**If either is missing:** print a warning (don't stop):
```
âš ï¸  Node.js not found.
    Install it from https://nodejs.org (LTS version recommended).
    Node is required to run the dev server and unit tests.
    (Setup continues â€" install it before running npm commands.)
```

## Step 11 â€" Initialize agent-state.json

Check if `.claude/agent-state.json` exists and has valid JSON:

```bash
cat "$MAIN_ROOT/.claude/agent-state.json" 2>/dev/null
```

If it exists and has `schema_version` and `issues` array, print: `✅ agent-state.json — present`

If it's missing or invalid, create it:

```bash
cat > "$MAIN_ROOT/.claude/agent-state.json" << 'EOF'
{
  "schema_version": "1.0",
  "last_updated": "TIMESTAMP",
  "issues": []
}
EOF
```

Replace `TIMESTAMP` with the current UTC timestamp (ISO 8601).

## Step 12 â€" Print setup summary

Print a clean summary of all checks:

```
=== arc Workflow Setup ===

âœ… Project context â€" repo root confirmed, not a worktree
âœ… workflow-config.json â€" loaded (repo: [repo])
âœ… Repo â€" up to date with remote                             [OR âš ï¸ warning]
âœ… workflow-config.local.json â€" gh_account: [gh_account]
âœ… gh auth â€" [gh_account] active, repo access confirmed
âœ… git remote â€" origin set to https://[gh_account]@github.com/[repo].git
âœ… git hooks â€" hooks/pre-push active                          [OR ⚠️ warning]
âœ… gcloud â€" [account] (\$GCLOUD_CONFIG config)                    [OR âš ï¸ warning]
âœ… GitHub labels â€" all required labels present
âœ… .gitignore â€" local config, screenshots, and session files excluded
✅ Playwright + Chromium — available                         [OR ❌ run: cd frontend && npx playwright install chromium]
ℹ️  Chrome MCP — [connected / not detected] (optional — manual debugging only)
âœ… frontend/.env.test.local â€" present                        [OR âš ï¸ warning]
âœ… Node.js â€" [version] / npm [version]                       [OR âš ï¸ warning]
âœ… agent-state.json â€" present

Setup complete. You're ready to use:
  /arc-start        — session brief
  /arc-fix-issue    — fix an issue end-to-end
  /arc-verify-issue — QA a fix on the dev environment
```

If any required step failed (gh auth, repo access, workflow-config.json missing), highlight it clearly and stop the summary at that point — the workflow won't function until it's resolved.

---

## To reset or correct setup

Re-running `/arc-setup` will show current values and let you update them interactively — no manual file editing needed.

To start completely from scratch (e.g. switching machines or accounts), delete the local config first and then re-run:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
rm "$MAIN_ROOT/.claude/workflow-config.local.json"
```

Then run `/arc-setup` again.
