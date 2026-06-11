# `/arc-init` — Project Setup Wizard

Run once when adopting po-skills on a new project. Generates `workflow-config.json` from scratch, then delegates to `/arc-setup` for machine verification.

**Do not run inside a git worktree.** Run from the repo root.

---

## Step 1 — Verify context

Check whether you are at the repo root (not inside a git worktree):

```bash
git rev-parse --git-dir
```

If the output contains `worktrees/` — you are inside a worktree. Tell the user: "Exit the worktree first — open the main repo folder in Claude Code, then re-run `/arc-init`." Stop.

If the command fails with "not a git repository" — tell the user: "Open the repo folder in Claude Code first, then re-run `/arc-init`." Stop.

Check if `workflow-config.json` already exists:

```bash
test -f .claude/workflow-config.json && echo "EXISTS" || echo "NOT_FOUND"
```

If it EXISTS, ask the user:

> "`workflow-config.json` already exists. Choose:
>   1. Overwrite — run the full wizard and replace it
>   2. Skip to machine setup — run `/arc-setup` (keeps existing config)
>
>   (1 / 2)"

Wait for their answer. If they choose 2, invoke `/arc-setup` and stop. If they choose 1, continue to Step 2.

If the config is NOT_FOUND, continue to Step 2 without asking.

---

## Step 2 — Auto-detect project values

Run these four detection commands silently (do not print intermediate output):

```bash
# 1. Repo
git remote get-url origin
```
Extract the `owner/repo` slug from the remote URL, handling both formats:
- HTTPS: `https://github.com/owner/repo.git` → strip `https://github.com/` prefix and optional `.git` suffix
- SSH: `git@github.com:owner/repo.git` → strip `git@github.com:` prefix and optional `.git` suffix

If this fails (no remote), ask the user: "No git remote found. What is your GitHub repo? (e.g. `myorg/myrepo`)" Use their answer as `repo`.

```bash
# 2. Base branch
REPO="<repo-from-above>"
gh api "repos/$REPO" --jq '.default_branch'
```
If `gh` is not installed or not authenticated, stop: "Install GitHub CLI (`gh`) first — https://cli.github.com — then re-run `/arc-init`."

```bash
# 3. Project name
[ -f package.json ] && python3 -c 'import sys,json; print(json.load(sys.stdin).get("name",""))' < package.json || echo ""
```
If `package.json` is missing or `name` is empty, fall back to the part after `/` in the repo slug (e.g. `myorg/my-app` → `my-app`).

```bash
# 4. Deployment provider (check in priority order — first match wins)
for f in apphosting.yaml firebase.json vercel.json .vercel/ railway.toml render.yaml; do [ -e "$f" ] && echo "$f" && break; done
```
Detection logic:
- `apphosting.yaml` or `firebase.json` present → `firebase-app-hosting`
- `vercel.json` or `.vercel/` present → `vercel`
- `railway.toml` or `render.yaml` present → `http-health`
- None found → `unknown`

**Show a single confirmation block to the user:**

```
Detected values — confirm or correct:
  1. repo:         <value>
  2. base_branch:  <value>
  3. project_name: <value>
  4. provider:     <value>  (detected from <filename> / not detected)

All correct? (y / or type the number to fix: 1=repo, 2=branch, 3=name, 4=provider)
```

If the user types a number, ask for the corrected value, update it, then re-show the block. Repeat until the user confirms with `y`.

Store all four values for use in later steps.

---

## Step 3 — Provider-specific questions

Ask only the questions required for the confirmed provider. One question at a time — wait for each answer before asking the next.

### If provider = `firebase-app-hosting`

Ask in order:

1. "Firebase web API key? (Firebase console → Project settings → General → Web API key)"
   → store as `firebase_api_key`

2. "App Hosting backend name? (Firebase console → App Hosting — shown as the backend resource name)"
   → store as `deployment.app_hosting_backend`

3. "gcloud config name for this project? (run `gcloud config configurations list` to check — e.g. `procureops`)"
   → store as `gcloud.config_name`

4. "GCP project ID for your dev/staging environment? (e.g. `myapp-dev`)"
   → store as `gcloud.project_dev`

5. "GCP project ID for your production environment? (e.g. `myapp-prod`)"
   → store as `gcloud.project_prod`

Derived values (no questions needed):
- `qa.auth_method` = `"firebase"`
- `deployment.version_field` = `"sha"` (default — overridable manually later)

---

### If provider = `vercel`

Ask:

1. "Vercel project slug? (the name of your project in Vercel — shown in the dashboard URL: `vercel.com/<team>/<project-slug>`)"
   → store as `deployment.vercel_project`

