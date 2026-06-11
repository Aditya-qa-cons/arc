---
name: arc-qa-session
description: Manual QA testing session — guides a tester through test cases from the E2E test plan one by one. Generates a capture card per TC, reads the browser console and takes a screenshot after each step, updates the test plan immediately with results, files GitHub issues on FAILs, and raises a PR at the end of the session. Triggers when the user says "I want to test Section N", "Let's run test case N.M", "Continue Section N — we left off at N.M", "I'm done for today", or any similar manual QA intent.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File → Open Folder → select the repo folder, then retry.

# Manual QA Session

Guided manual testing against the dev environment. You perform the UI steps; Claude handles capture cards, console capture, screenshots, test plan updates, issue filing, and the end-of-session PR.

Full process documentation: `docs/4-operations/MANUAL_QA_GUIDE.md`

---

## Step 1 — Read config and gh auth

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `gh_account` from local config — if missing, stop: "`.claude/workflow-config.local.json` not found — run `/arc-setup` first."
- `repo` from project config (e.g. `myorg/myrepo`)
- `dev_url` — dev environment base URL
- `qa.test_plan_path` — path to the test plan file; default to `docs/QA_E2E_TEST_PLAN_QC_WORKFLOW.md` if absent

```bash
gh auth status  # Must show [gh_account] active
```

If wrong account is active: `gh auth switch --user [gh_account]`

Fetch tester display name:
```bash
gh api user --jq '.name'
```

Keep `display_name` and `gh_account` in context — used for tester attribution: `[display_name] ([gh_account])`.

---

## Step 2 — Parse the request

Determine from the user's message:

