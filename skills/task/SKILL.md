---
name: task
description: End-to-end ticket workflow. Use when the user invokes /task or says "start a new task", "new ticket", "set up a task", or "start work on a ticket". Collects ticket info and repos, plans, implements in worktrees, commits, opens PRs, monitors CI, and optionally deploys to dev.
version: 3.0.0
disable-model-invocation: true
---

# Task Skill

End-to-end workflow: **Collect → Plan → Implement → Commit → PR → CI → Deploy.**

---

## Phase 1: Collect Inputs

Use AskUserQuestion to collect the following in sequence:

1. **Ticket number** — e.g. `MP-1234`. This becomes the branch name.

2. **Ticket description** — Tell the user to paste the full ticket content (Jira, Linear, etc.) including acceptance criteria, context, and any technical notes. Multi-line paste is fine.

3. **Repo selection** — Scan `~/example/` for git repos:
   ```bash
   for d in ~/example/*/; do [ -d "${d}.git" ] && basename "${d%/}"; done | sort
   ```
   Show a numbered list and ask the user for space-separated numbers or `all`.

---

## Phase 2: Planning

Spawn a **single Plan agent** (subagent_type: `Plan`) with **worktree isolation** so the user's working directories are not disturbed.

```
Agent(
  subagent_type: "Plan",
  model: "opus",
  isolation: "worktree",
  description: "Plan ticket implementation",
  prompt: "..."
)
```

The Plan agent prompt should include:
- The full ticket description and ticket number
- The absolute paths of all selected repos
- Instruction to explore each repo's relevant code before planning
- Instruction to produce a structured plan with a dedicated section per repo, each containing:
  - Which files need to change and why
  - Specific implementation steps (not vague — concrete function/struct/endpoint level)
  - Any cross-repo coordination concerns (shared protos, API contracts, event schemas, etc.)
  - What does NOT need to change in that repo (if applicable)

Example Plan agent prompt:
```
You are planning implementation for the following ticket.

Ticket: MP-1234
Description:
[full ticket text pasted by user]

Repos involved:
- /Users/connor.marchand/example/repo-a
- /Users/connor.marchand/example/repo-b

Explore each repo's relevant code thoroughly before planning. Then produce a structured implementation plan with a section for each repo. Each section must include:
1. Files to change and the specific changes needed (function/struct/handler level detail)
2. Step-by-step implementation order
3. Cross-repo dependencies or coordination needed (proto changes, API contract changes, Kafka topic/event schema changes, etc.)
4. Any repo that needs no changes should be explicitly called out and removed from scope.

End the plan with a "Cross-repo coordination" section if any exists.
```

**After the Plan agent returns**, display the full plan to the user and ask:
> "Does this plan look correct? Reply with 'yes' to proceed, or describe any changes needed."

If the user requests changes, revise the plan yourself based on their feedback and ask again. Repeat until approved.

---

## Phase 3: Spawn Parallel Implementation Agents (Worktrees)

Once the plan is approved, spawn one `general-purpose` agent per in-scope repo **in a single message** so they run in parallel. Use **worktree isolation** so the user's working directories are not disturbed.

Each agent uses `isolation: "worktree"`:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  isolation: "worktree",
  description: "Implement in repo-name",
  prompt: "..."
)
```

The worktree gives each agent a clean copy of the repo on the default branch. The agent must:
1. Create and check out the ticket branch (e.g. `git checkout -b MP-1234`)
2. Implement the changes per the plan
3. **Lint**: run `golangci-lint run --fix && goimports -l -w .` to auto-fix lint issues
4. **Test**: run `go test ./...` and fix any failures before proceeding
5. Stage all changed files with `git add`
6. Commit with a short, fitting message: `TICKET: concise summary` (e.g. `MP-1234: add retry logic to webhook handler`). Do NOT include any Co-Authored-By lines or mention Claude/AI. Keep it to one line.
7. Push the branch: `git push -u origin MP-1234`
8. Report: worktree path, files changed, summary of what was done

**Important constraints:**
- NEVER commit or push to `main` or `master` — only push to the ticket branch
- If lint or tests fail, fix the issues and re-run until they pass before committing
- The commit author will be whoever is configured in the repo's git config — do not override it

Each agent receives:
- The ticket number (to use as the branch name)
- The full ticket description
- **Only the plan section relevant to their repo** (not the entire plan)
- The cross-repo coordination section if one exists
- Their repo's absolute path

Example agent prompt template:
```
Ticket: MP-1234
Repo: /Users/connor.marchand/example/REPO

You are working in a git worktree (isolated copy of the repo). Do the following:

1. git checkout -b MP-1234
2. Implement the plan below
3. Lint: golangci-lint run --fix && goimports -l -w .
4. Test: go test ./...  — if tests fail, fix and re-run until green
5. git add all changed files
6. git commit -m "MP-1234: concise summary of changes"
   - Do NOT add Co-Authored-By or mention Claude/AI
   - Keep the message short (one line)
7. git push -u origin MP-1234
8. Report back: worktree path, files changed, summary

--- Ticket Description ---
[full ticket text]

--- Your Implementation Plan ---
[repo-specific section from the plan]

--- Cross-Repo Coordination Notes ---
[cross-repo section from plan, if any]

Implement this plan in your repo. Do not modify other repos.
```

---

## Phase 4: Open Pull Requests

After all implementation agents complete, open a PR for each repo **in parallel** using `gh pr create`. Each PR should:

- **Title**: `TICKET-NUMBER: short summary` (under 70 chars)
- **Body**: structured using this template:

```
## Summary

Brief description of the change.

### Changes
- Bullet list of what was changed and why (derived from the agent's report)

## Ticket

[TICKET-NUMBER](https://example.atlassian.net/browse/TICKET-NUMBER)

## Test plan

- [ ] CI passes
- [ ] Manual testing items relevant to this change
```

Use a HEREDOC to pass the body:
```bash
gh pr create --title "MP-1234: short summary" --body "$(cat <<'EOF'
## Summary
...

### Changes
- ...

## Ticket
[MP-1234](https://example.atlassian.net/browse/MP-1234)

## Test plan
- [ ] CI passes
EOF
)"
```

Run `gh pr create` from within each repo's worktree directory (or use `--repo` flag). Collect and display all PR URLs to the user.

---

## Phase 5: Monitor CI

After PRs are created, monitor CI in a loop until all PRs have passing checks.

**Loop logic:**

1. For each PR, run:
   ```bash
   gh pr checks BRANCH --repo Stodge-Inc/REPO
   ```
2. Parse the output. Each PR is in one of three states: **pending**, **passing**, or **failing**.
3. Report current status to the user (e.g. "repo-a: pending, repo-b: passing, repo-c: failed (lint)").
4. **If all passing** → move to Phase 6.
5. **If any failing** → report which repo/check failed. Invoke the `fix-ci` skill for the failing repo/PR to attempt an automatic fix. After fix-ci completes, continue the loop.
6. **If any still pending** → wait 30 seconds, then poll again.

Keep looping until every PR is green. Do not exit the loop early or ask the user whether to continue — just keep monitoring.

---

## Phase 6: Offer Dev Deployment

Once all PRs have passing CI, ask the user:

> "All CI checks are passing. Would you like to deploy any of these PRs to dev for testing?"

- **Yes** — For each PR the user wants deployed, invoke the `deploy-pr-to-dev` skill with the PR URL/number and repo name.
- **No** — End the workflow. Display final summary of all PR URLs.

---

## Final Summary

At the end of the workflow (whether deploying or not), display:
- All PR URLs
- CI status for each
- Deployment status (if applicable)
- Any outstanding items or blockers
