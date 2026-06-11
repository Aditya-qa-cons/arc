# arc

AI-powered developer workflow skills for [Claude Code](https://claude.ai/code).

`arc` automates the fix → PR → deploy wait → QA loop. Run one command and Claude handles investigating the issue, opening the PR, waiting for deploy, verifying the fix on your dev environment, and closing the issue.

---

## What it does

| Skill | What it runs |
|---|---|
| `/arc-run` | Full loop — picks open issues, fixes them, opens PRs, waits for deploy, verifies QA |
| `/arc-fix-issue` | Fix a single issue end-to-end (investigate → branch → PR) |
| `/arc-verify-issue` | QA a merged fix on the dev environment using Playwright |
| `/arc-finish-issue` | After a PR merges — reopen issue for QA, add labels, post verification instructions |
| `/arc-start` | Session brief — syncs repo, shows open PRs and issues |
| `/arc-work-summary` | End-of-session summary (WhatsApp / Slack / Markdown) |
| `/arc-init` | First-time setup wizard — detects your project, generates `workflow-config.json` |
| `/arc-setup` | Machine setup — verifies gh auth, git hooks, labels, Playwright |

---

## Quickstart

### 1. Install

```bash
# Clone into your global Claude Code skills directory
git clone https://github.com/Aditya-qa-cons/arc ~/.claude/plugins/arc

# Or install as a plugin (if supported by your Claude Code version)
claude plugins install github:Aditya-qa-cons/arc
```

### 2. Copy skills into your project

```bash
cp ~/.claude/plugins/arc/skills/*.md .claude/commands/
```

### 3. Run the setup wizard

Open Claude Code in your project and run:

```
/arc-init
```

This detects your project (GitHub repo, hosting provider) and generates `.claude/workflow-config.json` with all the configuration arc needs. Then delegates to `/arc-setup` for machine verification.

### 4. Start working

```
/arc-run
```

---

## Supported hosting providers

| Provider | Deploy check method |
|---|---|
| Firebase App Hosting | Polls `gcloud app-hosting backends describe` for commit SHA |
| Vercel | Polls `vercel deployments` for commit SHA |
| Any (health endpoint) | Polls a `/health` or `/api/health` URL for a version field |

---

## Requirements

- [Claude Code](https://claude.ai/code)
- [GitHub CLI (`gh`)](https://cli.github.com) — authenticated with repo access
- [Node.js](https://nodejs.org) 18+
- Playwright + Chromium (`npx playwright install chromium`) — for `/arc-verify-issue`
- Provider CLI (optional): `gcloud` for Firebase projects, `vercel` CLI for Vercel projects

---

## Configuration

`arc` is driven by two files in `.claude/`:

- `workflow-config.json` — project config (committed to repo, shared by the team)
- `workflow-config.local.json` — machine config (gitignored, personal to each developer)

Run `/arc-init` to generate `workflow-config.json`. Run `/arc-setup` to create `workflow-config.local.json` and verify your machine.

See [`workflow-config.sample.json`](workflow-config.sample.json) for the full schema.

---

## License

MIT
