---
name: status
description: Post a progress update to a work item derived from recent git history. Supports Azure DevOps and GitHub. Use mid-sprint to keep the ticket current without switching to the browser.
---

# /status \<id\>

Summarise recent git commits relevant to a work item and post the summary as a comment — keeping stakeholders informed without a context switch to the browser.

## When to use

Invoke with a work item or issue number: `/status 42`

- Mid-sprint check-in: you want the work item to reflect current progress
- Before a standup: turn your git history into a readable update
- After a significant commit or PR merge

## Prerequisites

**Azure DevOps:**
- `az` CLI authenticated
- Git repo with commits relating to the work item
- ADO org/project configured (see *Configuration*)

**GitHub:**
- `gh` CLI authenticated
- `github.owner` and `github.repo` in `.claude/workflow-config.json`

## Steps

### 1. Load configuration

Read `.claude/workflow-config.json`. Extract:
- `tracker` (default: `"ado"`)
- If ADO: `ado.org`, `ado.project`
- If GitHub: `github.owner`, `github.repo`
- `status.gitLookbackDays` (default: `7`)
- `status.autoTransitionState` (default: `false`)

### 2. Read the work item

**ADO:**
```bash
az boards work-item show --id {id} --output json
```
Extract: title, current state, and the most recent discussion comments (to avoid repeating what's already been posted).

**GitHub:**
```bash
gh issue view {id} --repo {owner}/{repo} --json number,title,body,state,comments
```
Extract: title, state, and recent comments.

### 3. Collect relevant git history

```bash
git log --oneline --since="{gitLookbackDays} days ago"
```

Filter commits relevant to this work item by looking for:
- The ID (`#42` or `AB#1234`) in commit messages
- File paths or keywords matching the work item title or description

If no clearly relevant commits are found, report that and ask the developer to provide context manually before continuing.

### 4. Draft the status update

Produce a concise update (5–10 lines) covering:
- What was completed
- What is currently in progress
- Any blockers or open questions
- What comes next

Write for a non-technical stakeholder — avoid internal jargon and code-level detail.

### 5. Preview and confirm

Show the draft comment to the developer. Wait for explicit approval or edits before posting.

```
--- STATUS PREVIEW: #42 ---

Comment to be posted:

  Completed: password reset token generation and email delivery integration.
  In progress: building the reset confirmation form (2 of 3 UI states done).
  Blockers: none.
  Next: integration tests, then PR.

Post this comment? (yes / edit / cancel)
```

### 6. Post the update

**ADO:**
```bash
az boards work-item update --id {id} --discussion "{comment}"
```

**GitHub:**
```bash
gh issue comment {id} --repo {owner}/{repo} --body "{comment}"
```

If `autoTransitionState` is `true`, offer to transition the work item state (e.g. "New" → "Active", or "Active" → "In Review") and wait for explicit confirmation before doing so.

### 7. Confirm

Report the comment was posted and show the item URL.

**ADO:** `{ado.org}/{ado.project}/_workitems/edit/{id}`
**GitHub:** `https://github.com/{owner}/{repo}/issues/{id}`

## Configuration

Teams drop `.claude/workflow-config.json` in their repo to override defaults. Full schema: `packages/flow/workflow-config.schema.json`.

**ADO example:**
```json
{
  "tracker": "ado",
  "ado": {
    "org": "https://dev.azure.com/your-org",
    "project": "YourProject"
  },
  "status": {
    "gitLookbackDays": 7,
    "autoTransitionState": false
  }
}
```

**GitHub example:**
```json
{
  "tracker": "github",
  "github": {
    "owner": "your-org",
    "repo": "your-repo"
  },
  "status": {
    "gitLookbackDays": 7,
    "autoTransitionState": false
  }
}
```

## Notes

- Never post without developer preview and approval.
- If no relevant commits are found in the lookback window, say so clearly and ask the developer to provide context manually.
- Avoid repeating content already present in existing comments on the item.
- Keep the tone factual and brief — this is an update, not a report.
