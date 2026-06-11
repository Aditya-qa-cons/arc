---
name: po-run
description: Full workflow loop — auto-selects open issues by priority, groups into fix batches, runs each batch end-to-end (investigate → PR → QA → test plan). No issue number needed.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# po-run — Full Workflow Loop

Automated end-to-end session: read the backlog → group issues into batches → fix each batch → PR → merge → QA → repeat.

No issue number needed. Run at the start of a work session and the agent handles the rest, pausing only at human checkpoints.

## Step 1 — Read config and gh auth

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if missing, stop: "Run `/arc-setup` first."
- `repo`, `base_branch`, `labels`, `deployment` values — same as po-fix-issue
- `po_run.auto_advance` — whether to automatically start the next batch (`false` = wait for "next")
- `po_run.max_batch_size` — maximum issues per batch
- `branch_naming.batch` — e.g. `fix/batch-{n}-{description}`

```bash
gh auth status  # Must show gh_account active; switch if not
```

### 1a — Check for in-progress work from a previous session

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat "$MAIN_ROOT/.claude/task-state.md" 2>/dev/null || echo "No in-progress task found."
```

If the file exists and has content, surface it:

```
⚠️  In-progress task found from a previous session:
[show the full task-state.md content]

Options:
  A. Resume — pick up from the saved next step
  B. Start fresh — discard the previous state and plan a new batch
```

If the user chooses B, delete the task-state file before continuing:

```bash
rm -f "$MAIN_ROOT/.claude/task-state.md"
rm -f "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

After reading config, print a capability summary so the user knows what mode they're running in before any work starts:

```
po-run — mode: [auto_advance: ON | OFF] · max batch size: [max_batch_size] issues
  ✅ Auto-advance [ON: batches start automatically after a 10s pause | OFF: waits for "next" between batches]
  ✅ PR gate: always pauses before creating a PR (regardless of auto-advance)
  ✅ Merge gate: always waits for you to signal "merged" before running QA
  ✅ QA: runs po-verify-issue for each issue after merge

To change auto-advance: edit po_run.auto_advance in .claude/workflow-config.json
Type "done" at any batch boundary to stop the loop and generate the work summary.
```

## Step 2 — Session brief

Run the po-start logic to sync the repo and get a current picture:

```bash
git fetch origin
git pull
```

Then fetch open issues:

```bash
gh issue list --repo [repo] --state open \
  --json number,title,labels,assignees,createdAt \
  --limit 100
```