| User said | What to do |
|-----------|-----------|
| "test Section N" | Start at the first TC in section N that is NOT already PASS |
| "run test case N.M" | Start at TC N.M specifically |
| "continue Section N — left off at N.M" | Start at TC N.M |
| "I'm done for today" / "that's all for now" | Jump to [Step 8 — Session End](#step-8--session-end) |

If the section or TC number is ambiguous, ask: "Which section or test case would you like to start with?"

---

## Step 3 — Check for an in-progress session

```bash
cat .claude/task-state.md 2>/dev/null || echo "none"
```

If the file exists and `Skill: arc-qa-session` is present:

```
⚠️  In-progress QA session found:
Section: [N]
Last TC: [N.M]
Branch: [branch]
Results so far: [summary]
Next step: [next step]
Last saved: [timestamp]

Resume this session? (y/n)
```

- **Yes** → checkout `[branch]`, skip Steps 4–5, jump to the TC indicated in next step
- **No** → discard any local changes from the aborted session (`git checkout -- [qa.test_plan_path]`), delete `task-state.md`, proceed with Steps 4–5 as a fresh session

If no task-state exists, proceed directly.

---

## Step 4 — Sync repo and create session branch

```bash
git checkout develop
git pull origin develop
```

Check whether a session branch already exists for today (locally or on remote):
```bash
git fetch origin
git branch -a --list "*docs/qa-section-[N]-*-$(date +%Y-%m-%d)"
```

This also surfaces branches created by other testers today — used by the multi-tester conflict check in the edge cases section.

If your branch (`docs/qa-section-[N]-[gh_account]-$(date +%Y-%m-%d)`) exists locally or on remote, check it out:
```bash
git checkout docs/qa-section-[N]-[gh_account]-$(date +%Y-%m-%d)
```

If it does not exist, create it:
```bash
git checkout -b docs/qa-section-[N]-[gh_account]-$(date +%Y-%m-%d)
```

Keep the branch name in context — needed for the end-of-session commit.

---

## Step 5 — Read the test plan and check prerequisites

Read the test plan file from `qa.test_plan_path`. Locate the requested section.

**Prerequisite check:**
Read the Prerequisites block for the section. For each prerequisite:
- If it references a completed section (e.g. "Inspection from Section 4 is completed"), check the Section Summary table for that section's PASS count
- If it references dev data (e.g. "at least 1 supplier exists"), note it as assumed present unless there is evidence otherwise
- If a prerequisite is clearly unmet (e.g. "Section 4 is NOT complete"), surface it:

```
⚠️  Prerequisite not met: [description]
This will likely block TCs [list]. 

Options:
  A. Proceed anyway — mark blocked TCs as BLOCKED when we reach them
  B. Switch to a different section that is ready
  C. Abort
```

Wait for choice before continuing.

If prerequisites look OK, print a one-line confirmation and continue.

---

## Step 6 — Pre-flight check

Use Chrome MCP to confirm the dev environment is up before the tester opens the app.

Navigate to `[dev_url]` and take a screenshot:
```
mcp: navigate to [dev_url]
mcp: take screenshot
```

If the page loads (login screen or dashboard visible): print `✅ [dev_url] is up — ready to test.`

If the page fails to load or shows an error:
```
⏸ arc-qa-session — dev environment not responding
What I see: [error or blank page at dev_url]

Options:
  A. Try again in 1 minute
  B. Proceed anyway (risk: tests may give false FAILs)
  C. Abort
```

After confirming the environment is up, print this reminder once:

```
📋 Before we start:
  1. Open [dev_url] in your browser
  2. Press F12 → Console tab
  3. Keep DevTools docked and visible for the entire session
     (Console errors appear a few seconds after actions — keeping it open means nothing is missed)
```

Wait for the tester to confirm they are ready.

---

## Step 7 — TC loop

Work through test cases one at a time. For each TC:

### 7a — Generate the capture card

Read the TC row from the test plan table:
- TC number and title
- Steps column
- Expected Result column
- Current result (PASS/FAIL/BLOCKED/NOT TESTED)

If the current result is already **PASS**, skip it and print:
```
⏭  TC [N.M] — already PASS in test plan. Skipping.
```
Move to the next TC automatically.

Otherwise, generate the capture card:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TC [N.M] — [Title]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Stay on the relevant page when you say "done" — Claude will
    screenshot your current browser tab at that moment.

Account to use: [email from test accounts table]
Entry point:    [URL path]

Steps:
  [numbered steps derived from the Steps column of the TC row]

Look for and note down:
  [ ] [specific item derived from Expected Result — e.g. "Status badge text (e.g. 'Completed')"]
  [ ] [e.g. "Pass Rate value shown (e.g. '100.0%')"]
  [ ] [e.g. "Any on-screen error message or toast"]
  [ ] Any red errors in the browser console (DevTools → Console)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Perform the steps above, then say "done" when you are finished.
```

**Deriving "Look for" items:**
- Extract the key nouns and values from the Expected Result column (status text, field values, UI elements)
- Always include "any on-screen error message" and "any red errors in the browser console"
- Limit to 3–5 items — if the expected result is complex, pick the most important observable outcomes

### 7b — Wait for "done"

Wait for the tester to say "done" (or any equivalent like "finished", "completed").

### 7c — Chrome MCP: read console and screenshot

After the tester says "done":

```
mcp: read console messages from the current browser tab
mcp: take screenshot of current browser tab
```

Note any console errors found. Note the screenshot was captured.

### 7d — Tester reports observations

Ask the tester to share what they saw, based on the capture card items:

```
What did you see? (just the capture card items — e.g. "Status showed Completed, Pass Rate was 100%, no console errors")
```

Collect their response.

### 7e — Determine result

Based on tester's observations AND the console errors from 7c:

- **PASS** — tester saw the expected result, no relevant console errors
- **FAIL** — tester saw wrong result, unexpected error, or console showed a relevant error
- **BLOCKED** — tester could not perform the step (missing data, prerequisite unmet, environment issue)

If it is unclear from the observations, ask one clarifying question before deciding.

### 7f — Update the test plan immediately

Edit `[qa.test_plan_path]`:

1. In the TC row, update the Result column: `PASS` / `FAIL` / `BLOCKED`
2. Append to the Notes column (do not overwrite existing notes — add a new sentence):
   `[Result] [date] — [one sentence of what was observed]. Tested by: [display_name] ([gh_account]).`

Example note addition:
```
PASS 2026-06-11 — Status badge showed "Completed", Pass Rate 100.0%, no console errors. Tested by: Aditya (Aditya-qa-cons).
```

3. Update the Section Summary table at the bottom of the section:
   - Increment the count for the result (PASS / FAIL / BLOCKED)
   - Add the TC number to the Items column
   - Update the date line if this is the first result in the session: `**Tested: [date]**`

### 7g — FAIL: search for existing issue

If the result is FAIL:

```bash
gh issue list --repo [repo] --state open \
  --search "[TC title keywords]" \
  --json number,title,labels,url \
  --limit 5
```

Also search by the error text if the tester reported a specific message:
```bash
gh issue list --repo [repo] --state open \
  --search "[key error phrase]" \
  --json number,title,labels,url \
  --limit 5
```

**If a matching open issue is found**, add to the TC Notes: `See issue #[NNN].` and print:
```
🔗 Linked to existing issue #[NNN]: [title]
   No new issue needed.
```

**If no match found**, draft a new issue and ask for confirmation:
```
No existing issue found for this failure. Draft:

Title: [TC title] — [brief failure description]
Body:
  **TC:** [N.M] — [Title]
  **Section:** [N] — [Section name]
  **Steps:** [steps from capture card]
  **Expected:** [expected result from test plan]
  **Actual:** [tester's observation]
  **Console errors:** [from 7c, or "none"]
  **Tested on:** [dev_url] — [date]
  **Tester:** [display_name] ([gh_account])

File this issue? (y/n)
```

If yes:
```bash
gh issue create --repo [repo] \
  --title "[title]" \
  --body "[body]" \
  --label "bug"
```

Add the new issue number to the TC Notes: `Filed as issue #[NNN].`

### 7h — Save session state

After each TC, write to `.claude/task-state.md`:

```
Skill: arc-qa-session
Section: [N]
Last TC: [N.M]
Branch: [branch]
Results so far: [e.g. "5.1 PASS, 5.2 PASS, 5.3 FAIL (#NNN)"]
Next step: Present TC [N.next] capture card
Last saved: [YYYY-MM-DD HH:MM]
```

### 7i — Advance to next TC

Print:
```
✅ TC [N.M] recorded as [RESULT].
```

Then immediately generate the capture card for the next TC (Step 7a) without waiting.

**End of section:** When all TCs in the section are processed (or the tester says "done for today"), move to Step 8.

---

## Step 8 — Session End

Triggered by "done for today", "that's all for now", "stopping here", or when all TCs in the section are exhausted.

Print a session summary:
```
Session complete — Section [N]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PASS:    [count] — [TC numbers]
FAIL:    [count] — [TC numbers]
BLOCKED: [count] — [TC numbers]
Skipped: [count] — already passing

Issues filed: [list of #NNN, or "none"]
```

### 8a — Commit

```bash
git add [qa.test_plan_path]
git commit -m "$(cat <<'EOF'
docs(qa): Section [N] manual test results — [YYYY-MM-DD] [display_name]

[count] TCs tested: [count] PASS, [count] FAIL, [count] BLOCKED.
[If any issues filed: "Filed issues: #NNN, #NNN."]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

### 8b — Push and open PR

```bash
git push -u origin [branch]

gh pr create \
  --repo [repo] \
  --base develop \
  --head [branch] \
  --title "docs(qa): Section [N] manual test results — [YYYY-MM-DD]" \
  --body "$(cat <<'EOF'
## QA Session Results — Section [N]

**Tester:** [display_name] ([gh_account])
**Date:** [YYYY-MM-DD]
**Environment:** [dev_url]

| Result | Count | TCs |
|--------|-------|-----|
| PASS | [n] | [list] |
| FAIL | [n] | [list] |
| BLOCKED | [n] | [list] |

[If issues filed:]
**Issues filed:** [#NNN — title, #NNN — title]

Docs only — no code changes.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 8c — Handle review comments

Wait 30 seconds, then check for any automated review comments:

```bash
gh pr view [PR_NUMBER] --repo [repo] --json reviews,comments
```

If Gemini or CodeRabbit posted inline comments, run the fix-pr-issues skill automatically:
```
/fix-pr-issues [PR_NUMBER]
```

### 8d — Monitor CI and notify

```bash
gh pr checks [PR_NUMBER] --repo [repo] --watch
```

When all checks pass, print:

```
✅ PR #[NNN] is ready to merge — all CI checks passed.
   Say "merge it" to merge, or review it first at: [PR URL]
```

Wait for the tester to say "merge it" (or equivalent) before merging.

### 8e — Merge

```bash
gh pr merge [PR_NUMBER] --repo [repo] --squash --delete-branch
```

### 8f — Clear session state

```bash
rm -f .claude/task-state.md
```

Print:
```
✅ Done. Section [N] results are on develop.
   Next session: say "I want to test Section [N+1]" to continue.
```

---

## Step 9 — Append run log entry

Resolve the main repo root and operator name:

```bash
MAIN_ROOT=$(git worktree list --porcelain | head -1 | cut -d' ' -f2-)
OPERATOR=$(jq -r '.gh_account' "$MAIN_ROOT/.claude/workflow-config.local.json")
```

Append one JSON line to `"$MAIN_ROOT/.claude/run-log-${OPERATOR}.jsonl"` (create the file if it doesn't exist):

```json
{
  "ts": "<ISO 8601 timestamp>",
  "skill": "arc-qa-session",
  "section": <section number from Step 2>,
  "section_title": "<section heading from Step 5>",
  "tcs_total": <total TC rows in the section>,
  "tcs_run": <TCs presented as capture cards this session — excludes TCs skipped in Step 7a as already PASS>,
  "pass": <count>,
  "fail": <count>,
  "blocked": <count>,
  "gap": <count>,
  "issues_filed": <array of issue numbers filed in Step 7g, or [] if none>,
  "blocker_patterns": <array of short snake_case labels per distinct blocker root cause, or [] if none>,
  "branch": "<session branch from Step 4>",
  "pr": <PR number from Step 8b, or null if no PR was created>,
  "notes": "<one sentence on anything non-obvious>"
}
```

**Deriving `blocker_patterns`:** For each BLOCKED TC, extract a short snake_case label for the root cause from the notes recorded in Step 7f. Deduplicate — if multiple BLOCKEDs share the same root cause, use one label. Examples: `sample_size_zero_pre_pr628`, `no_defect_test_data`, `missing_completed_inspection`. Use `[]` if there are no blocked TCs.

**Deriving `gap`:** Count TCs recorded with result GAP (feature not implemented). GAP is distinct from BLOCKED.

Write this entry even if the session ended early (tester said "done for today" mid-section) — partial run data is still useful. If a field is unknown, use `null`.

---

## Escalation triggers

Pause and ask the user when:

**Trigger 1 — Prerequisites clearly unmet**
A prerequisite references a section that is not complete or data that does not exist. Surface in Step 5 with options.

**Trigger 2 — Dev environment down**
Chrome MCP cannot load `dev_url` in Step 6. Surface with retry/proceed/abort options.

**Trigger 3 — Ambiguous result**
Tester's observations partially match the expected result (e.g. page loaded but the specific value was different). Ask one clarifying question before recording.

**Trigger 4 — Console error unclear**
Chrome MCP returns console errors but it is not clear whether they are related to this TC or pre-existing. Show the errors to the tester and ask: "Are these related to this test case, or pre-existing?"

**Trigger 5 — CI failing on PR**
A CI check fails on the QA PR. Inspect the failure:
- If it is a lint/type error introduced by the test plan edit (unlikely but possible): fix it
- If it is a pre-existing failure unrelated to the test plan: note it in the PR and offer to merge anyway or wait for it to be fixed

---

## Edge cases

**Chrome MCP not available or browser not open:**
Skip steps 7c (console + screenshot). Ask the tester to manually report any console errors as part of their observations. Note "console read skipped — Chrome MCP unavailable" in the TC notes.

**TC is marked NOT TESTED (not BLOCKED):**
Treat the same as BLOCKED — present the capture card and attempt to test it.

**Tester interrupted mid-TC (session ends before reporting observations):**
task-state.md will show the last completed TC. On resume, re-present the capture card for the interrupted TC — do not record a partial result.

**Section has no untested TCs (all already PASS):**
Print: "All TCs in Section [N] are already PASS — nothing to test. Would you like to start a different section?"

**Two testers working the same day:**
If a branch for today already exists (`docs/qa-section-[N]-[other-account]-[date]`), warn:
```
⚠️  Another tester may be working on Section [N] today.
    Branch docs/qa-section-[N]-[other-account]-[date] already exists.
    To avoid merge conflicts, coordinate — one tester should finish and
    merge before the other starts, or you should work on different sections.
    Proceed anyway? (y/n)
```
