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

The monorepo is scaffolded and the core files are in place:

```
proxima-skills/
├── packages/
│   ├── flow/              # the skill plugin
│   │   ├── skills/
│   │   │   ├── triage/SKILL.md      ← complete; tested on ADO and GitHub
│   │   │   ├── plan/SKILL.md        ← stub; structure only
│   │   │   ├── status/SKILL.md      ← stub; structure only
│   │   │   ├── create-pr/SKILL.md   ← stub; structure only
│   │   │   └── track/SKILL.md       ← stub; structure only
│   │   ├── lib/           # ADO client, git helpers, config loader (not yet built)
│   │   └── templates/     # triage-template.md complete; pr-description.md + dod.md are stubs
│   └── cli/               # npx proxima-skills install flow (not yet built)
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
- **DoD template**: TBD — needs to be pulled from the company's current standard
- **Field mappings + state transitions**: TBD — depends on ADO project config; GitHub uses labels + milestone for equivalent fields
- **PR description format**: TBD — needs to confirm existing company convention

## PoC scope

- Target audience: all teams consuming Proxima
- Ticket trackers: **ADO and GitHub** (both in scope; ADO for company teams, GitHub for open-source / GitHub-native teams)
- Forward flow: `/triage` → `/plan` → implement → `/create-pr`
- Include `/track` to test the reverse-flow hypothesis

## What has been built and tested

| Skill | ADO | GitHub | Notes |
|---|---|---|---|
| `/triage` | ✅ Tested | ✅ Tested | Fully implemented. Tested against ADO work item #19941 and GitHub issue #1. |
| `/plan` | ✅ Tested | ✅ Tested | Tested end-to-end on both trackers. ADO: 5 child Tasks created and linked to parent #19941 via `az boards work-item relation add`. GitHub: 8-task checklist appended to issue #1 with re-estimate 5→8. |
| `/status` | 📝 Written | ✅ Tested | Tested on GitHub (issue #1): filtered git log to only commits containing `#1`, posted progress comment with structure in-progress/completed/next/blockers. ADO not yet tested (command `az boards work-item update --discussion` already proven by `/plan`). |
| `/create-pr` | 📝 Written | ✅ Tested | Tested on GitHub: opened draft PR #3 from feature branch. Skill proactively chose draft mode for partial work and `Relates to #1` over `Closes #1` to avoid auto-closing the issue. ADO side (`az repos pr create`) not yet tested — the one genuinely unverified ADO command. |
| `/track` | 📝 Written | ✅ Tested | Tested on GitHub: created retroactive issue #2 from two bootstrap commits, labels auto-created, issue closed with commit references. ADO not yet tested (commands already proven by `/plan`). |

Test environments confirmed working:
- **ADO**: `https://dev.azure.com/Epinova-Sweden`, project `Proxima Flow Sandbox`
- **GitHub**: `https://github.com/Epinova-Sweden/proxima-flow-sandbox`

## Open questions (unresolved)

The following still need a decision from a manager or colleague. See `CONTEXT.md` for the full log.

- Language/runtime for the CLI — TypeScript or Python?
- Who is the pilot team?
- Is there an existing company DoD / PR template to ship as default?
- Existing ADO PAT flow, or assume `az` CLI?

Resolved questions (see `CONTEXT.md` decisions log for details):
- ~~Which ADO org + project to target for the PoC?~~ → `Epinova-Sweden / Proxima Flow Sandbox`
- ~~Which GitHub org/repo to target for the PoC?~~ → `Epinova-Sweden / proxima-flow-sandbox`

Append dated decisions to the `## Decisions log` section of `CONTEXT.md` as these are resolved.
