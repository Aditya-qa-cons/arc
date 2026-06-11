---
name: po-fix-issue
description: Fix a GitHub issue end-to-end — fetches issue context, investigates root cause, creates a worktree branch, implements the fix, creates a PR, and prepares QA handoff. Takes an issue number as argument (e.g. /arc-fix-issue 567). Use this whenever starting work on any open issue from the po-start brief or when the user says "fix issue #NNN", "work on #NNN", or "let's tackle #NNN".
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Fix an Issue

End-to-end workflow: read the issue → investigate root cause → fix in an isolated branch → PR → QA handoff reminder.

## Step 1 — Read config and gh auth

### 1a — Read workflow config

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if local config is missing, stop and tell the user: "`.claude/workflow-config.local.json` not found — run `/arc-setup` first to configure your account."
- `repo` from project config (e.g. `your-org/your-repo`)
- `base_branch` (e.g. `develop`)
- `labels` object — for QA handoff labels
- `workflow.steps[id="fix"].options.pr_gate` — whether to pause for approval before creating PR (default: `true` if not set)
- `pr_title_format` and `branch_naming` patterns

If `.claude/workflow-config.json` is missing, stop: "`.claude/workflow-config.json` not found — run `/arc-setup` first."

### 1b — gh auth check

```bash
gh auth status  # Must show the account from local config active
```

If the wrong account is active, run `gh auth switch --user [gh_account]` before continuing.

## Step 2 — Resolve issue number

If a number was passed as argument, use it directly.

If no number given, check the current branch name — `fix/issue-NNN-*` implies NNN. Otherwise ask: "Which issue number should I work on?"

## Step 3 — Fetch issue details

```bash
gh issue view NNN --repo [repo from config] \
  --json number,title,body,labels,assignees,comments
```

Print a brief before doing anything else:

```
Issue #NNN — [title]
Labels: [labels]
Priority: [p0-critical / priority-high / bug / etc.]

[body — trimmed to the key facts: what's broken, where, any reproduction steps]
```

If the issue has comments, scan them for extra context: related PRs, workarounds someone tried, updated reproduction steps.

## Step 4 — Check for existing PRs

Before investigating or writing any code, check whether an open PR already addresses this issue:

```bash
gh pr list --repo [repo from config] --state open \
  --search "(#NNN)" \
  --json number,title,headRefName,reviewDecision,statusCheckRollup
```

Also check the issue comments — contributors often link a PR there.

**If an open PR exists:**
- Print its number, title, CI status, and review decision
- Ask the user: "PR #NNN already addresses this — do you want to review/fix that PR instead of starting fresh?"
- If yes: stop here and suggest `/fix-pr-issues NNN` if CI is failing, or note it needs a reviewer
- If no (user wants to redo): continue to Step 5

**If no open PR exists:** proceed directly to Step 5.

## Step 5 — Investigate with engineering:debug

Invoke the `engineering:debug` skill to find the root cause before writing a single line of code. Don't skip this — guessing at a fix without understanding the cause leads to regressions.

**Investigation order — follow this sequence to minimise token use. Stop as soon as the root cause is clear:**

1. **Git log first** — check recent changes to the suspected path:
   ```bash
   git log --oneline -10 -- frontend/src/[suspected/path]
   git log --oneline -10 -- functions/src/[suspected/path]
   ```
   A recent commit to the exact file is the root cause ~30% of the time.

2. **Grep for the symbol** — search for the specific component, function name, or error string using the `Grep` tool across `frontend/src/` or `functions/src/`. This pins the exact file and line range without reading anything.

3. **Read only the relevant lines** — use `offset` and `limit` on the `Read` tool to read the specific section. Never read an entire file to find one thing.

The investigation should answer:
- Where exactly in the codebase is the broken behaviour?
- Is this frontend, backend (Cloud Functions), Firestore rules, or a combination?
- Are there related issues or recent PRs that touched the same area and may have caused this?
- What is the minimal change that fixes the root cause without side effects?

