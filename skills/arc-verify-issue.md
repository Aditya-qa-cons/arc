---
name: po-verify-issue
description: Verify a merged fix on the dev environment — waits for deployment to complete, reads the QA steps from the GitHub issue comment, logs into dev as the correct test user, follows each verification step, takes screenshots as evidence, and posts a pass/fail QA report back to the issue. Takes an issue number as argument (e.g. /arc-verify-issue 549). Use this after a PR has merged to develop. Also triggers when the user says "verify issue #NNN", "check if #NNN is fixed on dev", or "run QA on #NNN".
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Verify a Fix on Dev

Automated QA run: wait for deployment → read verification steps → log in to [dev_url] as the correct test user → follow each step → screenshot evidence → post pass/fail report to the issue.

Run this any time after a PR merges to `develop` — Step 2 will wait until the deployment is live before proceeding.

## Step 1 — Read config and gh auth

### 1a — Read workflow config

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if local config is missing, stop and tell the user: "`.claude/workflow-config.local.json` not found — run `/arc-setup` first to configure your account."
- `repo` from project config (e.g. `your-org/your-repo`)
- `dev_url` — the dev environment base URL (from workflow-config.json)
- `labels.merged_check`, `labels.merged_redo`, `labels.qa_passed` — for label management in Step 10
- `deployment.provider` — which hosting provider to use for deploy-wait logic (`firebase-app-hosting` | `vercel` | `http-health`). Default to `firebase-app-hosting` if field is absent.
- `deployment.app_hosting_backend` — only needed when `provider` is `firebase-app-hosting`
- `deployment.vercel_project` — only needed when `provider` is `vercel`
- `deployment.health_url` and `deployment.version_field` — only needed when `provider` is `http-health`
- `deployment.typical_duration_minutes` — how long a deploy normally takes (used in wait messages)
- `deployment.poll_interval_minutes` — how long to wait between status checks
- `deployment.timeout_minutes` — how long before escalating if still not deployed
- `qa.auth_method` — which login strategy Step 7 uses (`firebase` | `cookie` | `none`). Default to `firebase` if field is absent.
- `qa.login_path` — only needed when `auth_method` is `cookie`. The URL path of the login form (e.g. `/login`).

If `.claude/workflow-config.json` is missing, stop: "`.claude/workflow-config.json` not found — run `/arc-setup` first."

### 1b — gh auth check

```bash
gh auth status  # Must show the account from local config active
```

If the wrong account is active, run `gh auth switch --user [gh_account]` before continuing.

## Step 2 — Wait for deployment to complete

Firebase App Hosting deploys `develop` automatically after each merge. The deployment typically takes `deployment.typical_duration_minutes` minutes (from config). Running QA before it finishes produces false failures, so always confirm the fix is live first.

### 2a — Get the expected commit SHA

Find the PR that closed this issue using `closingIssuesReferences` — this is more reliable than text search:

```bash
gh pr list --repo [repo] --state merged --limit 20 \
  --json number,title,mergeCommit,closingIssuesReferences \
  --jq '.[] | select(.closingIssuesReferences[].number == NNN) | {number, title, sha: .mergeCommit.oid}'
```

