# @proxima/cli

> **Status**: Placeholder — not yet implemented.

This package will become the installer CLI:

```bash
npx proxima-skills install flow
```

It will copy (or symlink) the `packages/flow` plugin into the consuming team's repo under `.claude/plugins/proxima-flow/`, making all five skills available immediately.

## Planned behaviour

1. Detect whether Claude Code is installed in the target repo
2. Copy the `flow` plugin directory to `.claude/plugins/proxima-flow/`
3. Create a starter `.claude/workflow-config.json` pre-filled with the team's ADO org/project
4. Print next steps (configure PAT or `az login`, run `/triage <id>` to test)

## Language / runtime

TBD — see open questions in `CONTEXT.md`.
