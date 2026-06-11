---
name: arc-post-merge-check
description: Post-merge health check after a PR is squash-merged into develop. Verifies the deploy succeeded, CI is green on develop, and flags any issues. Takes a PR number as argument — e.g. /arc-post-merge-check 739. If no number given, asks the user which PR was just merged.
---

# Post-Merge Check

Runs after a PR is squash-merged into `develop`. Confirms the deploy pipeline ran, CI is clean, and surfaces any follow-up actions needed.

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

If a number was passed as argument, use it. Otherwise ask the user which PR was just merged.

```bash
gh pr view NNN --repo "$REPO" \
  --json state,title,mergedAt,mergeCommit,baseRefName \
  -q '{state, title, mergedAt, base: .baseRefName, sha: (if .mergeCommit then .mergeCommit.oid else null end)}'
```

Confirm `state` is `MERGED`. If not, stop and tell the user the PR is not yet merged.

## Step 3 — Confirm merge commit landed on develop

```bash
git fetch origin develop
git log --oneline origin/develop -5
```

Confirm the merge commit SHA from Step 2 appears in the log. If it does not, report "Merge commit not yet visible on develop — may still be propagating."

## Step 4 — Check CI on develop

```bash
gh run list --repo "$REPO" --branch develop --limit 8 \
  --json status,conclusion,name,createdAt,url \
  -q '.[] | {name, status, conclusion, createdAt, url}'
```

Look for runs triggered at or after `mergedAt`. Report each run:

| Workflow | Status | Conclusion |
|----------|--------|------------|
| Deploy Dev Environment | ... | ... |
| Backend | ... | ... |
| Frontend | ... | ... |
| ... | ... | ... |

Flag any `conclusion: failure` or `status: in_progress` runs.

## Step 5 — Check deploy succeeded

Look specifically for `Deploy Dev Environment` in the run list from Step 4.

- If `conclusion: success` and triggered after `mergedAt` → "Dev deployed successfully"
- If `conclusion: failure` → "Deploy FAILED — check run at [url]"
- If not triggered yet / still in_progress → "Deploy still running — check back in a few minutes"

## Step 6 — Check if any dependent PRs need rebasing

```bash
gh pr list --repo "$REPO" --base develop --state open \
  --json number,title,mergeable,headRefName \
  -q '.[] | select(.mergeable == "CONFLICTING") | {number, title, headRefName}'
```

List any open PRs that now have conflicts against develop. These need a rebase before they can merge.

## Step 7 — Print summary

```
Post-merge check — PR #NNN: [title]
Merged at: [mergedAt]

Deploy Dev Environment:  ✅ SUCCESS  /  ❌ FAILED  /  ⏳ IN PROGRESS
CI on develop:           ✅ All green  /  ❌ [list failures]

PRs needing rebase:      #NNN [title]  /  None

Next step: [one line — e.g. "Ready to test on dev" / "Fix deploy failure at [url]" / "Rebase PR #NNN before it can merge"]
```
