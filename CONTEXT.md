# Proxima Flow — Project Context

> Living document. Append as decisions are made. Keep it scrappy, not polished.

## What I'm building

**Proxima Flow** — a Claude Code skill plugin (shape: [obra/superpowers](https://github.com/obra/superpowers)) that encodes our company's dev lifecycle as terminal-based workflows integrated with Azure DevOps and GitHub.

The plugin provides slash commands that guide developers through the full ticket lifecycle without leaving the IDE:

- `/triage <id>` — enrich a vague work item (description, acceptance criteria, estimate, tags)
- `/plan <id>` — produce an implementation plan, re-estimate if scope shifted
- `/status <id>` — post a progress update derived from git history
- `/create-pr` — generate a reviewer-focused PR and link it to the work item
- `/track` — retroactively create a work item from existing git history (tech debt, DX fixes)

Implementation is **not** a custom skill — if triage + plan are done well, "Implement work item #123" with existing complementary skills is enough.

## Why

Today, dev process lives in people's heads. Symptoms:
- Under-refined tickets → misalignment and rework
- Context switches (ADO ↔ IDE ↔ PR) break flow
- Status updates + ticket hygiene get skipped as overhead
- Tech debt work happens without tickets, untracked

## Core design principles

1. **Company-wide defaults, team-level overrides** at every level (workflow, step, template, field).
2. **Meet developers where they are** — terminal/IDE only, no browser context switch.
3. **Bi-directional** — ticket-first *and* code-first flows.
4. **Progressive enrichment** — each step enriches the work item, no throwaway docs.
5. **Transparent, not magic** — developer previews and approves every external post.

## Work item hierarchy

| Type | Scope | Plugin behavior |
|------|-------|-----------------|
| Feature | High-level goal, can span repos | `/triage` breaks it into User Stories. Not directly implementable. |
| User Story | Main dev unit, testable/deliverable | Primary target for `/triage` and `/plan`. Scoped to current repo. |
| Task | Optional child of User Story, technical | Can be planned. Created during `/plan` if needed. |

References between work items use ID references in descriptions/comments, **not** ADO's built-in linking.

## PoC scope

- Target: all teams consuming Proxima.
- Ticket trackers: **ADO and GitHub** — both in scope. ADO for company teams; GitHub for open-source / GitHub-native teams.
- Full forward flow: `/triage` → `/plan` → implement → `/create-pr`.
- Include `/track` to test reverse-flow hypothesis.

## Architecture sketch

```
proxima-skills/            # monorepo
├── packages/
│   ├── flow/              # the skill plugin
│   │   ├── skills/
│   │   │   ├── triage/SKILL.md
│   │   │   ├── plan/SKILL.md
│   │   │   ├── status/SKILL.md
│   │   │   ├── create-pr/SKILL.md
│   │   │   └── track/SKILL.md
│   │   ├── lib/           # ADO client, git helpers, config loader
│   │   └── templates/     # default triage template, PR template, DoD
│   └── cli/               # npx proxima-skills install flow
├── CONTEXT.md             # this file
└── README.md
```

- **Ticket tracker abstraction**: thin interface with two implementations — ADO (`az boards` CLI / ADO REST API) and GitHub (`gh` CLI / GitHub REST API). Both in scope. Teams specify `tracker: "ado"` or `tracker: "github"` in `workflow-config.json`.
- **Auth**: developer's own credentials — ADO uses PAT or `az` CLI; GitHub uses `gh` CLI or `GITHUB_TOKEN`. Plugin does not manage auth setup.
- **Config**: `.claude/workflow-config.json` in the consuming team's repo — company defaults shipped, teams only specify divergences.

## Defaults shipped with the plugin

- **Triage template**: Problem/Request → Expected behavior → Acceptance criteria
- **Estimation scale**: Fibonacci 0/1/2/3/5/8/13/21, **effort-based** (not time), same scale for human and AI work
- **DoD template** (TBD — need to pull the company's current one)
- **Field mappings + state transitions** (TBD — need to confirm per ADO project)
- **PR description format** (TBD — need to confirm existing convention)

## Next steps

These are the things that need to happen before the plugin is ready for a real team to use. They are listed roughly in order — some can happen in parallel, but the first two are blockers for everything else.

- [x] **1. Answer the open questions below** — ADO org/project confirmed. GitHub org/repo confirmed. A few questions (pilot team, templates, CLI language) still need a conversation with a manager or colleague.
- [x] **2. Try it against a real board (first real test)** — `/triage` confirmed working against ADO (work item #19941) and GitHub (issue #1, proxima-flow-sandbox). Both trackers verified end-to-end.
- [x] **3. Fill in the remaining four commands** — All five skills now verified end-to-end on **both** ADO and GitHub. Full forward flow (`/triage → /plan → /status → /create-pr`) and reverse flow (`/track`) proven against real boards and real git history. Two small polish items discovered — see decisions log.
- [ ] **4. Replace the placeholder templates** — The PR description and Definition of Done are currently generic placeholders. Replace with whatever format the company already uses.
- [ ] **5. Make it easy to install** — At the moment, installing requires copying files manually. Goal: a one-line command (`npx proxima-skills install flow`) so any team can get started without technical help.
- [ ] **6. Run it with a pilot team** — Pick one team, have them use it on real work for a sprint or two, and collect feedback against the success criteria below.

---

## Open questions for manager / colleague

- [x] Which ADO org + project do we target for the PoC? → **Epinova-Sweden / Proxima Flow Sandbox** (`https://dev.azure.com/Epinova-Sweden`)
- [x] Which GitHub org/repo do we target for the PoC? → **Epinova-Sweden / proxima-flow-sandbox** (`https://github.com/Epinova-Sweden/proxima-flow-sandbox`)
- [ ] Language/runtime for the CLI — TypeScript or Python?
- [ ] Who is the pilot team? Need real tickets to dogfood against.
- [ ] Is there an existing company DoD / PR template I should ship as the default?
- [ ] Do we already have an ADO PAT flow the team uses, or do we assume `az` CLI?

## Success criteria (from spec)

| Metric | Target |
|---|---|
| Work item description quality (peer-rated) | Measurably better than pre-plugin |
| Time from ticket creation → "ready for dev" | Reduced |
| Developer satisfaction with process overhead | Neutral or positive |
| Retroactive ticket creation rate (tech debt / DX) | Increased |

## Decisions log

_(append dated entries as decisions are made)_

- **2026-04-22** — ADO test environment: org `https://dev.azure.com/Epinova-Sweden`, project `Proxima Flow Sandbox`.
- **2026-04-22** — Confirmed correct install path for consuming repos: skills go in `.claude/skills/` (not `.claude/plugins/`). Claude Code picks up project-level skills from `.claude/skills/*/SKILL.md`.
- **2026-04-22** — GitHub test environment: org `Epinova-Sweden`, repo `proxima-flow-sandbox` (`https://github.com/Epinova-Sweden/proxima-flow-sandbox`).
- **2026-04-22** — Updated `/triage` skill to support both ADO and GitHub. Tracker is selected via `tracker` field in `.claude/workflow-config.json`. GitHub uses `gh issue view` / `gh issue edit`; all enriched content (description, ACs, story points) goes into the issue body as markdown sections; labels serve as tags.
- **2026-04-22** — Wrote full ADO + GitHub implementations for `/plan`, `/status`, `/create-pr`, and `/track`.
- **2026-04-22** — `/plan` tested end-to-end on both trackers. GitHub (issue #1): 8-task checklist appended to body, re-estimated 5→8. ADO (work item #19941): created 5 child Tasks (#19958–19962) and linked each to the parent via `az boards work-item relation add --relation-type parent --target-id ...`. **Confirmed: the `relation add` command works as written — no fallback to manual linking needed.**
- **2026-04-22** — Also retested `/triage` on ADO #19941 after the original triage content was missing. Worked correctly: warned the developer that planning without triage was risky and offered to run `/triage` first — good defensive behaviour.
- **2026-04-22** — Tested `/track`, `/status`, and `/create-pr` on GitHub end-to-end using seeded commits on `feature/1-password-reset`. `/track` grouped two bootstrap commits into a single issue (#2) and auto-closed it. `/status` correctly filtered git log to only commits referencing `#1` and posted a structured comment. `/create-pr` made two notable judgment calls beyond the written spec: it proactively offered draft mode for clearly-partial work, and chose `Relates to #1` over `Closes #1` to avoid auto-closing the issue on merge. Draft PR #3 opened successfully.
- **2026-04-22** — End of session. Full forward + reverse flow proven end-to-end on GitHub. ADO has `/triage` and `/plan` verified; `/status`, `/create-pr`, and `/track` written but not yet exercised (only `az repos pr create` is genuinely unverified — all other command paths have been proven). Next session pickup: Option A — mirror the three untested skills on ADO against work item #19941 (already triaged and planned). See *Pickup for next session* below.
- **2026-04-23** — Finished the ADO mirror. `/track` on ADO created Task #19972 from the logger commit; `/status 19941` filtered on `AB#19941` and posted a discussion comment; `/create-pr` opened PR !12126 via `az repos pr create`. Every command path in every skill now verified on both trackers — Step 3 of Next steps is complete.
- **2026-04-23** — **Finding:** `/track` default state `"Done"` is not valid in this ADO project — the project's process template uses `"Closed"` as the terminal state (likely Agile or CMMI). Claude improvised and retried with `"Closed"`, which worked. Follow-up work needed in the skill: either query valid states via `az boards work-item-type show` before writing, or make the default configurable per project via `track.defaultState` (schema already supports this).
- **2026-04-23** — **Finding:** `/create-pr` on GitHub proactively offers draft mode when the PR is clearly partial, but the ADO path does not. `az repos pr create` supports `--draft`, so this is just a consistency gap in the skill prompt for ADO. Small polish item.
- **2026-04-23** — Resolved `/track` state-mapping finding. Updated `track/SKILL.md` so `track.defaultState` is optional; Step 5 now queries valid states via `az boards work-item-type show` and picks the terminal state by priority (config override → `Done`/`Closed`/`Completed`/`Resolved` → ask developer). Config docs and Notes updated to match.
- **2026-04-23** — Resolved `/create-pr` draft-mode asymmetry. Added a new Step 4 "Decide draft vs ready" with explicit signals (WIP commits, unchecked ACs, failing tests, partial diffs, early work-item state) that apply to both ADO and GitHub. Preview now shows the draft decision and reason; ADO path has an explicit `--draft` example mirroring GitHub. Notes updated to document the proactive behaviour.

## Pickup for next session

**Where we left off:** End of day 2026-04-23. **Every skill verified end-to-end on both ADO and GitHub, and both polish items from today's testing are resolved.** Step 3 of Next steps is done; no unblocked work remains in the plugin itself.

**Natural next step:** move to Step 4 (templates), Step 5 (CLI installer), or Step 6 (pilot team). All three are blocked on decisions from a manager/colleague — see Open questions. Recommended first action next session: send a short message summarising the four open questions and asking for answers so Step 4+ can start.
- **YYYY-MM-DD** — _(example)_ Chose TypeScript for CLI because team is JS-heavy.

## Working notes

_(scratch space — rough thoughts, things to revisit, dead ends)_
