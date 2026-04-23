# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

**Proxima Flow** — a Claude Code skill plugin (modelled after [obra/superpowers](https://github.com/obra/superpowers)) that encodes the company's dev lifecycle as terminal-based workflows integrated with Azure DevOps (ADO) and GitHub.

The plugin exposes five slash commands:

| Command | Purpose |
|---|---|
| `/triage <id>` | Enrich a vague work item — description, acceptance criteria, estimate, tags |
| `/plan <id>` | Produce an implementation plan; re-estimate if scope shifted |
| `/status <id>` | Post a progress update derived from git history |
| `/create-pr` | Generate a reviewer-focused PR and link it to the work item |
| `/track` | Retroactively create a work item from existing git history (tech debt, DX fixes) |

Implementation of actual work is **not** a skill — once triage + plan are done, "Implement work item #123" with existing complementary skills is enough.

## Design principles

1. **Company-wide defaults, team-level overrides** at every level (workflow, step, template, field).
2. **Terminal/IDE only** — no browser context switch.
3. **Bi-directional** — ticket-first *and* code-first flows.
4. **Progressive enrichment** — each step enriches the work item, no throwaway docs.
5. **Transparent** — developer previews and approves every external post before it happens.

## Work item hierarchy

| Type | Scope | Plugin behaviour |
|---|---|---|
| Feature | High-level goal, can span repos | `/triage` breaks it into User Stories. Not directly implementable. |
| User Story | Main dev unit, testable/deliverable | Primary target for `/triage` and `/plan`. Scoped to current repo. |
| Task | Optional child of a User Story | Can be planned. Created during `/plan` if needed. |

References between work items use ID references in descriptions/comments — **not** ADO's built-in link types.

## Repository structure

```
proxima-skills/
├── packages/
│   ├── flow/              # the skill plugin (5 skills, all tested on ADO + GitHub)
│   │   ├── skills/
│   │   │   ├── triage/SKILL.md
│   │   │   ├── plan/SKILL.md
│   │   │   ├── status/SKILL.md
│   │   │   ├── create-pr/SKILL.md
│   │   │   └── track/SKILL.md
│   │   └── templates/     # triage-template.md ready; pr-description.md + dod.md are placeholders
│   └── cli/               # npx proxima-skills install flow (placeholder — TBD)
├── CONTEXT.md
└── README.md
```

## Key architectural decisions

- **Ticket tracker abstraction**: thin interface with two implementations — ADO (`az boards` CLI / ADO REST API) and GitHub (`gh` CLI / GitHub REST API). Both are in PoC scope. Teams specify which to use via `tracker` in `workflow-config.json`.
- **Auth**: developer's own credentials — ADO uses PAT or `az` CLI; GitHub uses `gh` CLI (already authenticated) or `GITHUB_TOKEN`. The plugin does not manage auth setup.
- **Config**: `.claude/workflow-config.json` in the consuming team's repo. Company defaults are shipped with the plugin; teams only specify divergences.
- **Install path**: Skills are copied to `.claude/skills/` in the consuming repo (e.g. `.claude/skills/triage/SKILL.md`). Claude Code discovers skills from `.claude/skills/*/SKILL.md` automatically — no plugin registration needed for local use.

## Defaults shipped with the plugin

- **Triage template**: Problem/Request → Expected behaviour → Acceptance criteria
- **Estimation scale**: Fibonacci 0/1/2/3/5/8/13/21, effort-based (not time), same scale for human and AI work
- **ADO state transitions**: `/track` auto-detects valid terminal state per process template (Done/Closed/Completed/Resolved); override via `track.defaultState`. GitHub uses labels + milestone for equivalent fields.
- **PR description format**: placeholder — needs to confirm existing company convention
- **DoD template**: placeholder — needs to be pulled from the company's current standard

## PoC scope

- Target audience: all teams consuming Proxima
- Ticket trackers: **ADO and GitHub** (both in scope; ADO for company teams, GitHub for open-source / GitHub-native teams)
- Forward flow: `/triage` → `/plan` → implement → `/create-pr`
- Include `/track` to test the reverse-flow hypothesis

## What has been built and tested

| Skill | ADO | GitHub | Summary |
|---|---|---|---|
| `/triage` | ✅ Tested | ✅ Tested | Enriches description, ACs, estimate, tags. |
| `/plan` | ✅ Tested | ✅ Tested | Creates child Tasks (ADO) or a checklist (GitHub); re-estimates if scope shifted. |
| `/status` | ✅ Tested | ✅ Tested | Posts a progress comment filtered from recent git history. |
| `/create-pr` | ✅ Tested | ✅ Tested | Generates a reviewer-focused PR; proactively offers draft mode on both trackers. |
| `/track` | ✅ Tested | ✅ Tested | Retroactively creates + closes a work item from git history; auto-detects terminal state on ADO. |

Test environments:
- **ADO**: `https://dev.azure.com/Epinova-Sweden`, project `Proxima Flow Sandbox`
- **GitHub**: `https://github.com/Epinova-Sweden/proxima-flow-sandbox`

Detailed test findings and dated resolutions live in `CONTEXT.md`'s decisions log.

## Open questions (unresolved)

The following still need a decision from a manager or colleague. See `CONTEXT.md` for the full log.

- Language/runtime for the CLI — TypeScript or Python?
- Who is the pilot team?
- Is there an existing company DoD / PR template to ship as default?
- Existing ADO PAT flow, or assume `az` CLI?

Append dated decisions to the `## Decisions log` section of `CONTEXT.md` as these are resolved.
