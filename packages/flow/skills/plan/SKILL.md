---
name: plan
description: Produce an implementation plan for a triaged work item, break it into tasks, and re-estimate if scope has shifted. Supports Azure DevOps and GitHub. Run after /triage, before implementation starts.
---

# /plan \<id\>

Read a triaged work item, produce a concrete implementation plan, write it back, and re-estimate if the discovered scope differs from the current story-point value.

## When to use

Invoke with a work item or issue number: `/plan 42`

- A User Story or Issue has been triaged and is ready for planning
- You want to break the work into Tasks or a checklist before starting
- The estimate needs revisiting after a spike or more detailed analysis

## Prerequisites

**Azure DevOps:**
- Work item has been triaged (`/triage` run — description and ACs present)
- `az` CLI authenticated
- ADO org/project configured (see *Configuration*)

**GitHub:**
- Issue has been triaged (`/triage` run — body contains description and ACs)
- `gh` CLI authenticated
- `github.owner` and `github.repo` in `.claude/workflow-config.json`

## Steps

### 1. Load configuration

Read `.claude/workflow-config.json`. Extract:
- `tracker` (default: `"ado"`)
- If ADO: `ado.org`, `ado.project`
- If GitHub: `github.owner`, `github.repo`
- `plan.estimationScale` (default: `[0, 1, 2, 3, 5, 8, 13, 21]`)

### 2. Read the work item

**ADO:**
```bash
az boards work-item show --id {id} --expand all --output json
```
Extract: title, description, acceptance criteria, current story points, and any existing child Tasks.

**GitHub:**
```bash
gh issue view {id} --repo {owner}/{repo} --json number,title,body,labels,state
```
Extract: title, body (contains description and ACs from triage), and current labels.

If the item has not been triaged (no description or acceptance criteria), warn the developer and suggest running `/triage` first. Continue only if the developer confirms.

### 3. Analyse and plan

Using the acceptance criteria and any available codebase context:
- Identify the files, modules, and interfaces that need to change
- Break the work into discrete tasks — one per logical change or concern
- For each task: a short title, one-sentence description, and effort relative to siblings

Keep tasks independently completable where possible. Aim for 3–7 tasks for a typical User Story. For a small story (1–3 points), a simple checklist in the item body may be sufficient without creating separate child items.

### 4. Re-estimate if needed

If the discovered scope is materially different from the current story-point value, propose a revised estimate with a one-sentence rationale. Use the configured estimation scale.

### 5. Preview and confirm

Show the developer all proposed changes before writing anything.

**ADO preview:**
```
--- PLAN PREVIEW: #1234 ---

Title: Build user login page

Proposed Tasks:
  1. Set up authentication middleware
     Wire up session handling and JWT validation.
  2. Build login form component
     Input fields, validation, and error states.
  3. Write integration tests
     Cover happy path, invalid credentials, and locked account.

Revised estimate: 8 (was 5) — scope includes session handling not captured at triage.

Create these tasks and update the story? (yes / edit / cancel)
```

**GitHub preview:**
```
--- PLAN PREVIEW: #42 ---

Title: User can reset their password

Implementation plan (appended to issue body):
  ## Implementation Plan
  - [ ] Set up token generation and storage
  - [ ] Build password reset request form
  - [ ] Build password reset confirmation form
  - [ ] Wire up email delivery
  - [ ] Write integration tests

Revised estimate: 8 (was 5) — email delivery not in scope at triage.

Write plan to issue? (yes / edit / cancel)
```

Wait for explicit approval before writing anything.

### 6. Write back

**ADO — create Tasks:**

For each proposed task:
```bash
az boards work-item create \
  --type "Task" \
  --title "{task_title}" \
  --description "{task_description}"
```

Link each new task to the parent work item:
```bash
az boards work-item relation add \
  --id {task_id} \
  --relation-type parent \
  --target-id {parent_id}
```

If the estimate changed:
```bash
az boards work-item update --id {id} \
  --fields "Microsoft.VSTS.Scheduling.StoryPoints={revised_points}"
```

Post a summary comment:
```bash
az boards work-item update --id {id} \
  --discussion "Plan: created Tasks {task_ids}. Revised estimate to {points}."
```

**GitHub — append plan to issue body:**

Preserve the existing triage content and append the plan as a checklist:
```bash
gh issue edit {id} --repo {owner}/{repo} --body "{existing_body}

## Implementation Plan
- [ ] {task_1}
- [ ] {task_2}
- [ ] {task_3}

---
**Story Points:** {revised_points}"
```

If the estimate changed, update the story points value already in the body and replace the `sp:` label if present.

### 7. Confirm

Report what was written and show the item URL.

**ADO:** `{ado.org}/{ado.project}/_workitems/edit/{id}` — list Task IDs created
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
  "plan": {
    "estimationScale": [0, 1, 2, 3, 5, 8, 13, 21]
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
  "plan": {
    "estimationScale": [0, 1, 2, 3, 5, 8, 13, 21]
  }
}
```

## Notes

- Never skip the preview step.
- If the work item already has child Tasks (ADO) or an existing Implementation Plan section (GitHub), show them alongside the proposed additions and ask whether to merge or replace.
- Tasks should be ordered logically — dependencies and foundational work first.
- If the work item has not been triaged, warn the developer and offer to run `/triage` first.
- **ADO**: If `az boards work-item relation add` fails, create the tasks and provide the work item URL so the developer can link them manually in the ADO board.
- **ADO**: Descriptions are HTML by default; wrap generated text in `<p>` tags if the project does not use Markdown rendering.