Derived values:
- `qa.auth_method` = `"cookie"`
- `deployment.version_field` = `"sha"` (default)

---

### If provider = `http-health`

Ask in order:

1. "Health endpoint URL for your dev environment? (e.g. `https://staging.example.com/api/health`)"
   → store as `deployment.health_url`

2. "Which JSON field in that response contains the deployed commit SHA? (press Enter for default: `sha`)"
   → store as `deployment.version_field` (if empty, use `"sha"`)

Derived values:
- `qa.auth_method` = `"cookie"`

---

### If provider = `unknown` (not detected in Step 2)

Ask:

"Deployment provider not detected. Choose:
  1. firebase-app-hosting — Firebase App Hosting (apphosting.yaml)
  2. vercel — Vercel
  3. http-health — Any platform with a health endpoint (Railway, Render, etc.)

(1 / 2 / 3)"

Wait for answer, then set `provider` to the chosen value (`firebase-app-hosting`, `vercel`, or `http-health`) and proceed with that provider's questions above. This `provider` value will be used in Step 6 to select the correct config template.

---

## Step 4 — Environment + team

Four questions in order. One at a time — wait for each answer.

1. "Dev/staging environment URL? (e.g. `https://dev.example.com`)"
   → store as `dev_url`

2. "Who can merge pull requests? Comma-separated names (e.g. `Ben, Alice, Carlos`)"
   → split on commas, trim whitespace, store as `team.reviewers` array

3. "Who is the primary QA person? (single name)"
   → store as `team.qa_person`

4. "Work summary format?
     1. whatsapp
     2. slack
     3. markdown
   (1 / 2 / 3)"
   → map choice to string: `"whatsapp"` / `"slack"` / `"markdown"`, store as `team.work_summary_format`

Also store `project_name` (confirmed in Step 2) as `team.project_name`.

---

## Step 5 — QA users

Ask:

> "Do you have dedicated test accounts for QA verification? `/arc-verify-issue` logs in as these accounts to check fixes on the dev environment.
>
> (y / skip — fill in later)"

**If skip:**
- Store `qa_users` as an empty array `[]`
- Print a reminder to the user:
  ```
  ⚠️  qa_users left empty — /arc-verify-issue will ask for credentials manually
      until you fill this in. Add entries to .claude/workflow-config.json when
      your test accounts are ready.
  ```
- Continue to Step 6.

**If y:**
Collect one user at a time. For each user, ask in order:

1. "Role name? (e.g. `super_admin`, `brand_user`, `inspector`)"
   → `role`
2. "Email address for this test account?"
   → `email`
3. "Environment variable name for this user's password? (e.g. `TEST_USER_ADMIN_PASSWORD`)"
   → `env_var`
4. "Entry path after login? (e.g. `/dashboard`)"
   → `entry_path`

Build a user object: `{ "role": "...", "email": "...", "env_var": "...", "entry_path": "..." }`

After each user ask:
> "Add another test user? (y / done)"

Continue collecting until the user answers `done`. Store the full array as `qa_users`.

---

## Step 6 — Generate and write workflow-config.json

Assemble all collected values into the full config object. Use the template for the confirmed provider.

### Template: `firebase-app-hosting`

```json
{
  "schema_version": "1.0",
  "repo": "[REPO]",
  "base_branch": "[BASE_BRANCH]",
  "dev_url": "[DEV_URL]",
  "firebase_api_key": "[FIREBASE_API_KEY]",
  "deployment": {
    "provider": "firebase-app-hosting",
    "_provider_note": "Valid values: firebase-app-hosting (default), vercel, http-health. Each provider requires different sub-fields below.",
    "app_hosting_backend": "[APP_HOSTING_BACKEND]",
    "typical_duration_minutes": 10,
    "poll_interval_minutes": 2,
    "timeout_minutes": 20,
    "_vercel_note": "vercel_project is required when provider=vercel (slug shown in vercel.com dashboard URL)",
    "vercel_project": "",
    "_http_health_note": "health_url and version_field required when provider=http-health. version_field is the JSON key containing the commit SHA.",
    "health_url": "",
    "version_field": "sha"
  },
  "qa": {
    "auth_method": "firebase",
    "_auth_method_note": "firebase=pw-page.mjs IDB injection (Firebase projects). cookie=form-based login (pw-act.mjs). none=no auth required.",
    "login_path": "/login"
  },
  "gcloud": {
    "config_name": "[GCLOUD_CONFIG_NAME]",
    "project_dev": "[GCLOUD_PROJECT_DEV]",
    "project_prod": "[GCLOUD_PROJECT_PROD]"
  },
  "labels": {
    "merged_check": "Merged - Check",
    "merged_redo": "Merged - Redo",
    "qa_passed": "QA Passed",
    "p0_critical": "p0-critical",
    "priority_high": "priority-high"
  },
  "branch_naming": {
    "bug_fix": "fix/issue-{number}-{description}",
    "batch":   "fix/batch-{n}-{description}",
    "feature": "feat/{description}"
  },
  "pr_title_format": "{type}({area}): {description} (#{issue})",
  "po_run": {
    "auto_advance": false,
    "max_batch_size": 5
  },
  "team": {
    "project_name": "[PROJECT_NAME]",
    "reviewers": [REVIEWERS],
    "qa_person": "[QA_PERSON]",
    "work_summary_format": "[WORK_SUMMARY_FORMAT]"
  },
  "qa_users": [],
  "workflow": {
    "steps": [
      { "id": "fix",              "skill": "arc-fix-issue",        "enabled": true, "options": { "pr_gate": true } },
      { "id": "fix_pr_issues",    "skill": "fix-pr-issues",       "enabled": true },
      { "id": "finish",           "skill": "arc-finish-issue",     "enabled": true },
      { "id": "verify",           "skill": "arc-verify-issue",     "enabled": true, "requires": ["finish"] },
      { "id": "update_test_plan", "skill": "arc-update-test-plan", "enabled": true }
    ]
  }
}
```