## Step 6 — Create worktree and branch

Once the root cause is clear, create an isolated worktree using the `superpowers:using-git-worktrees` skill.

Branch naming:
- Bug fix: `fix/issue-NNN-short-description` (3-5 word kebab-case summary of what the fix does)
- e.g. `fix/issue-567-inspectors-tab-load-error`

The branch **must** be based on `develop`. Never branch from `main`.

## Step 7 — Implement the fix

Work in the worktree. Keep the change minimal — only what resolves the root cause identified in Step 5. Don't clean up unrelated code, don't refactor, don't add features.

After implementing:

```bash
# 1. Run targeted tests first — faster feedback, no noise from unrelated failures
#    Test files are co-located with source: same directory, .test.ts or .spec.ts suffix
#    e.g. frontend/src/components/InspectorHistory.tsx → InspectorHistory.test.ts
#    If no co-located test exists, look in the nearest __tests__/ directory
cd frontend && npm test -- --run [path/to/affected.test.ts]
npx tsc --noEmit

# 2. Run the full suite only once before creating the PR (not during iteration)
cd frontend && npm test -- --run

# If the fix touched Cloud Functions, also run backend tests:
cd ../functions && npm test
```

If tests fail, fix them before proceeding. If a pre-existing test was already broken (unrelated to this issue), note it explicitly rather than fixing it silently.

## Step 8 — Save task state

Resolve the main repo root first (works whether running from the main checkout or a worktree):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

Write `"$MAIN_ROOT/.claude/task-state.md"` so `po-start` can find it regardless of which directory the next session opens from:

```markdown
Branch: fix/issue-NNN-short-description
Issue: #NNN — [title]
Last commit: [hash] [message]
Next step: [e.g. "Create PR" / "Monitor CI" / "Waiting for review"]
Last saved: [YYYY-MM-DD HH:MM]
```

Update this file whenever the next step changes (e.g. after PR is created, update to "Monitor CI").

## Step 9 — Create PR using finishing-a-development-branch

### 9a — PR gate check

First check `po_run.auto_advance` from the workflow config. If it is `true`, skip the gate entirely — no confirmation needed in automated mode.

If `po_run.auto_advance` is `false` (the default) **and** `pr_gate` is `true` in the workflow config, pause before creating the PR:

```
⏸ po-fix-issue — ready to create PR
What I see: Fix implemented, tests pass, [N] files changed
  - [file 1] — [what changed]
  - [file 2] — [what changed]
Shall I create the PR now? (y/n)
```

Wait for confirmation. If the user says no, save task state with next step "Create PR" and stop.

### 9b — Create PR

Invoke the `superpowers:finishing-a-development-branch` skill to create the PR.

Project-specific requirements (from workflow-config.json):
- **Base branch**: value of `base_branch` from config — pass `--base [base_branch]` explicitly, never trust GitHub's default
- **Title format**: follow `pr_title_format` from config — e.g. `fix(ops): resolve inspectors tab load error (#567)`
- **Body must include**: `Closes #NNN` so GitHub auto-closes the issue on merge

### 9c — Handle review comments (po-run mode only)

If `po_run.auto_advance` is `true`:

1. Wait 90 seconds for Gemini/CodeRabbit to post their initial review
2. Check for inline review comments:
   ```bash
   gh api repos/[repo from config]/pulls/[PR_NUMBER]/comments \
     --paginate --jq '[.[] | select(.in_reply_to_id == null)] | length'
   ```
3. If the count is greater than 0, invoke `/fix-pr-issues NNN` to process all threads
4. Update task state: `Next step: Waiting for merge — PR #NNN`

If `po_run.auto_advance` is `false`, skip this step. Tell the user: "PR #NNN created. Gemini will post a review shortly — run `/fix-pr-issues NNN` to address any inline comments before the PR is merged."

## Step 10 — After the PR is merged

