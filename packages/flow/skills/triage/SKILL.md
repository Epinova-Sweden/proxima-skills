---
name: triage
description: Enrich a vague work item — fills in description, acceptance criteria, story-point estimate, and tags. Supports Azure DevOps and GitHub. Use when given a work item or issue ID that needs enrichment before development starts.
---

# /triage \<id\>

Enrich a work item by reading its current state, analysing what's missing, generating a complete developer-ready description, and writing it back after developer approval.

## When to use

Invoke with a work item or issue number: `/triage 42`

- A ticket has a vague title and no description
- A Feature/Epic needs to be broken into User Stories or Issues
- A User Story or Issue needs acceptance criteria, an estimate, or tags

## Prerequisites

**Azure DevOps:**
- `az` CLI installed and authenticated (`az login` or `AZURE_DEVOPS_EXT_PAT` set)
- `az devops configure --defaults organization=<org> project=<project>` already run, **or** `ado.org` + `ado.project` present in `.claude/workflow-config.json`

**GitHub:**
- `gh` CLI installed and authenticated (`gh auth login`)
- `github.owner` and `github.repo` present in `.claude/workflow-config.json`

## Steps

### 1. Load configuration

Read `.claude/workflow-config.json` if it exists. Extract:
- `tracker` (default: `"ado"`)
- If ADO: `ado.org`, `ado.project` (fall back to `az devops configure` defaults)
- If GitHub: `github.owner`, `github.repo`
- `triage.template` (fall back to this skill's `../../templates/triage-template.md`)
- `triage.estimationScale` (default: `[0, 1, 2, 3, 5, 8, 13, 21]`)
- `triage.defaultTags` (default: `[]`)

### 2. Read the work item

**ADO:**
```bash
az boards work-item show --id {id} --output json
```
Extract: `id`, `System.WorkItemType`, `System.Title`, `System.Description`,
`Microsoft.VSTS.Common.AcceptanceCriteria`, `Microsoft.VSTS.Scheduling.StoryPoints`,
`System.Tags`, `System.State`, `System.TeamProject`.

**GitHub:**
```bash
gh issue view {id} --repo {owner}/{repo} --json number,title,body,labels,state,milestone
```
Extract: `number`, `title`, `body` (existing description/content), `labels` (existing tags), `state`.

If the item is not found or the command fails, report the error and stop.

### 3. Classify and decide

**ADO:**

| Type | Action |
|---|---|
| **Feature** | Break into User Stories (see *Feature flow*) |
| **User Story** | Enrich in place (see *Enrichment flow*) |
| **Task** | Enrich in place — brief description + 2–4 ACs, no story points |

**GitHub:** All issues use the *Enrichment flow*. If an issue is labelled as an Epic or is clearly a container for other issues, treat it like a Feature.

### 4. Enrichment flow

Using the triage template (loaded in step 1), generate:

| Field | What to produce |
|---|---|
| **Description** | Problem statement + expected behaviour. Max 3 short paragraphs. Plain language. |
| **Acceptance Criteria** | 3–7 specific, testable criteria. Prefer Given/When/Then or a numbered checklist. Each criterion must be independently verifiable. |
| **Story Points** | Pick from the estimation scale. Reflect *effort*, not clock time. If genuinely uncertain, state a range and ask the developer to decide. |
| **Tags / Labels** | Up to 5 tags derived from context (component, domain, type of work). Prepend any `triage.defaultTags` from config. |

**GitHub body format** — write all fields into the issue body as markdown sections:

```markdown
## Problem / Request
[description]

## Expected Behaviour
[expected behaviour]

## Acceptance Criteria
1. Given ... when ... then ...
2. ...

---
**Story Points:** 5
```

**ADO format** — description and acceptance criteria go into their own fields; story points go into `StoryPoints`; tags are semicolon-separated.

### 5. Feature flow (ADO only)

For each logical, sprint-sized slice of the Feature, draft one User Story:
- Title format: "As a [role], I want [goal] so that [value]"
- One repo, one sprint
- Brief description and at least 2 initial acceptance criteria

Present the full list of proposed stories for developer review **before creating anything**.
On approval, create each story and add a comment on the Feature listing the new story IDs.

### 6. Preview and confirm

Format all changes clearly and present them to the developer. **Do not write to the tracker yet.**

**ADO preview:**
```
--- TRIAGE PREVIEW: #1234 ---

Title:       [unchanged] Build user login page
Type:        User Story

Description:
  [generated text]

Acceptance Criteria:
  1. Given … when … then …
  2. …

Story Points: 5
Tags:         frontend; auth; epinova-team

Write these changes to ADO? (yes / edit / cancel)
```

**GitHub preview:**
```
--- TRIAGE PREVIEW: #42 ---

Title:       [unchanged] Build user login page

Issue body (replaces current body):
  ## Problem / Request
  [generated text]

  ## Expected Behaviour
  [generated text]

  ## Acceptance Criteria
  1. Given … when … then …
  2. …

  ---
  **Story Points:** 5

Labels to add: frontend, auth, epinova-team

Write these changes to GitHub? (yes / edit / cancel)
```

Wait for explicit approval ("yes", "go ahead", or equivalent) before proceeding.
If the developer wants edits, apply them and show the preview again.

### 7. Write back

**ADO:**
```bash
az boards work-item update --id {id} \
  --description "{description}" \
  --fields \
    "Microsoft.VSTS.Common.AcceptanceCriteria={acceptance_criteria}" \
    "Microsoft.VSTS.Scheduling.StoryPoints={story_points}" \
    "System.Tags={tags}"
```

**GitHub:**

First, check which labels already exist in the repo to avoid errors:
```bash
gh label list --repo {owner}/{repo}
```
Create any missing labels before adding them:
```bash
gh label create "{label}" --repo {owner}/{repo} --color "#0075ca"
```
Then update the issue:
```bash
gh issue edit {id} --repo {owner}/{repo} --body "{body}"
gh issue edit {id} --repo {owner}/{repo} --add-label "{label1}" --add-label "{label2}"
```

**ADO — Feature → Stories:** Create each story:
```bash
az boards work-item create \
  --type "User Story" \
  --title "{title}" \
  --description "{description}" \
  --fields \
    "Microsoft.VSTS.Common.AcceptanceCriteria={ac}" \
    "System.Tags={tags}"
```
Then post a comment on the Feature:
```bash
az boards work-item update --id {feature_id} \
  --discussion "Triage: created User Stories {ids} from this Feature."
```

### 8. Confirm

Report which fields were written and show the item URL.

**ADO:** `{ado.org}/{ado.project}/_workitems/edit/{id}`
**GitHub:** `https://github.com/{owner}/{repo}/issues/{id}`

## Configuration

Teams drop `.claude/workflow-config.json` in their repo to override defaults.
Full schema: `packages/flow/workflow-config.schema.json`.

**ADO example:**
```json
{
  "tracker": "ado",
  "ado": {
    "org": "https://dev.azure.com/your-org",
    "project": "YourProject"
  },
  "triage": {
    "estimationScale": [0, 1, 2, 3, 5, 8, 13, 21],
    "defaultTags": ["team-name"]
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
  "triage": {
    "estimationScale": [0, 1, 2, 3, 5, 8, 13, 21],
    "defaultTags": ["team-name"]
  }
}
```

## Notes

- Never skip the preview step — the developer must approve before any write.
- **ADO**: If `az` is not authenticated, prompt the developer to run `az login` or set `AZURE_DEVOPS_EXT_PAT`.
- **GitHub**: If `gh` is not authenticated, prompt the developer to run `gh auth login`.
- **ADO**: Descriptions are HTML by default; wrap generated text in `<p>` tags if the project does not use Markdown rendering.
- **GitHub**: Issue bodies are always Markdown — use the heading sections shown in step 4.
- **GitHub labels**: Labels that don't exist in the repo must be created before adding. Always check with `gh label list` first.
- Keep descriptions factual and concise — do not pad to hit a word count.
