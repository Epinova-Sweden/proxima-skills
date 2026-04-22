---
name: create-pr
description: Generate a reviewer-focused pull request from the current branch and link it to the associated work item or issue. Supports Azure DevOps and GitHub. Run when implementation is complete and ready for review.
---

# /create-pr

Inspect the current branch's commits and diff, generate a clear PR description for reviewers, open the PR, and link it to the work item or issue.

## When to use

Invoke without arguments from your feature branch: `/create-pr`

- Implementation is complete (or you want a draft PR for early feedback)
- The branch references a work item or issue ID
- You want a reviewer-friendly description without writing it from scratch

## Prerequisites

**Azure DevOps:**
- Current branch is not `main`/`master` and has commits ready to review
- `az` CLI authenticated with `az repos` access
- ADO org/project/repository configured (see *Configuration*)

**GitHub:**
- Current branch is not `main`/`master` and has commits ready to review
- Branch has been pushed to the remote (`git push`)
- `gh` CLI authenticated
- `github.owner` and `github.repo` in `.claude/workflow-config.json`

## Steps

### 1. Load configuration

Read `.claude/workflow-config.json`. Extract:
- `tracker` (default: `"ado"`)
- If ADO: `ado.org`, `ado.project`, `ado.repository`
- If GitHub: `github.owner`, `github.repo`
- `createPr.targetBranch` (default: `"main"`)
- `createPr.template` (path to custom PR description template)
- `createPr.draft` (default: `false`)
- `createPr.defaultReviewers` (default: `[]`)

### 2. Identify the linked work item or issue

Extract the ID from:
1. The branch name — e.g. `feature/42-password-reset` or `feature/AB#1234-login` → ID `42` or `1234`
2. Commit messages containing `#42`, `AB#1234`, `Fixes #42`, or `Closes #42`
3. If ambiguous or not found, ask the developer

Fetch the work item or issue to get the title, description, and acceptance criteria to inform the PR description.

### 3. Check the branch state

```bash
git log origin/{target_branch}..HEAD --oneline
git diff origin/{target_branch}..HEAD --stat
```

If there are no commits ahead of the target branch, report that and stop.

If the branch has not been pushed yet (GitHub), remind the developer:
```bash
git push -u origin {current_branch}
```

### 4. Generate PR description

Using the PR template (`createPr.template` or `templates/pr-description.md`), fill in:

- **Summary**: what this PR does and why (1–3 bullets)
- **Changes**: key files or modules changed, with brief notes on non-obvious choices
- **How to test**: how a reviewer can verify the changes (reference the ACs)
- **Work item link**: `AB#{id}` (ADO) or `Closes #{id}` (GitHub)
- **Notes for reviewers**: trade-offs made, areas needing extra attention

### 5. Preview and confirm

Show the complete PR title and description to the developer. Wait for approval or edits.

```
--- PR PREVIEW ---

Title:  Password reset flow (#42)

Description:
  ## Summary
  - Adds self-service password reset: request form, email delivery, confirmation form.
  - Closes #42

  ## Changes
  - `auth/reset.ts` — token generation, expiry, and single-use validation
  - `email/templates/reset.html` — reset email template
  - `pages/reset/` — request and confirmation form pages
  - `tests/reset.integration.ts` — end-to-end flow tests

  ## How to test
  1. Run `npm test` — all integration tests should pass
  2. Start the app, go to /login, click "Forgot password?"
  3. Submit a registered email — verify the email arrives within 2 minutes
  4. Use the link to set a new password — verify redirect and successful login

  ## Notes for reviewers
  - Token storage uses Redis with a 1-hour TTL (see `auth/reset.ts:42`)
  - User enumeration protection: same response shown for registered and unregistered emails

Open this PR? (yes / edit / cancel)
```

### 6. Open the PR

**ADO:**
```bash
az repos pr create \
  --title "{title}" \
  --description "{description}" \
  --source-branch {current_branch} \
  --target-branch {target_branch} \
  --draft {true|false}
```

If default reviewers are configured, add them:
```bash
az repos pr reviewer add --id {pr_id} --reviewers {reviewer1} {reviewer2}
```

Work item linking is handled automatically by `AB#{id}` appearing in the PR description — ADO auto-links it.

**GitHub:**
```bash
gh pr create \
  --title "{title}" \
  --body "{description}" \
  --base {target_branch} \
  --head {current_branch}
```

For a draft PR:
```bash
gh pr create \
  --draft \
  --title "{title}" \
  --body "{description}" \
  --base {target_branch}
```

Issue linking is handled by `Closes #{id}` in the PR body — GitHub auto-links it and will close the issue when the PR is merged.

### 7. Confirm

Report the PR URL and remind the developer to assign reviewers if none were set via config.

**ADO:** PR URL from the `az repos pr create` output
**GitHub:** PR URL from the `gh pr create` output

## Configuration

**ADO example:**
```json
{
  "tracker": "ado",
  "ado": {
    "org": "https://dev.azure.com/your-org",
    "project": "YourProject",
    "repository": "your-repo"
  },
  "createPr": {
    "targetBranch": "main",
    "draft": false,
    "defaultReviewers": ["user@company.com"]
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
  "createPr": {
    "targetBranch": "main",
    "draft": false
  }
}
```

## Notes

- Never open the PR without developer preview and approval of the description.
- If the branch has uncommitted changes, warn the developer before proceeding.
- **ADO**: `AB#{id}` in the description is the auto-linking syntax. Always include it.
- **GitHub**: `Closes #{id}` will auto-close the linked issue on merge. Use `Relates to #{id}` if you do not want that behaviour.
- **ADO**: If `az repos` commands fail, verify the `azure-devops` extension is installed: `az extension add --name azure-devops`.