If that returns nothing (some PRs don't use closing keywords), fall back to text search and cross-check manually:

```bash
gh pr list --repo [repo] --state merged --search "(#NNN)" \
  --json number,title,body,mergeCommit,closingIssuesReferences \
  --jq '.[] | {number, title, sha: .mergeCommit.oid, closes: [.closingIssuesReferences[].number]}'
```

Pick the PR whose body explicitly mentions fixing issue NNN. If the PR can't be found either way, check the issue's "Merged — Ready for QA Check" comment for a PR number, then:

```bash
gh pr view [PR_NUMBER] --repo [repo] --json mergeCommit --jq '.mergeCommit.oid'
```

Keep the full SHA in context — this is the commit that must be deployed before QA can proceed.

### 2b — Poll deployment status

Check `deployment.provider` (from Step 1a). If the field is absent, treat it as `firebase-app-hosting`.

- **`firebase-app-hosting`** → follow the [Provider: firebase-app-hosting](#provider-firebase-app-hosting) section below
- **`vercel`** → follow the [Provider: vercel](#provider-vercel) section below
- **`http-health`** → follow the [Provider: http-health](#provider-http-health) section below

---

#### Provider: firebase-app-hosting

**Batch shortcut — check for confirmed deployment first:**

When running inside `po-run`, the first issue in a batch confirms deployment and writes a cache file. Subsequent issues can skip the poll entirely:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT/.claude/deployment-confirmed.json" 2>/dev/null
```

If the file exists and:
- `confirmed_sha` equals `[expected-sha]` OR `[expected-sha]` is an ancestor of `confirmed_sha` (run `git merge-base --is-ancestor [expected-sha] [confirmed_sha]`)
- AND the file is less than 60 minutes old (check `confirmed_at`)

Then: print `✅ Deployment live — confirmed by earlier check this batch (SHA [short SHA])` and skip to Step 3. No polling needed.

If the file is absent, stale, or the SHA check fails, proceed with the full poll below.

**Primary method** — uses only `gh` and `git`, no gcloud/firebase required:

```bash
# Get the SHA of the most recent successful "Deploy Dev Environment" run on develop
gh run list --repo [repo] --branch develop --limit 5 \
  --json databaseId,name,status,conclusion,headSha,createdAt \
  --jq '.[] | select(.name == "Deploy Dev Environment" and .conclusion == "success") | {sha: .headSha, created: .createdAt}' \
  | head -1

# Then verify the fix SHA is included in the deployed SHA
git fetch origin develop
git merge-base --is-ancestor [expected-sha] [deployed-sha] && echo "included" || echo "not yet deployed"
```

If `included`, the fix is live. If `not yet deployed`, a new deploy may be in progress.

**Fallback** (only if the GitHub Actions workflow is not named "Deploy Dev Environment"):

```bash
CLOUDSDK_ACTIVE_CONFIG_NAME=[gcloud.config_name] \
  firebase apphosting:rollouts:list [deployment.app_hosting_backend] \
  --project [gcloud.project_dev] --json 2>/dev/null
```

Note: `firebase apphosting:rollouts:list` requires Firebase CLI v16+. If the command errors, it means an older CLI version is installed — stick with the primary method above.

**If deployment is complete:** print `✅ Deployment live — commit [short SHA] serving on [dev_url]`. Then write the deployment cache so subsequent issues in the same batch can skip the poll:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
echo "{\"confirmed_sha\":\"[deployed-sha]\",\"confirmed_at\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  > "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

Then proceed to Step 3.

**If deployment is still in progress or the commit isn't deployed yet:**

Before showing the wait options, write task state so the session can recover if it gets archived during the wait. Only write if a parent `po-run` batch has not already written the state (to avoid overwriting richer batch context):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
EXISTING_SKILL=$(grep "^Skill:" "$MAIN_ROOT/.claude/task-state.md" 2>/dev/null | awk '{print $2}')
if [ "$EXISTING_SKILL" != "arc-run" ]; then
  cat > "$MAIN_ROOT/.claude/task-state.md" << 'EOF'
Skill: po-verify-issue
Issue: #NNN
Expected SHA: [expected-sha]
Current deployed SHA: [deployed-sha or "none"]
Next step: Resume Step 2b — re-poll deployment status, then continue QA for issue #NNN
Last saved: [YYYY-MM-DD HH:MM]
EOF
fi
```

Then show the wait options:

```
⏸ po-verify-issue — deployment not ready
What I see: Latest live commit is [SHA], expected [expected SHA]
Estimated wait: ~[N] minutes remaining

Options:
  A. Wait and retry in 2 minutes (re-run this step)
  B. Skip the deployment check and proceed anyway (risk: testing stale build)
  C. Abort — I'll re-run /arc-verify-issue NNN once the deploy is done
```

Wait for user choice. For option A, wait `deployment.poll_interval_minutes` minutes and re-run Step 2b. Repeat until deployed or user aborts.

**Session recovery:** If this session was archived during a deployment wait and has just been unarchived, re-run Step 2b immediately — the saved task state provides enough context to resume without user input.

**Timeout:** If more than `deployment.timeout_minutes` minutes have passed since the merge and the deployment hasn't succeeded, escalate:

```
⏸ po-verify-issue — deployment taking longer than expected
What I see: Rollout started [N] minutes ago and hasn't reached SUCCEEDED state
This may indicate a build failure or infrastructure issue.
Check Firebase App Hosting console or ask a reviewer (`team.reviewers` from workflow-config.json) to investigate.
```

---

#### Provider: vercel

**Pre-requisite:** `vercel` CLI must be installed and authenticated (`vercel whoami` should print your account). `deployment.vercel_project` must be set in workflow-config.json.

**Batch shortcut — check for confirmed deployment first:**

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT/.claude/deployment-confirmed.json" 2>/dev/null
```

If the file exists and `confirmed_sha` equals `[expected-sha]` OR is an ancestor, and less than 60 minutes old — print `✅ Deployment live — confirmed by earlier check this batch (SHA [short SHA])` and skip to Step 3.

**Poll the latest deployment:**

```bash
vercel list [deployment.vercel_project] --json 2>/dev/null | \
  node -e "
    process.stdin.setEncoding('utf8');
    let d = '';
    process.stdin.on('data', c => d += c).on('end', () => {
      const deps = JSON.parse(d);
      deps.slice(0, 5).forEach(dep => console.log(JSON.stringify({
        sha: dep.meta?.githubCommitSha || dep.meta?.gitlabCommitSha || dep.meta?.bitbucketCommitSha || '',
        url: dep.url,
        state: dep.state,
        created: dep.created
      })));
    });
  "
```

Each line is one deployment. Find the entry where `state` is `"READY"` and `sha` starts with the first 7 characters of `[expected-sha]`.

**If the Vercel CLI is unavailable**, use the Vercel REST API (requires `VERCEL_TOKEN` env var):

```bash
curl -s "https://api.vercel.com/v6/deployments?projectId=[deployment.vercel_project]&limit=10&state=READY" \
  -H "Authorization: Bearer $VERCEL_TOKEN" | \
  node -e "
    process.stdin.setEncoding('utf8');
    let d = '';
    process.stdin.on('data', c => d += c).on('end', () => {
      const data = JSON.parse(d);
      (data.deployments || []).slice(0, 5).forEach(dep => console.log(JSON.stringify({
        sha: dep.meta?.githubCommitSha || dep.meta?.gitlabCommitSha || dep.meta?.bitbucketCommitSha || '',
        url: dep.url,
        state: dep.state
      })));
    });
  "
```

**Check if the fix is included in the deployed commit:**

```bash
git fetch origin develop
git merge-base --is-ancestor [expected-sha] [sha-from-vercel] && echo "included" || echo "not yet deployed"
```

If `included`, print `✅ Deployment live — commit [short SHA] serving on [dev_url]`. Write the deployment cache:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
echo "{\"confirmed_sha\":\"[deployed-sha]\",\"confirmed_at\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  > "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

Then proceed to Step 3.

**If not yet deployed** — write task state and show wait options (same as firebase-app-hosting):

```
⏸ po-verify-issue — deployment not ready
What I see: Latest Vercel deployment SHA is [SHA], expected [expected SHA]
Estimated wait: ~[N] minutes remaining

Options:
  A. Wait and retry in 2 minutes (re-run this step)
  B. Skip the deployment check and proceed anyway (risk: testing stale build)
  C. Abort — I'll re-run /arc-verify-issue NNN once the deploy is done
```

**Timeout:** If more than `deployment.timeout_minutes` have passed, escalate:

```
⏸ po-verify-issue — deployment taking longer than expected
What I see: No READY Vercel deployment found for commit [expected-sha] after [N] minutes.
Check the Vercel dashboard or ask a reviewer (`team.reviewers` from workflow-config.json) to investigate.
```

---

#### Provider: http-health

**Pre-requisite:** `deployment.health_url` and `deployment.version_field` must be set in workflow-config.json. The URL must return JSON containing the deployed commit SHA (full or short) in `deployment.version_field`.

Example config:
```json
"health_url": "https://dev.example.com/api/health",
"version_field": "sha"
```

**Batch shortcut — check for confirmed deployment first:**

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT/.claude/deployment-confirmed.json" 2>/dev/null
```

If the file exists and `confirmed_sha` equals `[expected-sha]` OR is an ancestor, and less than 60 minutes old — print `✅ Deployment live — confirmed by earlier check this batch (SHA [short SHA])` and skip to Step 3.

**Fetch the health endpoint:**

```bash
curl -sf "[deployment.health_url]" 2>/dev/null | \
  node -e "
    process.stdin.setEncoding('utf8');
    let d = '';
    process.stdin.on('data', c => d += c).on('end', () => {
      try {
        const obj = JSON.parse(d);
        console.log(obj['[deployment.version_field]'] || '');
      } catch (e) {
        console.log('');
      }
    });
  "
```

This prints the deployed SHA (may be a short SHA like `abc1234`). If the output is empty, the endpoint is unreachable or doesn't contain the expected field — treat as "not yet deployed."

**Check if the fix is deployed:**

The health endpoint often returns a short SHA (7 chars). Use prefix matching:

```bash
DEPLOYED_SHA="<output from above>"
EXPECTED_SHA="[expected-sha]"
EXPECTED_SHORT="${EXPECTED_SHA:0:7}"

if [[ "$DEPLOYED_SHA" == "$EXPECTED_SHORT"* ]] || \
   git merge-base --is-ancestor "[expected-sha]" "$DEPLOYED_SHA" 2>/dev/null; then
  echo "included"
else
  echo "not yet deployed"
fi
```

Note: `git merge-base --is-ancestor` only works with full SHAs. The prefix match handles short SHAs.

If `included`, print `✅ Deployment live — commit [short SHA] serving on [dev_url]`. Write the deployment cache:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
echo "{\"confirmed_sha\":\"[deployed-sha]\",\"confirmed_at\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  > "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

Then proceed to Step 3.

**If not yet deployed** — write task state and show wait options:

```
⏸ po-verify-issue — deployment not ready
What I see: Health endpoint returning [DEPLOYED_SHA], expected [expected SHA]
Estimated wait: ~[N] minutes remaining

Options:
  A. Wait and retry in 2 minutes (re-run this step)
  B. Skip the deployment check and proceed anyway (risk: testing stale build)
  C. Abort — I'll re-run /arc-verify-issue NNN once the deploy is done
```

**Timeout:** If more than `deployment.timeout_minutes` have passed, escalate:

```
⏸ po-verify-issue — deployment taking longer than expected
What I see: Health endpoint at [deployment.health_url] is returning [DEPLOYED_SHA], expected [expected-sha].
Check your CI/CD pipeline or ask a reviewer (`team.reviewers` from workflow-config.json) to investigate.
```

---

## Step 3 — Fetch the verification steps

Filter to only the QA comments at fetch time — issues often have dozens of comments (discussion, lock/unlock, previous QA reports) that are not needed here:

```bash
gh issue view NNN --repo [repo] \
  --json number,title,labels,comments \
  --jq '{
    number,
    title,
    labels: [.labels[].name],
    qa_comment: ([.comments[] | select(.body | startswith("## Merged — Ready for QA Check"))] | sort_by(.createdAt) | last)
  }'
```

The `qa_comment` field is the most recent `## Merged — Ready for QA Check` comment, or `null` if none exists. There may be more than one if the issue was fixed, failed QA, and re-fixed — `sort_by(.createdAt) | last` always picks the latest.

### 3a — Quality check: generic vs detailed comment

If `qa_comment` is `null` (no QA comment exists yet), offer to fix it inline:

```
⏸ po-verify-issue — no verification comment found on issue #NNN
What I see: Issue has no "## Merged — Ready for QA Check" comment yet.
This means /arc-finish-issue was not run after the PR merged.

Options:
  A. Run /arc-finish-issue NNN now, then continue QA automatically
  B. Abort — I'll run /arc-finish-issue manually and re-run /arc-verify-issue NNN afterward
```

If the user chooses A, invoke the `po-finish-issue` skill for NNN, wait for it to complete, then re-run the Step 3 fetch and continue.

If `qa_comment` is not null, check whether it is usable for automation. A comment is **generic** (old format) if it is missing ALL THREE of:
- A specific URL path line (`- URL: /...`)
- A bodyText assertion (`Confirm in page text:` or `Confirm in snapshot bodyText:`)
- A pw-act JSON block (a fenced ` ```json ` block inside the comment)

**If the comment is generic:** rewrite it automatically — invoke the `po-finish-issue` skill for this issue number, which will fetch the PR diff and issue body, then post a new detailed comment using the current template. No user input needed. Print: `ℹ️  #NNN — generic verification comment detected, rewrote with detailed template before proceeding.`

After the rewrite, re-run the Step 3 fetch to get the updated `qa_comment`.

**If the comment is already detailed:** proceed directly to extracting steps below.

**If rewriting fails** (e.g., po-finish-issue cannot find the merged PR or cannot determine the affected URLs), escalate:

```
⏸ po-verify-issue — could not auto-rewrite generic comment
What I see: Issue #NNN has a generic verification comment and the rewrite failed.
Reason: [specific error from po-finish-issue, e.g. "no merged PR found for this issue"]

Options:
  A. I'll update the verification comment manually, then re-run /arc-verify-issue NNN
  B. Continue anyway with the generic comment using fallback prose-translation rules (less reliable)
```

### 3b — Extract steps

Extract from `qa_comment.body`:
- **Login email** — e.g. `[email]`
- **Password env var** — e.g. `TEST_USER_INSPECTOR_PASSWORD`
- **Navigation path** — e.g. `/inspector/history`
- **Verification steps** — numbered list of actions + expected outcomes
- **Pre-requisites** — anything that must be done before testing (e.g. "deploy Firestore indexes")

## Step 4 — Check pre-requisites

If the verification comment lists pre-requisites, surface them with the exact command to run before continuing:

```
⚠️  Pre-requisites required before verification:

[For Firestore index deploys]:
  CLOUDSDK_ACTIVE_CONFIG_NAME=[gcloud.config_name] firebase deploy \
    --only firestore:indexes --project [gcloud.project_dev]

[For Cloud Functions deploys]:
  CLOUDSDK_ACTIVE_CONFIG_NAME=[gcloud.config_name] firebase deploy \
    --only functions --project [gcloud.project_dev]

Has this been done? (y/n)
```

Wait for confirmation before continuing — running verification against an incomplete deployment produces false failures that waste time.

If there are no pre-requisites listed, skip this step entirely.

## Step 5 — Preflight: confirm credentials file

The pw-*.mjs scripts read passwords directly from `frontend/.env.test.local` — you do not need to read or pass the password manually. Before calling any script, just confirm the file exists and contains the required variable:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
grep -q "^TEST_USER_XXX_PASSWORD=" "$MAIN_ROOT/frontend/.env.test.local" 2>/dev/null \
  && echo "found" || echo "missing"
```

Substitute the actual env var name from the verification comment (e.g. `TEST_USER_INSPECTOR_PASSWORD`).

If missing, stop: "Credentials not found — `frontend/.env.test.local` is missing or doesn't contain `TEST_USER_XXX_PASSWORD`. Ask a teammate for the file."

If found, proceed. The pw scripts will read the password themselves at runtime.

## Step 6 — Screenshot directory

`pw-page.mjs` and `pw-act.mjs` both call `mkdirSync` to create the screenshot directory automatically. No manual setup needed. Screenshots land at `.claude/qa-screenshots/issue-NNN/` in the main repo root. This directory is gitignored — files won't be committed. The QA report posted to GitHub describes what was seen in text; screenshots are for your own local reference.

## Step 7 — Navigate to the target page as the test user

Check `qa.auth_method` (from Step 1a). If the field is absent, treat it as `firebase`.

- **`firebase`** → follow the [Auth: firebase](#auth-firebase) section below
- **`cookie`** → follow the [Auth: cookie](#auth-cookie) section below
- **`none`** → follow the [Auth: none](#auth-none) section below

---

#### Auth: firebase

Use the Playwright helper script (`pw-page.mjs`) — it calls the Firebase REST API for fresh auth tokens, injects them into a headless browser via `addInitScript` (before Firebase SDK initialises), navigates to the target page, and returns page content + screenshot. This bypasses all Chrome MCP login issues since auth injection happens before any page JavaScript runs.

**Important — always resolve MAIN_ROOT first:**
```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

**Navigate to the initial page from the verification comment:**
```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
node "$MAIN_ROOT/.claude/pw-page.mjs" \
  <login-email-from-step-3> \
  <navigation-path-from-step-3> \
  login-success \
  NNN
```

Example for inspector/history:
```bash
node "$MAIN_ROOT/.claude/pw-page.mjs" [email from qa_users where role=inspector] /inspector/history login-success 551
```

**Parse the returned JSON:**
- `ok: true` + `redirectedToLogin: false` → auth worked, page loaded
- `redirectedToLogin: true` → auth failed; stop and escalate (see Trigger 1)
- `bodyText` → page content to check against expected outcomes
- `consoleErrors` → browser console errors (array of strings)
- `networkErrors` → 4xx/5xx responses (array of `{url, status}`)
- `screenshotPath` → absolute path of the saved screenshot

**On success:** note the page URL and key content visible in `bodyText` in the QA report. This counts as the "login succeeded" confirmation.

**Note — credentials:** `pw-page.mjs` reads passwords directly from `frontend/.env.test.local` using the same env var mapping as Step 5. If Step 5 confirmed the variable exists, this step will find it.

---

#### Auth: cookie

Use `pw-act.mjs` to fill the login form on `[dev_url][qa.login_path]`. The script navigates to the login page, fills the email and password fields, submits the form, and waits for a redirect away from the login URL.

**Credentials:** Read the email from `qa_users` (matching the role from Step 3). The password is read automatically from `frontend/.env.test.local` via the `env_var` field.

**Important — always resolve MAIN_ROOT first:**
```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

**Login action sequence:**

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
node "$MAIN_ROOT/.claude/pw-act.mjs" \
  <login-email-from-step-3> \
  "[qa.login_path]" \
  login-success \
  NNN \
  '[
    {"action": "fill",  "selector": "input[type=email], input[name=email], input[id*=email]",    "value": "<login-email-from-step-3>"},
    {"action": "fill",  "selector": "input[type=password], input[name=password]",                 "value": "__ENV_VAR__"},
    {"action": "click", "selector": "button[type=submit], input[type=submit], button:has-text(\"Sign in\"), button:has-text(\"Log in\"), button:has-text(\"Login\")"},
    {"action": "wait",  "ms": 2000}
  ]'
```

**Selector fallback:** If the generic selectors above don't match, inspect the page HTML and substitute specific `id` or `data-testid` selectors from the actual login form.

**Parse the returned JSON:**
- `ok: true` + `redirectedToLogin: false` → login succeeded, session cookie is active
- `redirectedToLogin: true` → login failed; stop and escalate (check credentials and form selectors)
- `bodyText` → page content after redirect — note the landing page in the QA report

**On success:** Note the post-login URL in the QA report. This counts as the "login succeeded" confirmation.

**Note — no persistent session:** Each `pw-act.mjs` or `pw-page.mjs` call runs a fresh headless browser with no shared session. For cookie-auth projects, every Step 8 pw-* call must include the full login action sequence. Always use `pw-act.mjs` for cookie-auth projects — `pw-page.mjs` uses Firebase IDB injection which will not work here.

---

#### Auth: none

No authentication is required for the pages under test. Skip Step 7 entirely and proceed to Step 8.

In Step 8, when a verification step says "navigate to [path]", use `pw-page.mjs` with the email of the first user in `qa_users` (any entry) — since the page is public, the auth injection will fail gracefully but the navigation will succeed and `bodyText` + `screenshotPath` will still be populated. Alternatively use `curl -s "[dev_url][path]"` for text-only checks.

If a step asks you to "log in as [role]" and then verify something — that step assumes auth is needed. For `none` apps, omit the login action and navigate directly to the target path.

---

## Step 8 — Follow each verification step

Work through the numbered steps from the verification comment one at a time.

### Decision: pw-page or pw-act?

Use this table to decide which script to use for each step:

| The step says… | Use | Why |
|---|---|---|
| "navigate to X and verify Y appears/is absent" | `pw-page.mjs` | Read-only — no interaction needed |
| "check the count / stat / value on a page" | `pw-page.mjs` | Read-only — just check bodyText |
| "click X then verify Y appears" | `pw-act.mjs` | Requires click action |
| "fill a form, submit, verify result" | `pw-act.mjs` | Requires fill + click + navigation |
| "open a dropdown / dialog / modal" | `pw-act.mjs` | Requires click to trigger |
| The comment already includes a JSON action array | `pw-act.mjs` | Use the provided array directly — do not rewrite it |

**If the comment already has a `pw-act actions:` block with a JSON array, use it verbatim.** Do not invent a new action sequence when one is provided.

### Running pw-page (read-only steps)

```bash
node "$MAIN_ROOT/.claude/pw-page.mjs" \
  <login-email> \
  <target-path> \
  step-N \
  <issue_num>
```

Check `ok`, `bodyText`, `consoleErrors`, `networkErrors` in the JSON output.

### Running pw-act (interactive steps)

Save the action array to a file and pass the path:

```bash
# PowerShell — write actions file
$actions = '[
  {"type":"waitForSelector","selector":"<initial-selector>","timeout":10000},
  {"type":"fill","selector":"<input-selector>","value":"<value>"},
  {"type":"select","selector":"select","value":"<option-label>"},
  {"type":"click","text":"<button-text>"},
  {"type":"waitForURL","pattern":"<url-substring>","timeout":12000},
  {"type":"snapshot","name":"step-N-result"}
]'
$actions | Set-Content "$MAIN_ROOT/.claude/qa-screenshots/issue-NNN/step-N-actions.json" -Encoding utf8

node "$MAIN_ROOT/.claude/pw-act.mjs" \
  <login-email> \
  <initial-path> \
  step-N \
  <issue_num> \
  "$MAIN_ROOT/.claude/qa-screenshots/issue-NNN/step-N-actions.json"
```

Check `ok`, `finalUrl`, `finalSnapshot.bodyText`, `snapshots[N].bodyText`, `consoleErrors` in the JSON output.

If the verification comment includes a **Teardown** section, run the teardown actions after the main verification passes:

```bash
# Save teardown actions, then pass as 6th argument
$teardown = '[{"type":"navigate","path":"/path/to/delete"},{"type":"click","text":"Delete"}]'
$teardown | Set-Content "$MAIN_ROOT/.claude/qa-screenshots/issue-NNN/step-N-teardown.json" -Encoding utf8

node "$MAIN_ROOT/.claude/pw-act.mjs" \
  <login-email> \
  <initial-path> \
  step-N \
  <issue_num> \
  "$MAIN_ROOT/.claude/qa-screenshots/issue-NNN/step-N-actions.json" \
  "$MAIN_ROOT/.claude/qa-screenshots/issue-NNN/step-N-teardown.json"
```

### Supported action types

| Type | Key fields | Notes |
|---|---|---|
| `navigate` | `path` | Navigate to another path on [dev_url] |
| `click` | `selector` or `text` | Click by CSS selector or visible button text |
| `fill` | `selector`, `value` | Fill textarea/input — fires real key events, works with React Hook Form |
| `select` | `selector`, `value` | Select dropdown option by visible label — native `<select>` only |
| `waitForText` | `text`, `timeout?` | Wait until text appears in body |
| `waitForURL` | `pattern` or `regex`, `timeout?` | Wait until URL matches — use `regex` for negative lookaheads |
| `waitForSelector` | `selector`, `state?`, `timeout?` | Wait for element to become visible |
| `snapshot` | `name` | Capture bodyText + screenshot at this point |
| `evaluate` | `script` | Run arbitrary JS in the page; result in step.result |
| `wait` | `ms` | Fixed pause — use only when waitForText/waitForSelector won't work |

### Translating ambiguous prose steps (fallback rules)

If the verification comment is vague or predates the new format, use these rules to interpret it:

- **"verify X loads / works / appears"** → pw-page, check bodyText does not contain "Failed to load", "error", or an empty spinner; check `ok: true`
- **"check that X is no longer showing"** → pw-page, confirm the error string is absent from bodyText
- **"open X and check Y"** → if X requires a click to open: pw-act with `click` + `waitForText`; if X is a direct URL: pw-page
- **"submit the form and verify"** → pw-act; use realistic fill values (description ≥ 20 chars, select a real option); check `finalSnapshot.bodyText` for the success indicator
- **"verify as two different roles"** → run pw-page or pw-act twice with different email args

### For each step (either script):

1. **Identify the URL** — every step needs an exact path; derive it from the step description if not stated explicitly
2. **Run the script** — screenshot is saved automatically to `.claude/qa-screenshots/issue-NNN/`
3. **Check `bodyText`** — confirm the specific string from the verification comment is present (or absent)
4. **Check `consoleErrors`** — flag only errors tied to this feature; pre-existing unrelated errors are noted but don't fail
5. **Check `networkErrors`** — 4xx/5xx on feature-relevant API calls is a fail
6. **Record the result** — ✅ pass or ❌ fail with a concrete description of what was seen (exact values, counts, text)

### What counts as a pass
- The page/component renders without an error state
- The expected content is visible (text, values, list items matching the issue's expected behaviour)
- No console errors from this feature's API calls or Firestore queries
- API calls for this feature returned 2xx

### What counts as a fail
- Error card, "Failed to load", or spinner content in `bodyText` when real data should exist
- Expected content absent from `bodyText` when it should be present
- `consoleErrors` contains errors tied to this feature (Firestore permission denied, 400/500 from the relevant API)
- `networkErrors` shows 4xx/5xx on the feature's API calls
- `ok: false` or `redirectedToLogin: true` on a step that should load normally

Pre-existing unrelated console errors (from other features) are OK — note them but don't fail on them.

## Step 9 — Post QA report to GitHub

Post a comment describing exactly what was seen — not just pass/fail verdicts:

```bash
gh issue comment NNN --repo [repo] --body '[report]'
```

Use this format:

```markdown
## QA Verification — [PASSED ✅ / FAILED ❌]

**Tested on**: [dev_url]  
**Login as**: [email]  
**Date**: [YYYY-MM-DD]

### Results

| Step | Result | What was seen |
|---|---|---|
| Navigate to History tab | ✅ Pass | Page loaded, showing 3 completed inspections with supplier names, dates, and pass/fail badges |
| Check stat cards | ✅ Pass | Total Completed: 3, Pass Rate: 67%, Average Score: 72 |
| Navigate to Notifications | ❌ Fail | "Failed to load notifications" error card — console shows Firestore index error |

### Console / Network
[List any errors observed, or "No errors detected on feature-relevant requests"]
```

Descriptions in the "What was seen" column must be specific — name actual values, counts, or text visible on screen. This is the evidence since screenshots are local only.

## Step 10 — Update labels and close

Use label names from the workflow config (`labels.merged_check`, `labels.qa_passed`, `labels.merged_redo`).

### If all steps passed

```bash
gh issue edit NNN --repo [repo from config] \
  --remove-label "[labels.merged_check]" --add-label "[labels.qa_passed]"
gh issue close NNN --repo [repo from config]
```

### If any step failed

```bash
gh issue edit NNN --repo [repo from config] \
  --remove-label "[labels.merged_check]" --add-label "[labels.merged_redo]"
```

Leave the issue open — it will surface in the next `/arc-start` brief under "Merged - Redo".

## Step 11 — Append run log entry and clear task state

Derive the operator name from `gh_account` in `workflow-config.local.json` and append one JSON line to `"$MAIN_ROOT/.claude/run-log-[gh_account].jsonl"` (create the file if it doesn't exist). This per-operator file is committed to the repo so runs from all teammates are visible:

Also clear the task-state file if this skill owns it (only when `Skill: po-verify-issue` — skip clearing if a parent `po-run` batch owns the state, as that parent will clean it up itself):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
OPERATOR=$(jq -r '.gh_account' "$MAIN_ROOT/.claude/workflow-config.local.json")
echo '{...}' >> "$MAIN_ROOT/.claude/run-log-${OPERATOR}.jsonl"
EXISTING_SKILL=$(grep "^Skill:" "$MAIN_ROOT/.claude/task-state.md" 2>/dev/null | awk '{print $2}')
[ "$EXISTING_SKILL" = "arc-verify-issue" ] && rm -f "$MAIN_ROOT/.claude/task-state.md"
```

If running standalone (not inside po-run), also clean up the deployment cache:

```bash
EXISTING_SKILL=$(grep "^Skill:" "$MAIN_ROOT/.claude/task-state.md" 2>/dev/null | awk '{print $2}')
[ "$EXISTING_SKILL" != "arc-run" ] && rm -f "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

When running inside po-run, po-run cleans up the cache file after all issues in the batch are verified (see po-run Step 7).

```json
{
  "ts": "<ISO 8601 timestamp>",
  "skill": "arc-verify-issue",
  "issue": <NNN>,
  "outcome": "<passed | failed | aborted>",
  "deployment_wait_mins": <how many minutes waited for deployment, 0 if already live>,
  "steps_total": <total verification steps in the QA comment>,
  "steps_passed": <steps that passed>,
  "steps_failed": <steps that failed>,
  "escalations": ["<list of escalation trigger IDs that fired, e.g. login_failed, page_load_timeout, ambiguous_result>"],
  "notes": "<one sentence on anything non-obvious — stale CDN, missing index, wrong test data, false failure>"
}
```

Do this even if the run was aborted mid-verification.

## Edge cases

**Page still shows old behaviour after Step 2 confirmed deployment:**
CDN caching can serve a stale build even after App Hosting reports `SUCCEEDED`. Re-run the same `pw-page.mjs` call — it launches a fresh headless browser with no cache, so it always hits the live build directly.

**Firestore index not deployed:**
Shows as a 500 error in `networkErrors` or "no matching index" in `consoleErrors`. This is a pre-requisite failure, not a code bug. Flag it clearly in the report: "Firestore index not deployed to dev — not a code regression. A reviewer needs to run: `CLOUDSDK_ACTIVE_CONFIG_NAME=[gcloud.config_name] firebase deploy --only firestore:indexes --project [gcloud.project_dev]`"

**Auth fails (`redirectedToLogin: true`):**
pw-page.mjs uses IDB injection via `addInitScript`. If the redirect happens, check: (1) `frontend/.env.test.local` has the correct password for the email, (2) the Firebase project `[gcloud.project_dev]` is accessible. Re-running the command will retry with fresh tokens.

**Verification comment has no UI steps (test-only or CI-only fix):**
If the comment says no UI change, skip Steps 6–8. Instead confirm the CI checks on the merged PR all passed and note that in the report.

---

## Escalation triggers

Pause and ask the user in these specific situations. Use this format:

```
⏸ po-verify-issue — input needed
What happened: [concrete description of the blocker]
What I see: [specific evidence — URL, error text, console output, screenshot filename]
Options:
  A. [option A]
  B. [option B]
  C. [other — describe your preference]
```

**Trigger 1 — Auth injection failed**
`pw-page.mjs` returned `redirectedToLogin: true` or `ok: false`. This means the Firebase IDB injection didn't work — the browser was redirected to the login page. Don't retry silently — check whether `frontend/.env.test.local` has the correct password for the email, and whether the Firebase REST API returned a successful token (errors appear in the `pw-page.mjs` stderr output).

**Trigger 2 — Page doesn't load after 15 seconds**
Navigated to the expected page but the content area never populated (spinner still spinning, or blank area where data should be). This could be a Firestore index missing, a permission error, or a broken API call — each needs a different fix.

**Trigger 3 — Ambiguous result on a step**
A step partially passes: some data loads but not the specific content the verification step describes, or the UI shows something that could be either the old or new behaviour. Don't guess — surface what was actually seen and ask whether it counts as a pass.

**Trigger 4 — Console shows a feature-related error but tests were supposed to fix it**
The browser console shows a Firestore permission error or 4xx/5xx on the exact API call that the fix was supposed to address. This may mean the fix didn't deploy, or there's a secondary issue. Surface the error and let the user decide whether to re-check deployment status or mark as fail.
