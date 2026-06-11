---
name: fix-pr-issues
description: Processes all unresolved inline review comments on a PR (Gemini, CodeRabbit, or human reviewers). For each comment: reads the actual code, evaluates the suggestion, applies valid fixes, replies to the thread. Takes a PR number as argument — e.g. /fix-pr-issues 553. If no number given, detects from the current branch.
---

# Fix PR Review Issues

Work through every unresolved inline review comment on a PR. Apply valid fixes, push back on incorrect ones, reply to every thread, and commit all changes in one commit.

## Step 1 — Read config + gh auth check

Read `repo` from `.claude/workflow-config.json`:
```bash
python3 -c 'import json; c=json.load(open(".claude/workflow-config.json")); print(c["repo"])'
```
Store as `REPO`. If the file is missing, stop: "`.claude/workflow-config.json` not found — run `/arc-init` first."

```bash
gh auth status
```

## Step 2 — Resolve the PR number

If a number was passed as argument, use it.

If not:
```bash
git branch --show-current
gh pr list --repo "$REPO" --state open \
  --head "$(git branch --show-current)" --json number --jq '.[0].number'
```

## Step 3 — Fetch all inline review comments

```bash
gh api "repos/$REPO/pulls/NNN/comments" \
  --paginate --jq '.[] | {id, in_reply_to_id, user: .user.login, path, line, original_line, diff_hunk, body}'
```

Then build the thread map:
- **Root comments** — entries where `in_reply_to_id` is null
- **Replies** — entries where `in_reply_to_id` is set (links back to a root comment id)

Read `github_account` from `.claude/workflow-config.local.json` (the current user's personal config). A thread needs attention if ALL of these are true:
1. Root comment author is NOT the value of `github_account` (i.e. from a reviewer, not us)
2. No reply in the thread is from `github_account` (we haven't responded yet)
3. `path` and `line` are not null (skip comments on outdated/deleted lines)

If zero threads need attention, print "Nothing to do — all review comments have been addressed." and stop.

## Step 4 — Process each thread

Work through unresolved threads ONE AT A TIME. For each:

### 4a — Understand the comment

Read the `body` and `diff_hunk`. The `diff_hunk` shows the surrounding code at the time the comment was written. Look at:
- What the reviewer is saying
- Which file and line it refers to
- Whether it is flagged as high/medium/low priority (Gemini uses `![high]`/`![medium]`/`![low]` image tags)

### 4b — Read the current code

```bash
# Read ~15 lines around the flagged line
```

Use the `Read` tool on `comment.path` starting from `max(1, comment.line - 7)` for 20 lines. The diff_hunk shows the old state; always check the current file before deciding.

### 4c — Evaluate the suggestion

Follow these rules (from the `superpowers:receiving-code-review` skill):

**Apply the fix when:**
- The reviewer identified a real bug (wrong key, missing transform, broken logic)
- The suggestion aligns with patterns used elsewhere in this codebase
- Applying it would not break existing tests or functionality

**Do NOT apply the fix when:**
- The suggestion is technically incorrect for this stack
- It adds code that is never called (YAGNI — grep to verify)
- It conflicts with an existing architectural decision
- It is a pure style preference with no functional impact
- You cannot verify the claim without more context — in that case, reply asking for clarification instead

**Never:**
- Apply a suggestion "just to be safe"
- Apply without reading the current code first
- Assume the reviewer is right without checking

### 4d — Apply or decline

**If applying:** Edit the file. Note what you changed.

**If declining:** Note the technical reason (e.g. "unused code — endpoint not called anywhere", "transformer already handles this", "would break existing test X").

## Step 5 — Reply to each thread

After deciding on each thread, reply to the root comment using the root comment's `id`:

```bash
gh api "repos/$REPO/pulls/NNN/comments/ROOT_COMMENT_ID/replies" \
  -X POST --field body='REPLY_TEXT'
```

Note: use `--field body=` (not `-f body=`) and single-quote the body to avoid backtick/shell interpolation issues.

**For applied fixes:**
> Fixed in [filename:line]. [One sentence describing what changed.]

**For declined suggestions:**
> Not applying — [technical reason]. [One sentence explaining the codebase reality that makes this wrong or unnecessary.]

**Never write:**
- "You're absolutely right!" / "Great catch!" / "Thanks for the feedback!"
- Vague non-answers like "Will keep this in mind"
- Apologies

Just state the outcome factually.

## Step 6 — Commit all changes

If any files were modified:

```bash
git add [only the files that were changed]
git commit -m "fix: address review comments on PR #NNN

[list the files changed and what was fixed, one line each]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push
```

If zero files were modified (all comments declined or already handled):
No commit needed.

## Step 7 — Print a summary

```
PR #NNN — X threads processed
  Applied (Y):
    - file.tsx:42 — [what was fixed]
    - ...
  Not applied (Z):
    - file.tsx:88 — [reason]
    - ...
```

## gh auth reminder

The root comment's `id` is what you use for replies — NOT the `in_reply_to_id`.
Always use `ROOT_COMMENT_ID/replies` endpoint, not the top-level PR comments endpoint.