Once a reviewer (read `team.reviewers` from workflow-config.json) merges the PR:

1. Re-resolve the main repo root and delete the task state file:
   ```bash
   MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
   rm -f "$MAIN_ROOT/.claude/task-state.md"
   ```
2. Run `/arc-finish-issue` — reopens the issue for QA, adds "Merged - Check" label, writes verification comment
3. **If running standalone** (not from po-run): check whether the fix touched user-facing code before running `/arc-update-test-plan`:
   ```bash
   gh pr view [PR_NUMBER] --repo [repo] --json files \
     --jq '[.files[].path | select(startswith("frontend/src/") or startswith("functions/src/"))] | length'
   ```
   If the output is `0` (CI-only, config, docs, or migration fix), skip `/arc-update-test-plan`. Otherwise run it to sync the QA test plan.

   **If running from po-run** (`po_run.auto_advance: true`): skip — po-run handles the conditional check and calls `/arc-update-test-plan` at the end of the full batch
4. **If running standalone**: ask the user:
   > PR merged and QA comment posted. Do you want to run `/arc-verify-issue NNN` now (~10 min wait for deployment), or verify it later?
   - If **now**: invoke `/arc-verify-issue NNN`
   - If **later**: confirm that the QA person (`team.qa_person` from workflow-config.json) can pick it up via the "Merged - Check" label, and stop
   
   **If running from po-run**: skip — po-run handles verification in its own Step 7

## Step 11 — Append run log entry

Derive the operator name from `gh_account` in `workflow-config.local.json` and append one JSON line to `"$MAIN_ROOT/.claude/run-log-[gh_account].jsonl"` (create the file if it doesn't exist). This per-operator file is committed to the repo so runs from all teammates are visible:

```bash
OPERATOR=$(jq -r '.gh_account' "$MAIN_ROOT/.claude/workflow-config.local.json")
echo '{...}' >> "$MAIN_ROOT/.claude/run-log-${OPERATOR}.jsonl"
```

```json
{
  "ts": "<ISO 8601 timestamp>",
  "skill": "arc-fix-issue",
  "issue": <NNN>,
  "outcome": "<pr_created | aborted_pr_gate | aborted_investigation>",
  "escalations": ["<list of escalation trigger IDs that fired, e.g. pr_gate, ambiguous_root_cause>"],
  "fix_attempts": <number of implementation iterations before tests passed>,
  "review_rounds": <number of times fix-pr-issues was run on the PR>,
  "qa_result": "<passed | failed | deferred | null>",
  "notes": "<one sentence on anything non-obvious — regressions caught, root cause surprises, scope creep>"
}
```

Do this even if the run was aborted — a partial run is still data. If a field is unknown, use `null`.

---

## Escalation triggers

Pause and ask the user in these specific situations. Use this format:

```
⏸ po-fix-issue — input needed
What happened: [concrete description of the blocker]
What I see: [specific evidence — file names, error messages, test output]
Options:
  A. [option A]
  B. [option B]
  C. [other — describe your preference]
```

**Trigger 1 — Ambiguous root cause after investigation**
The `engineering:debug` investigation points to two or more unrelated locations and it's not clear which is the real cause. Don't guess — stop and surface both candidates with evidence.

**Trigger 2 — Fix requires touching unrelated areas**
Implementing the fix requires changing code in a module that seems unrelated to the issue (e.g. fixing a display bug requires changing the Firestore schema). This may indicate the issue scope is larger than filed, or there's a dependency the issue didn't surface.

**Trigger 3 — Tests fail and can't be fixed quickly**
After implementing the fix, `npm test -- --run` or `npx tsc --noEmit` fails and the failure isn't obviously caused by the fix (e.g. a different test file fails, or an existing type error is exposed). Surface the exact error output.

**Trigger 4 — PR gate (Step 9a)**
`pr_gate: true` in config. Always pause here — see Step 9a above.
