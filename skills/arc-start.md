---
name: po-start
description: Session startup — syncs repo, shows open PR CI status, lists open issues by priority, and prints a "pick up here" brief. Run at the start of every working session to recover context in under 30 seconds.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Session Start Brief

Recover full context at the start of a session. No arguments needed.

## Step 0 — Read config and gh auth

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if missing, stop: "Run `/arc-setup` first."
- `repo` from workflow-config.json (e.g. `your-org/your-repo`)

```bash
gh auth status  # Must show [gh_account] active
```

If wrong account is active, run `gh auth switch --user [gh_account]` before continuing.

---

## Step 0.5 — Task continuation check

Check for in-progress work from a previous session:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT/.claude/task-state.md" 2>/dev/null || echo "No in-progress task found."
```

If the file exists and has content:
- Show it to the user in this format:
  ```
  ⚠️  In-progress task found:
  Skill: [skill name, if present]
  Branch: [branch, or "N/A — QA/verification phase"]
  Last commit: [commit, if present]
  Next step: [next step]
  Last saved: [timestamp]
  ```
- Ask: "Do you want to continue this task, or start fresh?"
- If **continue**:
  - If the task-state includes a `Branch:` field (development/fix phase): switch to the branch (`git checkout [branch]`), skip the rest of po-start, and proceed with the next step from the saved state.
  - If the task-state is for a QA/verification phase (Skill: `po-verify-issue` or `po-run (QA phase)`): there is no branch to check out. Skip the git checkout, skip the rest of po-start, and invoke the appropriate skill directly using the issue number and next step from the saved state.
- If **start fresh**: delete the task state file (`rm "$MAIN_ROOT/.claude/task-state.md"`) and proceed with the full po-start flow below.

If the file does not exist: proceed directly to Step 1 below.

---

## Step 0.7 — Stale lock cleanup

Check for any `po-run:active` issues that have been abandoned (claim is old, no open PR).

```bash
gh issue list --repo [repo] --state open --label "arc-run:active" \
  --json number,title,comments,updatedAt \
  --limit 50
```

Before processing any issue, fetch all open PRs once (to avoid per-issue API calls):

```bash
gh pr list --repo [repo] --state open --json number,body --limit 100
```

Store the PR list in memory and use it for reference checks in step 3 below.

For each issue returned:

1. **Find the claim comment** — look through the issue's comments for the most recent one
   whose body matches `🤖 \`po-run\` claimed this issue —`. The ISO 8601 timestamp
   (format: `YYYY-MM-DDTHH:MM:SSZ`) is at the end of that line, after the ` — ` separator.
   If no such comment is found, use the issue's `updatedAt` value as the proxy timestamp
   (covers the edge case where the session crashed after labelling but before posting the comment).

2. **Check age** — compare the claim timestamp (or `updatedAt` proxy) to now (current UTC time):
   - If the age is **< 7200 seconds (2 hours)**: skip this issue entirely — an active
     session is mid-investigation. Do not touch it.
   - If the age is **≥ 7200 seconds (2 hours)**: proceed to step 3.

3. **Check for an open PR** — scan the pre-fetched PR list for any PR whose body contains
   `closes #NNN`, `fixes #NNN`, or `resolves #NNN` (case-insensitive):

   - If **any open PR** is found: skip it — work is genuinely in progress.
   - If **no open PR** is found: the lock is stale — release it.

4. **Release the stale lock**:

   ```bash
   gh issue edit NNN --repo [repo] --remove-label "arc-run:active"
   gh issue comment NNN --repo [repo] \
     --body "🤖 Stale \`po-run:active\` lock released by po-start — no open PR found after 2+ hours."
   ```

After processing all locked issues, print a summary line only if any were found:

```
🔒 Lock cleanup: N stale locks released (#NNN, #NNN), M skipped (active).
```

Then continue to Step 1.

---

## Step 1 — Sync repo

```bash
git fetch origin
git status
git log --oneline origin/develop..HEAD  # any local commits not yet on develop?
```

Pull if on develop and behind:
```bash
git pull
```

Note the current branch — if not on `develop`, the user is mid-task on a feature/fix branch.

## Step 2 — Open PRs

```bash
gh pr list --repo [repo] \
  --state open --base develop \
  --json number,title,headRefName,isDraft,statusCheckRollup,reviewDecision,updatedAt \
  --limit 20
```

