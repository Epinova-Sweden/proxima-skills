# proxima-skills

A Claude Code skill plugin that encodes the Proxima / Epinova dev lifecycle as terminal-based workflows integrated with Azure DevOps and GitHub.

## The five skills

| Command | Purpose | Status |
|---|---|---|
| `/triage <id>` | Enrich a vague work item — description, acceptance criteria, estimate, tags | ✅ Complete |
| `/plan <id>` | Produce an implementation plan; re-estimate if scope shifted | ✅ Complete |
| `/status <id>` | Post a progress update derived from git history | ✅ Complete |
| `/create-pr` | Generate a reviewer-focused PR and link it to the work item | ✅ Complete |
| `/track` | Retroactively create a work item from existing git history | ✅ Complete |

> **What works today:** All five skills are implemented and tested end-to-end against both Azure DevOps and GitHub. The one-line CLI installer and the company DoD / PR templates are still pending — until then, install manually using the steps below.

## Installation

Until the CLI is available, install manually. Run these commands from your team's repo root:

**1. Copy the skills into your repo:**
```bash
mkdir -p .claude/skills
cp -r /path/to/proxima-skills/packages/flow/skills/* .claude/skills/
```

**2. Create your config file:**
```bash
cp /path/to/proxima-skills/packages/flow/workflow-config.schema.json .claude/workflow-config.json
```
Open `.claude/workflow-config.json` and fill in your tracker details. For Azure DevOps:
```json
{
  "tracker": "ado",
  "ado": {
    "org": "https://dev.azure.com/your-org",
    "project": "Your Project"
  }
}
```
For GitHub:
```json
{
  "tracker": "github",
  "github": {
    "owner": "your-org",
    "repo": "your-repo"
  }
}
```

**3. Authenticate:**

Azure DevOps:
```bash
az login
az extension add --name azure-devops
az devops configure --defaults organization=https://dev.azure.com/your-org project="Your Project"
```
GitHub:
```bash
gh auth login
```

**4. Test the install** — open Claude Code in the repo and run:
```
/triage <work-item-or-issue-id>
```

## Configuration

Teams override plugin defaults by dropping `.claude/workflow-config.json` in their repo.
See [`packages/flow/workflow-config.schema.json`](packages/flow/workflow-config.schema.json) for the full schema.

## Repository structure

```
proxima-skills/
├── packages/
│   ├── flow/                        # The plugin
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json          # Plugin manifest
│   │   ├── skills/
│   │   │   ├── triage/SKILL.md
│   │   │   ├── plan/SKILL.md
│   │   │   ├── status/SKILL.md
│   │   │   ├── create-pr/SKILL.md
│   │   │   └── track/SKILL.md
│   │   ├── templates/
│   │   │   ├── triage-template.md
│   │   │   ├── pr-description.md    # stub — confirm company convention
│   │   │   └── dod.md               # stub — confirm company standard
│   │   └── workflow-config.schema.json
│   └── cli/                         # npx proxima-skills install flow (TBD)
├── CLAUDE.md
└── CONTEXT.md
```

## Design principles

1. Company-wide defaults, team-level overrides at every level
2. Terminal/IDE only — no browser context switch
3. Bi-directional — ticket-first and code-first flows
4. Progressive enrichment — each skill enriches the work item, no throwaway docs
5. Transparent — developer previews and approves every external write before it happens