### Template: `vercel`

Same as above **except**:
- Remove `firebase_api_key` field entirely
- Remove `gcloud` block entirely
- Set `deployment.provider` = `"vercel"`
- Set `deployment.vercel_project` = `"[VERCEL_PROJECT]"` (from Step 3)
- Remove `deployment.app_hosting_backend`
- Set `qa.auth_method` = `"cookie"`

### Template: `http-health`

Same as above **except**:
- Remove `firebase_api_key` field entirely
- Remove `gcloud` block entirely
- Set `deployment.provider` = `"http-health"`
- Set `deployment.health_url` = `"[HEALTH_URL]"` (from Step 3)
- Set `deployment.version_field` = `"[VERSION_FIELD]"` (from Step 3, default `"sha"`)
- Remove `deployment.app_hosting_backend`
- Set `qa.auth_method` = `"cookie"`

### Substitution rules

Replace all `[PLACEHOLDER]` tokens with the actual collected values:
- `[REPO]` → confirmed repo from Step 2
- `[BASE_BRANCH]` → confirmed base_branch from Step 2
- `[PROJECT_NAME]` → confirmed project_name from Step 2
- `[DEV_URL]` → dev_url from Step 4
- `[FIREBASE_API_KEY]` → firebase_api_key from Step 3 (firebase only)
- `[APP_HOSTING_BACKEND]` → deployment.app_hosting_backend from Step 3 (firebase only)
- `[GCLOUD_CONFIG_NAME]` → gcloud.config_name from Step 3 (firebase only)
- `[GCLOUD_PROJECT_DEV]` → gcloud.project_dev from Step 3 (firebase only)
- `[GCLOUD_PROJECT_PROD]` → gcloud.project_prod from Step 3 (firebase only)
- `[VERCEL_PROJECT]` → deployment.vercel_project from Step 3 (vercel only)
- `[HEALTH_URL]` → deployment.health_url from Step 3 (http-health only)
- `[VERSION_FIELD]` → deployment.version_field from Step 3 (http-health only)
- `[REVIEWERS]` → replace with the collected `team.reviewers` array as a JSON array of strings (e.g., `["Ben", "Alice"]`) — include all N entries
- `[QA_PERSON]` → team.qa_person from Step 4
- `[WORK_SUMMARY_FORMAT]` → team.work_summary_format from Step 4
- `qa_users` → replace the empty array `[]` with the collected array from Step 5 (or leave `[]` if skipped)

### Write the file

Use the Write tool to write the assembled JSON to `.claude/workflow-config.json`.

Print the full file content to the user.

Ask:

> "`workflow-config.json` written. Does this look right? (y / edit)"

If `edit`: tell the user to make changes directly in `.claude/workflow-config.json`, validate the JSON is still valid (`python3 -m json.tool .claude/workflow-config.json`), then ask "Ready to continue? (y)" before proceeding.
If `y`: continue to Step 7.

---

## Step 7 — Handoff to /arc-setup

Print:

```
✅ workflow-config.json — written
Running /arc-setup to verify machine setup...
```

Invoke `/arc-setup`. It will handle: local config creation, gh auth, git remote verification, git hooks, gcloud check, GitHub label creation, .gitignore, Playwright, .env.test.local reminder, Node.js version, and agent-state.json.

When `/arc-setup` completes, `/arc-init` is done.
