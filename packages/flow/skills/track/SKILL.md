---
name: track
description: Retroactively create a work item or issue from existing git history — for tech debt fixes, DX improvements, or unplanned work that was merged without a ticket. Supports Azure DevOps and GitHub.
---

# /track

Inspect recent git history, infer what was done and why, and create a properly-formed work item or issue that captures the work retroactively. Useful for code-first flows where implementation came before ticketing.

## When to use

Invoke without arguments: `/track`

- You fixed tech debt or a production bug without creating a ticket first
- A DX improvement was merged and you want it tracked for reporting
- You want to close the loop on unplanned work so the board reflects reality

## Prerequisites

**Azure DevOps:**
- `az` CLI authenticated
- Git repo with relevant recent commits
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
- `track.defaultType` (default: `"Task"`)
- `track.defaultState` (optional override; if unset, auto-detect — see Step 5)
- `track.defaultTags` (default: `["unplanned"]`)
- `track.areaPath`, `track.iterationPath` (ADO only, optional)

### 2. Collect git context

Show recent history and ask the developer which commits to cover:

```bash
git log --oneline -20
```

If the scope is obvious (e.g. all commits on the current branch), propose it. Otherwise wait for the developer to specify the commit range.

Once the range is identified:
```bash
git diff {start_commit}..{end_commit} --stat
```

### 3. Infer work item content

From the commits and diff, generate:

| Field | What to infer |
|---|---|
| **Type** | Task (default), Bug, or User Story — ask if ambiguous |
| **Title** | Concise, imperative title describing what was done |
| **Description** | What was done, why, and what problem it solves |
| **Acceptance Criteria** | Derived from the commits — frame as "Verified by: …" |
| **Story Points** | Fibonacci estimate based on observed effort |
| **Tags / Labels** | Component, type (`tech-debt`, `bugfix`, `dx`), team tags |

### 4. Preview and confirm

Show all generated fields before creating anything.

**ADO preview:**
```
--- TRACK PREVIEW ---

Type:        Task
Title:       Refactor auth middleware to remove deprecated JWT library
State:       Done

Description:
  Replaced the deprecated jsonwebtoken v8 dependency with the v9 API.
  Required updating token signing, verification, and error handling
  across the auth module.

Acceptance Criteria:
  Verified by: all existing auth tests pass with the new library (CI #847).

Story Points: 3
Tags:         tech-debt; auth; unplanned

Create this work item in ADO? (yes / edit / cancel)
```

**GitHub preview:**
```
--- TRACK PREVIEW ---

Title:       Refactor auth middleware to remove deprecated JWT library

Issue body:
  ## What was done
  Replaced the deprecated jsonwebtoken v8 dependency with the v9 API.
  Required updating token signing, verification, and error handling.

  ## Why
  The v8 library had known security vulnerabilities and was blocking
  a dependency upgrade.

  ## Verified by
  All existing auth tests pass with the new library (CI run #847).

  ---
  **Story Points:** 3

Labels: tech-debt, auth, unplanned

Create and immediately close this issue? (yes / edit / cancel)
```

### 5. Create the work item

**ADO:**
```bash
az boards work-item create \
  --type "{type}" \
  --title "{title}" \
  --description "{description}" \
  --fields \
    "Microsoft.VSTS.Common.AcceptanceCriteria={ac}" \
    "Microsoft.VSTS.Scheduling.StoryPoints={points}" \
    "System.Tags={tags}"
```

If area path or iteration path are configured, add them:
```bash
az boards work-item create \
  --type "{type}" \
  --title "{title}" \
  --description "{description}" \
  --area "{areaPath}" \
  --iteration "{iterationPath}" \
  --fields \
    "Microsoft.VSTS.Common.AcceptanceCriteria={ac}" \
    "Microsoft.VSTS.Scheduling.StoryPoints={points}" \
    "System.Tags={tags}"
```

Then transition to a terminal state.

ADO process templates differ — Agile uses `Closed`, Basic uses `Done`, Scrum uses `Done`, CMMI uses `Closed`. Auto-detect the valid state rather than hard-coding:

1. Query the valid states for this work item type:
   ```bash
   az boards work-item-type show \
     --process "{process}" \
     --type "{type}" \
     --query "states[].name"
   ```
   If `process` isn't known, fall back to listing states from an existing work item of the same type in the project.

2. Pick the terminal state using this priority:
   a. `track.defaultState` from config, if set and present in the returned list
   b. First match from: `Done`, `Closed`, `Completed`, `Resolved`
   c. If none match, show the valid states to the developer and ask which to use

3. Apply it:
   ```bash
   az boards work-item update --id {id} --state "{resolved_state}"
   ```

If `track.defaultState` is set but not in the valid list, warn the developer and fall back to the priority list above.

**GitHub:**

Check and create any missing labels first:
```bash
gh label list --repo {owner}/{repo}
gh label create "{label}" --repo {owner}/{repo} --color "#e4e669"
```

Create the issue:
```bash
gh issue create \
  --repo {owner}/{repo} \
  --title "{title}" \
  --body "{body}" \
  --label "{label1}" \
  --label "{label2}"
```

Immediately close it — the work is already done:
```bash
gh issue close {id} \
  --repo {owner}/{repo} \
  --comment "Work completed retroactively. Commits: {commit_hashes}."
```

### 6. Confirm

Report the created item ID and URL.

**ADO:** `{ado.org}/{ado.project}/_workitems/edit/{id}` (shown as Done)
**GitHub:** `https://github.com/{owner}/{repo}/issues/{id}` (shown as closed)

## Configuration

**ADO example:**
```json
{
  "tracker": "ado",
  "ado": {
    "org": "https://dev.azure.com/your-org",
    "project": "YourProject"
  },
  "track": {
    "defaultType": "Task",
    "defaultState": "Closed",
    "defaultTags": ["unplanned"],
    "areaPath": "YourProject\\TeamArea",
    "iterationPath": "YourProject\\Current Sprint"
  }
}
```

`defaultState` is optional — omit it to let the skill auto-detect the terminal state from the work item type. Set it only when you need to force a specific state (e.g. your process uses a custom terminal state).

**GitHub example:**
```json
{
  "tracker": "github",
  "github": {
    "owner": "your-org",
    "repo": "your-repo"
  },
  "track": {
    "defaultType": "Task",
    "defaultState": "Done",
    "defaultTags": ["unplanned"]
  }
}
```

## Notes

- Never create the work item without developer preview and approval.
- If multiple unrelated changes are found in the commit range, suggest creating separate items rather than one catch-all ticket.
- The created item is intentionally closed/Done — it exists for traceability, not active tracking.
- **GitHub**: GitHub does not have work item types like ADO. Use labels to convey type (`bug`, `tech-debt`, `enhancement`).
- **ADO state names vary by process template** (Agile → `Closed`, Basic/Scrum → `Done`, CMMI → `Closed`). The skill auto-detects valid states; set `track.defaultState` only if you need to force a specific one.
