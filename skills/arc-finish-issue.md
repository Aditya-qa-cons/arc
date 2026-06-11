---
name: po-finish-issue
description: Use after merging PRs that close issues — reopens issues for QA, adds "Merged - Check" label, and writes verification comments with login instructions for the QA person. Run at end of session or after any PR merge.
---

> **If you see "isn't a recognized command here":** You're using the wrong app.
> Open the **Claude Code desktop app** (not claude.ai) — File -> Open Folder — select the repo folder, then retry.

# Issue Finishing

Process recently closed/fixed issues for QA handoff.

## Step 0 — Read config

```bash
cat .claude/workflow-config.json
cat .claude/workflow-config.local.json 2>/dev/null
```

Extract and keep in context:
- `repo` from project config — used in all `gh` calls below
- `gh_account` from local config — if missing, stop: "Run `/arc-setup` first."
- `labels.merged_check` — label to apply after merge

If `.claude/workflow-config.json` is missing, stop: "Not in the project folder — open the repo root in Claude Code and retry."

```bash
gh auth status  # Must show [gh_account] active; switch if not
```

## Workflow

1. **Identify issues** — Find issues closed or fixed in the current session (check git log, PR merges, and conversation context)
2. **For each issue:**
   a. **Fetch issue state in one call** — gather everything needed to decide what to do:
      ```bash
      gh issue view NNN --repo [repo] --json state,labels,comments \
        --jq '{state: .state, labels: [.labels[].name], has_qa_comment: ([.comments[] | select(.body | startswith("## Merged — Ready for QA Check"))] | length > 0)}'
      ```
   b. **Reopen** — if `state` is `CLOSED`, run:
      ```bash
      gh issue reopen NNN --repo [repo]
      ```
   c. **Add label** `Merged - Check` — only if `labels` from step (a) does not already contain `"Merged - Check"`:
      ```bash
      gh issue edit NNN --repo [repo] --add-label "Merged - Check"
      ```
   d. **Add verification comment** — write one using the process and template below.

      **If `has_qa_comment` is `false`:** post a new comment.

      **If `has_qa_comment` is `true`:** fetch the existing comment body and check whether it is generic (old format) or detailed (new format). A comment is **generic** if it is missing ALL THREE of:
      - A specific URL path line (`- URL: /...`)
      - A bodyText assertion (`Confirm in page text:` or `Confirm in snapshot bodyText:`)
      - A pw-act JSON block (a fenced ` ```json ` block inside the comment)

      If the existing comment is **generic**: rewrite it — post a new comment using the detailed template below, then note in the output: `ℹ️  #NNN — generic verification comment detected, rewrote with detailed template.`

      If the existing comment is already **detailed**: skip and note: `✅ #NNN — verification comment already in detailed format, skipping.`

## Test Users for Verification Comments

Read the `qa_users` array from workflow-config.json. Each entry has `role`, `email`, `env_var` (the password env var name — actual password is in `frontend/.env.test.local`), and `entry_path`.

When writing verification steps, look up the correct user by `role` and use their `email` and `env_var` in the step template.

## How to write the verification comment

Before writing, answer three questions from the PR diff and issue description:

**1. What exact URL does the fix affect?**
Every step must reference a specific path (`/inspector/history`, `/capa/NNN`, `/dashboard`), not a page description ("the history page"). If the fix affects a URL that only exists after some state (e.g., a CAPA detail page), note how to reach it.

**2. What specific text, value, or element confirms the fix worked?**
Every expected result must name something observable in the page — a string that appears in bodyText, a count that changes, an error that disappears. Avoid "verify it loads" or "check it works." Instead: "confirm bodyText contains 'N completed inspections'" or "confirm the error banner is absent from bodyText."

**3. Does any step create test data?**
If a step requires submitting a form, creating a record, or triggering a write operation, the comment must include a Teardown section with the cleanup action. If all steps are read-only (navigating and observing), omit the Teardown section entirely.

## Step classification

For each verification step, decide which type it is and write it accordingly:

**Read-only step** — navigate to a URL and check what's visible:
```
**Step N — [what you're checking]**
- URL: `/exact/path`
- Login as: `[email from qa_users config]` (`ENV_VAR_NAME`)
- Confirm in page text: "[exact string or partial string that must appear]"
- Confirm absent: "[text that must NOT appear, if relevant]"
```

**Interactive step** — requires clicking, filling a form, or multi-step in-page action:
```
**Step N — [what you're doing]**
- URL: `/exact/path`
- Login as: `[email from qa_users config]` (`ENV_VAR_NAME`)
- Interaction required — pw-act actions:
  ```json
  [
    {"type":"waitForSelector","selector":"[CSS selector]","timeout":10000},
    {"type":"fill","selector":"[input selector]","value":"[test value — use realistic content, min length ≥ required]"},
    {"type":"select","selector":"select","value":"[option label text]"},
    {"type":"click","text":"[button label]"},
    {"type":"waitForURL","pattern":"/expected-path/","timeout":12000},
    {"type":"snapshot","name":"step-N-result"}
  ]
  ```
- Confirm in snapshot bodyText: "[exact string that proves the action succeeded]"
```

**Tips for writing pw-act action arrays:**
- Use `waitForSelector` before `fill` or `select` to ensure the form is ready
- Use `waitForText` or `waitForURL` after `click` to confirm navigation or state change
- Use `snapshot` at the end of each meaningful interaction to capture evidence
- For `fill` values: use realistic strings that meet any minimum-length validation (e.g., descriptions ≥ 20 chars)
- For `select` values: use the visible option label, not the internal value
- Use `waitForURL` with `regex` field for negative patterns: `{"type":"waitForURL","regex":"/path/(?!new)","timeout":12000}`

## Comment template

```markdown
## Merged — Ready for QA Check

**PR**: #NNN (merged to develop)
**Change**: [1-2 sentence summary of what was fixed and why it matters]

### Verification

**Step 1 — [action name]**
- URL: `/exact/path`
- Login as: `[email from qa_users config]` (`ENV_VAR_NAME`)
- Confirm in page text: "[specific string]"

**Step 2 — [action name requiring interaction]**
- URL: `/exact/path`
- Login as: `[email from qa_users config]` (`ENV_VAR_NAME`)
- Interaction required — pw-act actions:
  ```json
  [
    {"type":"waitForSelector","selector":"[selector]","timeout":10000},
    {"type":"fill","selector":"[selector]","value":"[value with realistic content]"},
    {"type":"click","text":"[button text]"},
    {"type":"waitForText","text":"[confirmation text]","timeout":10000},
    {"type":"snapshot","name":"step-2-result"}
  ]
  ```
- Confirm in snapshot bodyText: "[text that proves the fix worked]"

### Teardown *(omit this section entirely if no test data is created)*

After verification, clean up test data created during the steps above:
- [Specific cleanup: e.g., "Navigate to `/capa/[id]` → click Delete → confirm deletion" or "No cleanup needed — all steps were read-only"]

**Pre-requisites**: [Firestore index deploy, Cloud Functions deploy, or "None"]
**Automated tests**: [X/X] frontend suites pass, [X/X] backend suites pass.
```

## For test-only or CI-only fixes

If the fix doesn't affect the UI:
- Skip the [dev_url] steps
- Instead provide the test command to run locally
- Still add the label and reopen

