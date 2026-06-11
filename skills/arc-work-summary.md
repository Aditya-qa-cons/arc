---
name: po-work-summary
description: Generates a work summary of all PRs merged to develop since the last time this command was run (incremental). First run falls back to 14 days. Shows what was done, what it unblocks, and who needs to act next. Format is controlled by `team.work_summary_format` in workflow-config.json. Run at the end of a session or whenever you need to share a progress update with the team.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Generate Work Summary

Produce a formatted summary of recent work. No arguments needed — auto-detects what changed since the last summary.

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

## Step 2 — Determine the cutoff date

Resolve the main repo root (works from any worktree):

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
```

Read the last-run timestamp file:

```bash
cat "$MAIN_ROOT/.claude/last-work-summary.json" 2>/dev/null
```

The file contains `{"last_run": "YYYY-MM-DDTHH:MM:SSZ", "pr_cutoff": "YYYY-MM-DDTHH:MM:SSZ"}`.

**If the file exists:** use `pr_cutoff` as the cutoff. Cap it at 14 days back — if `pr_cutoff` is older than 14 days, use `now - 14 days` and note "showing last 14 days (last summary was N days ago)" in the header.

**If the file does not exist (first run):** use `now - 14 days` as the cutoff.

Keep the cutoff timestamp in context — it is the `mergedAt` lower bound for Step 3.

## Step 3 — Find merged PRs

```bash
gh pr list --repo [repo] \
  --state merged --base develop \
  --json number,title,body,mergedAt \
  --limit 50 \
  --jq '[.[] | select(.mergedAt > "[cutoff timestamp]")]'
```

Also check current open PRs for in-progress work:

```bash
gh pr list --repo [repo] \
  --state open --base develop \
  --json number,title,headRefName,isDraft
```

## Step 4 — Gather QA outcomes from run log

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
OPERATOR=$(node -e "console.log(JSON.parse(require('fs').readFileSync('$MAIN_ROOT/.claude/workflow-config.local.json','utf-8')).gh_account)" 2>/dev/null)
tail -20 "$MAIN_ROOT/.claude/run-log-${OPERATOR}.jsonl" 2>/dev/null
```

Filter entries to `arc-verify-issue`, `arc-run`, and `arc-qa-session` skill logs with `ts` after the cutoff date from Step 2. Collect:
- Issues that `outcome: "passed"` — confirmed working on dev
- Issues that `outcome: "failed"` — QA rejected, have `Merged - Redo` label
- Issues `outcome: "aborted"` — could not complete QA (e.g. deployment timeout)

For `arc-qa-session` entries, separately collect:
- `section` and `section_title` — which section was tested
- `pass`, `fail`, `blocked`, `gap` — TC result counts
- `issues_filed` — issue numbers raised during the session
- `ts` and operator name (use the `$OPERATOR` variable resolved above) — for attribution

## Step 4a — Identify action items

Scan PR bodies and titles for items that require external action:
- Deploy/infra keywords ("firebase deploy", "cloud functions", "index deploy") → action for `team.reviewers` (read from workflow-config.json)
- "pending QA re-test" / "re-test on dev" keywords → action for `team.qa_person` (read from workflow-config.json)
- "blocked on" → note as a dependency

Also check `docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md` — items saying "pending QA re-test" = qa_person actions; items saying deploy/infra keywords = reviewer actions.

## Step 5 — Write the summary

Read `team.work_summary_format` from workflow-config.json. Use the appropriate format:
- `whatsapp` — plain text, `*bold*` with asterisks, no markdown headers (template below)
- `slack` — same structure, use Slack `*bold*` and channel-friendly formatting
- `markdown` — standard markdown headers and bold

Template (substitute `[project_name]` = team.project_name, `[reviewer]` = team.reviewers[0], `[qa_person]` = team.qa_person, `[dev_url]` = dev_url):

```
*[project_name] Work Summary — [date range]*

*Merged to develop:*

*PR #NNN* — [1-line description]
- [bullet: what it fixes/adds and who it helps]
- [only include bullets if there are 2+ noteworthy items]

[repeat for each merged PR]

*In review (CI passing / pending merge):*

*PR #NNN* — [1-line description]
- [key items]

*QA results (verified on [dev_url]):*

✅ #NNN [title] — passed
✅ #NNN [title] — passed
❌ #NNN [title] — failed (re-fix needed)
⏸ #NNN [title] — not yet verified (deployment pending)
[omit this section entirely if no QA was run this period]

*Manual QA sessions:*

📋 Section [N] — [pass] PASS · [fail] FAIL · [blocked] BLOCKED · [gap] GAP
   [If fail > 0:] FAILs: issue #[NNN], #[NNN]
   Tested by: [OPERATOR] · [date] · PR #[NNN]

[Repeat for each arc-qa-session entry in the period]
[Omit "Manual QA sessions" subsection entirely if no arc-qa-session entries exist in the period]

**Manual QA sessions rendering rules:**
- One block per session entry (one per section tested)
- If `fail == 0 && blocked == 0 && gap == 0`: single ✅ line with PASS count only — omit sub-bullets
- If `fail > 0`: always show the FAILs sub-bullet with issue numbers from `issues_filed`
- If `blocked + gap >= 3`: show a BLOCKED/GAP sub-bullet
- If `pr` is null, omit the PR number suffix from the Tested by line
- Omit the "Manual QA sessions" subsection entirely if no `arc-qa-session` entries exist in the period

---

*Who needs to act — [reviewer] first:*

1. *[Action title]* — [what to do, include exact command if it's a deploy]
   Reason: [what is blocked until this is done]

2. [next reviewer action if any]

*[qa_person] — after [reviewer]'s steps above:*

1. *[Test case title]* — [what to test, which login, which URL]
2. [next qa_person item]

[omit a person's section entirely if they have no actions]
```

### Date range label in the header

- Normal incremental run: `[last summary date] – [today]` e.g. `13–15 May 2026`
- First run / capped at 14 days: `last 14 days (to [today])`
- If zero PRs were found: say so — "No new product PRs since [last summary date]" — and still save the timestamp (Step 6).

## Rules

- Keep each bullet to one line — this is for sharing, not a doc
- Only include PRs since the cutoff date — never repeat PRs already in a previous summary
- List reviewer actions before qa_person actions — QA testing is often blocked on deploy steps
- If a qa_person test requires a reviewer deploy, note "(requires [reviewer] step N above)" next to it
- If there are no external actions needed, end after the merged PRs section
- Do not include CI/automation/docs-only PRs unless they unblock something meaningful for the team
- PRs that only changed `.claude/`, `.github/workflows/`, or `schema/migrations/` don't need a bullet unless they solve a pain point the team would care about

## Step 6 — Output the summary

Print the formatted summary as plain text. The user will copy-paste it into their preferred channel.
Do not wrap it in a code block.

## Step 7 — Save the run timestamp

After outputting the summary, update the timestamp file so the next run knows where to start:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
# pr_cutoff = mergedAt of the most recent PR included, or now if no PRs found
echo '{"last_run":"[now ISO8601]","pr_cutoff":"[most recent PR mergedAt, or now]"}' \
  > "$MAIN_ROOT/.claude/last-work-summary.json"
```

Do this **after** printing the summary — never before, so a crash mid-summary doesn't advance the cutoff.