For each open PR, extract:
- CI status: look at `statusCheckRollup` — find the "PR Check (develop)" workflow entries. Summarise as ✅ passing / ❌ failing / ⏳ pending
- Review decision: `APPROVED` / `CHANGES_REQUESTED` / `REVIEW_REQUIRED`
- Draft: skip drafts unless noting they exist

## Step 3 — Open issues

```bash
gh issue list --repo [repo] \
  --state open \
  --json number,title,labels,assignees,createdAt \
  --limit 40
```

Group by label — in this exact order of urgency:

- `Merged - Redo` — **QA failed on a previously merged fix** — highest priority; the fix landed but verification caught a regression or incomplete behaviour. Surface these first, separately from regular bugs so the context is clear.
- `p0-critical` / `priority-high` / `bug` — fix candidates
- `Merged - Check` — waiting for QA verification; deployment may still be in progress
- `enhancement` / `feature` — backlog
- No label or `needs-triage` — unreviewed

## Step 4 — Recent develop activity

```bash
git log origin/develop --oneline --since="7 days ago"
```

Note which issues/PRs landed recently that might affect open work.

## Step 4.5 — Read QA standing from run log

Read all operator log files and extract the most recent `arc-qa-session` entry per section:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT"/.claude/run-log-*.jsonl 2>/dev/null | grep '"skill":"arc-qa-session"'
```

From the results: parse each line as JSON. Skip any line that is not valid JSON or is missing `section` or `ts` fields. Group by `section`, keep the entry with the highest `ts` per section. Keep this data in context for Step 5.

Also read the test plan to know how many sections exist (to determine "not yet started" sections). Extract `qa.test_plan_path` from `workflow-config.json` (defaulting to `docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md` if absent), and read section headers from the test plan:

```bash
grep "^## [0-9]" "$MAIN_ROOT/[qa.test_plan_path]" 2>/dev/null | head -20
```

Keep the highest section number seen in context. If the file is unreadable, skip this command — the "not yet started" line will be omitted from the brief.

If no `arc-qa-session` entries exist in any log file, skip Step 4.5 entirely — the QA standing block will be omitted from the brief.

QA standing:
  Section [N] — [N] FAILs → #[NNN] · [N] BLOCKEDs [[blocker_patterns comma-joined]]
  Section [N] — ✅ fully clean ([N]/[tcs_total] PASS)
  Section [N+]+ — not yet started
  [omit this entire section if no arc-qa-session entries exist in any log file]

**QA standing rendering rules:**
- One line per section tested (most recent log entry per section) — cap at 5 sections; if more tested sections exist, append `…and N more sections` on a final line
- If `fail > 0`: show `N FAILs → #NNN, #NNN` — comma-join all issue numbers from `issues_filed` (e.g. `3 FAILs → #759, #760, #762`); omit the arrow entirely if `issues_filed` is empty
- If `blocked > 0`: show `N BLOCKEDs [blocker_patterns comma-joined]` — patterns give the unblock path at a glance
- If `gap > 0`: append `· N GAPs` (no detail — GAPs are product backlog items)
- If `fail == 0 && blocked == 0 && gap == 0`: show `✅ fully clean (N/tcs_total PASS)`
- "Section N+ — not yet started": only if the test plan has confirmed sections beyond the highest tested section number; omit if the test plan path is not set or unreadable
- Omit the entire QA standing block if no `arc-qa-session` entries exist in any log file

## Step 5 — Print the brief

Use this format (plain text, no markdown headers):

```
=== Session Brief — [date] ===

Branch: [current branch or "develop (clean)"]

Open PRs:
  #NNN [title] — ✅ CI / REVIEW_REQUIRED
  #NNN [title] — ❌ CI failing (Frontend unit tests)
  [none if no open PRs]

Needs re-fix (Merged - Redo):
  #NNN [title] — QA failed: [one-line reason from QA report]
  [none if clear — omit section entirely if no Redo issues]

Issues needing work:
  #NNN [title] — [bug / high priority]
  #NNN [title]
  [none if clear]

Waiting on QA (Merged - Check):
  #NNN [title]
  [none if clear]

Recent merges (last 7 days):
  PR #NNN — [title] (merged Xd ago)

Suggested next action:
  [One sentence — e.g. "Re-fix #NNN (QA failed)" or "Fix CI on PR #554" or "Start on issue #561" or "Nothing urgent — all PRs green"]
```

Keep the brief tight — the goal is 30-second context recovery, not a full status report.
