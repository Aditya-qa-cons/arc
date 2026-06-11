---
name: po-update-test-plan
description: Syncs QA_E2E_TEST_PLAN_QC_WORKFLOW.md by auto-detecting all code PRs merged to develop since the last update. Run after any batch of PRs merges, or whenever the GitHub Action reminder comment appears on a merged PR.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Update QA Test Plan

Auto-detect and apply all changes to `docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md` since its last update.
No arguments needed — the command gathers its own context.

## Step 1 — Read config and gh auth

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if missing, stop: "Run `/arc-setup` first."
- `repo` from project config (e.g. `your-org/your-repo`)

```bash
gh auth status  # Must show [gh_account] active
```

If wrong account is active, run `gh auth switch --user [gh_account]` before continuing.

## Step 2 — Find last update date

Read `docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md`. Find the "Last updated" line at the very bottom — it contains a date in the form `YYYY-MM-DD`. That is the cutoff.

## Step 3 — Fetch merged PRs since cutoff

```bash
gh pr list --repo [repo] \
  --state merged --base develop \
  --json number,title,body,mergedAt \
  --limit 50
```

Filter the results to PRs where `mergedAt` is after the cutoff date.

## Step 4 — Filter to code-touching PRs

For each PR from Step 3, check its changed files:

```bash
gh pr view <NUMBER> --repo [repo] --json files --jq '.files[].path'
```

**Keep** the PR if any file matches `frontend/src/**` or `functions/src/**`.

**Skip** the PR if all changed files are in: `docs/`, `.github/`, `.claude/`, `schema/migrations/`, or are `*.md` / `*.yml` / config files only.

## Step 5 — Read PR context

For each kept PR, you already have `title` and `body` from Step 3. That is the source of truth for what changed. Read them carefully before writing any updates.

## Step 6 — Update the test plan

Apply updates following the document's existing structure and conventions:

### New features / new UI
- Add a **NOT TESTED** test case in the most relevant section
- Use the existing table format: `| N.X | Description | Steps | Expected Result | NOT TESTED | Implemented PR #NNN (issue #NNN). Pending QA re-test on dev. |`

### Bug fixes
- Add a **"Resolved (date, PR #NNN):"** bullet in the section's bug list
- Add the issue to that section's **"Open issues"** list as "pending QA re-test on dev"
- If the fix unblocks a currently BLOCKED test case, update its Notes column

### Backend-only / infra / CI / index changes
- No new test case needed
- If it unblocks a test, update the test's Notes column with "Unblocked by PR #NNN"
- If an index deploy is required, note: "Requires a manual index deploy by a team member with GCP access"

### Section summaries
- Recount Pass / Fail / Blocked / Not Tested / Gap for the section
- Update the summary table at the bottom of the section

### Scorecard
- Recount totals in the "## Summary Scorecard" table

### Last updated line
Replace the last line with:
```
*Last updated: YYYY-MM-DD — [brief description of what changed]. Previous: [previous last-updated text]*
```
Use today's date — run `date +%Y-%m-%d` to get it.

## Step 7 — Commit

Before committing, confirm you are on `develop` (or the main checkout's default branch). This skill must never push a test plan update to a fix branch or worktree branch:

```bash
git branch --show-current
```

If the current branch is anything other than `develop`, switch first:

```bash
git checkout develop
git pull
```

Then commit and push. Use `--rebase` to handle the case where another PR merged between our pull and push (common during active sessions):

```bash
git add docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md
git commit -m "docs(qa): sync test plan — PRs $(list PR numbers) [auto]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git pull --rebase origin develop
git push origin develop
```

If `git pull --rebase` produces a conflict on `QA_E2E_TEST_PLAN_QC_WORKFLOW.md` (another teammate updated the test plan at the same time), resolve by keeping both sets of changes, then `git rebase --continue` before pushing.

## Rules

- Do not change PASS/FAIL results for tests Ganesh has already verified — only add notes
- Do not remove existing test cases — update or extend them
- If a PR's changes don't map cleanly to any existing section, add a note to the most related section's "Open issues" list
- If zero relevant PRs were found since the cutoff, say so and make no changes