**Filter to actionable issues** — exclude any with these labels (use names from config):
- `[labels.merged_check]` (waiting for QA — already merged, don't re-fix)
- `[labels.merged_redo]` (will be picked up but treat as high-priority — add to batch like any bug)
- `[labels.qa_passed]` (done)
- `po-run:active` (claimed by another active po-run session — skip to avoid conflicts)

Do NOT filter by assignee — any issue can be picked up regardless of who it is assigned to,
since AI capability is not constrained by individual ownership. Assignees are used for human
attention routing only.

Print a brief count: `Found N actionable issues (X p0-critical, Y priority-high, Z bug/other)`

## Step 3 — Group issues into batches

Analyse the issue titles and bodies to cluster them by the area of the codebase they affect. Use this priority order within and across batches:

1. `p0-critical` — always in Batch 1, alone if unrelated to other issues
2. `priority-high` — next, grouped with related issues
3. `bug` / unlabelled — lowest priority

**Grouping logic:**

Group issues together when they share at least one of:
- Same frontend module or route area (e.g. `/inspector/…`, `/supplier/…`, `/ops/…`, `/admin/…`)
- Same backend service or Cloud Function area
- Related root cause (e.g. "all Firestore query failures", "all auth/permission issues")
- Same component or type file likely to be touched

Keep each batch within `po_run.max_batch_size`. If a p0-critical issue is unrelated to all other issues, put it alone in its own batch.

**Present the proposed batches:**

```
⏸ po-run — proposed fix batches

Batch 1 — [area, e.g. "inspector module"] (N issues):
  #NNN [title]
  #NNN [title]

Batch 2 — [area] (N issues):
  #NNN [title]

Batch 3 — [area] (N issues):
  #NNN [title]

Proceed with this grouping? Or adjust before we start?
  A. Start with Batch 1 as proposed
  B. Adjust — [describe change]
```

Wait for confirmation before continuing. If the user adjusts the grouping, update the batch plan and confirm once more before proceeding.

### 3a — Claim the batch (before any investigation)

Before touching any code or running any investigation, claim every issue in the confirmed
batch by adding the `po-run:active` label and posting a timestamp comment. This prevents
another concurrent po-run session from picking up the same issues.

For each issue number `NNN` in the confirmed batch:

1. **Re-verify the issue is still unclaimed** — check current labels before applying the lock
   (another session may have claimed it during the confirmation pause):

   ```bash
   gh issue view NNN --repo [repo] --json labels --jq '.labels[].name'
   ```

   If `po-run:active` appears in the output, skip this issue and warn:
   ```
   ⚠️  #NNN already claimed by another po-run session — removing from this batch.
   ```
   Continue claiming the remaining issues.

2. **Apply the lock**:

   ```bash
   gh issue edit NNN --repo [repo] --add-label "arc-run:active"
   gh issue comment NNN --repo [repo] \
     --body "🤖 \`po-run\` claimed this issue — $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
   ```

After all issues are processed, confirm claimed vs skipped:

```
✅ Claimed Batch N: #NNN, #NNN — proceeding to investigation.
   (⚠️  #NNN already claimed by another session — skipped.)
```

If all issues in the batch were already claimed, surface this to the user as a conflict and re-group before proceeding.

Only after all claims are posted, proceed to Step 4.

## Step 4 — Fix the current batch

Work through one batch at a time. For Batch N:

### 4a — Create worktree and branch

Use the `superpowers:using-git-worktrees` skill to create an isolated worktree.

Branch name: follow the `branch_naming.batch` pattern from config (`fix/batch-{n}-{description}`), substituting `{n}` with the batch number and `{description}` with a 3-5 word kebab-case summary of the batch's theme (e.g. `fix/batch-1-inspector-history-fixes`).

The branch **must** be based on `base_branch` from config.

### 4b — Investigate all issues before touching any code

Before writing a single line of code, run `engineering:debug` to understand ALL issues in the batch. Doing this upfront (rather than issue by issue) reveals shared root causes and avoids fixing the same file twice in conflicting ways.

**Investigation order — follow this sequence to minimise token use:**

For each issue, investigate in this order and stop as soon as the root cause is clear:

1. **Git log first** — check recent changes to the suspected file area:
   ```bash
   git log --oneline -10 -- frontend/src/[suspected/path]
   git log --oneline -10 -- functions/src/[suspected/path]
   ```
   This alone identifies the root cause ~30% of the time (recent commit broke it).

2. **Grep for the symbol** — use the `Grep` tool (not bash `grep`) to search for the specific component name, function name, or error string across `frontend/src/` or `functions/src/`. This pins the exact file and line range without reading anything.

3. **Read only the relevant lines** — use `offset` and `limit` on the `Read` tool to read only the specific section. Never read an entire file to find one thing.

For each issue:
- Where in the codebase is the broken behaviour?
- Frontend, backend (Cloud Functions), Firestore rules, or combination?
- Does the fix overlap with any other issue in this batch?

After investigating all issues, summarise:

```
Batch N investigation complete:
  #NNN — root cause: [1 sentence], fix: [1 sentence]
  #NNN — root cause: [1 sentence], fix: [1 sentence]
  Shared files: [list any files touched by multiple fixes]
  Fix order: [recommended order to avoid conflicts]
```

If two issues conflict (e.g. both require changing the same function in incompatible ways), pause:

```
⏸ po-run — conflicting fixes in batch
What I see: #NNN and #NNN both require changing [file:function] in incompatible ways.
Options:
  A. Fix #NNN first, then adapt #NNN's fix to the new code
  B. Split — move #NNN to the next batch
  C. Other — describe your preference
```

### 4c — Implement fixes in sequence

Fix each issue in the order determined in 4b. Keep changes minimal — only what resolves the root cause. Don't clean up unrelated code.

After all fixes are implemented, run tests once:

```bash
cd frontend && npm test -- --run
npx tsc --noEmit

# If any fix touched Cloud Functions:
cd ../functions && npm test
```

If tests fail due to one of the fixes (not a pre-existing failure), fix it before continuing. If a pre-existing test was already broken, note it explicitly.

### 4d — Save batch state

Resolve the main repo root first (works whether running from the main checkout or a worktree):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

Write `"$MAIN_ROOT/.claude/task-state.md"`:

```markdown
Batch: fix/batch-{n}-{description}
Issues: #NNN, #NNN, #NNN — [titles]
Last commit: [hash] [message]
Next step: [e.g. "Create PR" / "Monitor CI" / "Waiting for merge"]
Last saved: [YYYY-MM-DD HH:MM]
```

### 4e — PR gate

Always pause before creating the PR:

```
⏸ po-run — ready to create PR for Batch N
What I see: [N] issues fixed, tests pass, [X] files changed
  - [file 1] — [what changed and why]
  - [file 2] — [what changed and why]

PR will close: #NNN, #NNN, #NNN
Shall I create the PR now? (y/n)
```

Wait for confirmation.

### 4f — Create PR

Use `superpowers:finishing-a-development-branch` to create the PR.

Project-specific requirements (from workflow-config.json):
- **Base branch**: `base_branch` from config — pass `--base [base_branch]` explicitly
- **Title**: `fix([area]): [summary of all fixes] (#NNN, #NNN, #NNN)` — e.g. `fix(inspector): resolve history tab, badge count, and dashboard stats (#567, #572, #580)`
- **Body must include**: one `Closes #NNN` line per issue so GitHub auto-closes them on merge

## Step 5 — Wait for merge

After the PR is created, print:

```
✅ PR #NNN created — fix/batch-{n}-{description}
   Closes: #NNN, #NNN, #NNN

Waiting for CodeRabbit and Gemini reviews to pass, then anyone on the team (`team.reviewers`
from workflow-config.json, or any team member with merge access) can merge. Let me know when it's merged ("merged" or "PR merged").
```

Stop and wait. Do not proceed to Step 6 until the user signals the merge.

**If CI goes red while waiting:** run `/fix-pr-issues [PR_NUMBER]` to process all inline review comments and address failing checks. Once CI is green again, return to waiting for the merge signal.

When the user signals merge (any message containing "merged", "it's merged", "PR merged", "merged it", etc. — **not** "done", which is reserved for stopping the loop in Step 9):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
rm -f "$MAIN_ROOT/.claude/task-state.md"
```

Then proceed to Step 6.

## Step 6 — Post-merge: finish issues

> **Context:** The `issue-handoff.yml` GitHub Action fires within seconds of a PR merging and handles reopening + labeling automatically. By the time the user signals "merged" (typically 1–5 minutes after the merge), the Action has already run. This means po-finish-issue's reopen and label steps will be no-ops — its primary job in po-run context is upgrading the generic verification comment the Action posted into a detailed pw-act version.

For each issue in the batch, invoke the `po-finish-issue` skill:

```
Invoke po-finish-issue skill for issue #NNN
```

The skill handles everything: reopening the issue (if somehow still closed), adding the `Merged - Check` label (idempotent), and writing the structured verification comment with pw-act action arrays. Do NOT inline this logic here — always delegate to the skill so that every improvement to po-finish-issue automatically applies in batch mode.

Do this for all issues in the batch before moving on to QA.

## Step 7 — Wait for deployment and QA

Invoke the `po-verify-issue` skill for each issue in the batch sequentially. Do NOT re-implement the QA logic inline — always delegate to the skill so that every improvement to `po-verify-issue` (login approach, escalation handling, run-log entries, etc.) automatically applies in batch mode too.

**Deployment check:** `po-verify-issue` Step 2 checks deployment internally. For the first issue in the batch, let it run the full deployment poll — it will write `.claude/deployment-confirmed.json` when live. For subsequent issues in the same batch, po-verify-issue reads that file and skips the poll automatically.

**For each issue in the batch:**

Before invoking po-verify-issue, update task state so the session can recover if archived during QA:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
cat > "$MAIN_ROOT/.claude/task-state.md" << 'EOF'
Skill: po-run (QA phase)
Batch: fix/batch-{n}-{description}
Issues in batch: #NNN, #NNN — [titles]
Current QA target: #NNN
Remaining issues to QA: #MMM, #OOO (or "none")
Next step: Invoke po-verify-issue for issue #NNN, then continue remaining issues
Last saved: [YYYY-MM-DD HH:MM]
EOF
```

Then invoke the skill:

```
Invoke po-verify-issue skill for issue #NNN
```

The skill handles everything: deployment check, login, verification steps, QA report comment, label updates, run-log entry. Do not duplicate any of these steps here.

If the skill escalates (login failed, page timeout, ambiguous result), handle it per the escalation protocol in `po-verify-issue` and surface the blocker to the user before moving to the next issue.

If any issue **fails** QA:
- The skill will have applied `Merged - Redo` label and left the issue open
- Continue invoking the skill for the remaining issues in the batch — don't abort the whole batch on one failure

After all issues are verified, clean up the deployment cache and print a QA summary:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
rm -f "$MAIN_ROOT/.claude/deployment-confirmed.json"
```

```
Batch N QA complete:
  ✅ #NNN [title] — QA Passed, issue closed
  ✅ #NNN [title] — QA Passed, issue closed
  ❌ #NNN [title] — QA Failed (Merged - Redo), will appear in next batch plan
```

## Step 8 — Update test plan (conditional)

First check whether the batch touched any files in `frontend/src/` or `functions/src/`. Replace `NNN` with the PR number created in Step 4f:

```bash
gh pr view NNN --repo [repo] --json files \
  --jq '[.files[].path | select(startswith("frontend/src/") or startswith("functions/src/"))] | length'
```

- If the output is `0` (fix was CI-only, config-only, docs-only, or migration-only): **skip this step entirely** — no user-facing behaviour changed, so the test plan has nothing to add.
- If the output is `> 0`: run the po-update-test-plan logic to sync the QA test plan doc with the merged PR.

## Step 9 — Cycle to next batch

After completing Steps 6–8 for the current batch, check `po_run.auto_advance`:

**If `auto_advance: false` (default):**

```
✅ Batch N complete.

Next up — Batch N+1 — [area] (N issues):
  #NNN [title]
  #NNN [title]

Say "next" to start Batch N+1, or "done" to stop and run the work summary.
```

Wait for the user's signal.

**If `auto_advance: true`:**

Print the same preview, then automatically start Batch N+1 after a 10-second pause (allows the user to interrupt with "stop" if needed).

**If no more batches remain:**

Proceed directly to Step 10.

## Step 10 — Work summary

Once all batches are complete (or the user says "done"), run the po-work-summary logic to generate the team-communication-ready summary of everything merged in this session.

## Step 11 — Append run log entry

Derive the operator name from `gh_account` in `workflow-config.local.json` and append one JSON line to the per-operator log file (so logs persist across worktree cleanups and are visible to all teammates):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
OPERATOR=$(jq -r '.gh_account' "$MAIN_ROOT/.claude/workflow-config.local.json")
```

Append to `"$MAIN_ROOT/.claude/run-log-${OPERATOR}.jsonl"` (create the file if it doesn't exist):

```json
{
  "ts": "<ISO 8601 timestamp>",
  "skill": "arc-run",
  "batches_planned": <total batches in plan>,
  "batches_completed": <batches fully merged and QA'd>,
  "issues_fixed": [<list of issue numbers merged this session>],
  "issues_redo": [<issue numbers that failed QA and got Merged-Redo>],
  "escalations": ["<list of escalation trigger IDs that fired across all batches>"],
  "auto_advance": <true | false>,
  "notes": "<one sentence on anything non-obvious — regroupings, conflicts between fixes, p0 hotfixes, unexpected blockers>"
}
```

---

## Escalation triggers

Use this format when pausing for input:

```
⏸ po-run — input needed
What happened: [concrete description]
What I see: [specific evidence]
Options:
  A. [option A]
  B. [option B]
  C. [other — describe]
```

**Trigger 1 — Conflicting fixes within a batch**
Two issues require incompatible changes to the same code. See Step 4b.

**Trigger 2 — Investigation reveals batch should be re-grouped**
After investigating, it becomes clear that two issues share a root cause with an issue in a different batch, or that an issue is far more complex than the title suggested (e.g. requires a schema change). Surface this before writing any code.

**When an issue is removed from a batch (any trigger):**

If an issue is dropped from the current batch for any reason (re-grouped, out of scope,
requires human design decision), release its lock immediately:

```bash
gh issue edit NNN --repo [repo] --remove-label "arc-run:active"
gh issue comment NNN --repo [repo] \
  --body "🤖 \`po-run\` released this issue — removed from current batch. Reason: [one sentence]"
```

If the issue needs a human owner, assign it at the same time:
- Needs manual QA verification on dev → assign to `ogvetsoncall`
- Needs feature design/implementation → assign to `jmat35` or `TamerineSky`
- Needs infra/QA/dev work → assign to the operator running this session (`gh_account`)

**Trigger 3 — Tests fail after all fixes applied**
`npm test -- --run` or `npx tsc --noEmit` fails and the failure is from one of the fixes. Identify which fix caused it and surface the error output before attempting a repair.

**Trigger 4 — QA fails on a p0-critical issue**
A p0-critical issue fails QA after merge. Don't wait for the next cycle — surface it immediately and ask whether to start a hotfix batch now.

**Trigger 5 — PR gate (Step 4e)**
Always pause here, regardless of `auto_advance` setting.
