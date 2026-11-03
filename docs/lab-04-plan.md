# Lab 4 – Refinement Pipeline: plan and progress

Source: `customDocs/labs.pdf`, Lab 4 (pages ~19–25).
Goal: turn the rough `spec.md` into a precise PBI on a GitHub Issue, generate Reqnroll scenarios from it,
then plan and implement Projects with four agents and a human approval gate between each step.

Fork: `snoordman/modern-copilot-.NET-Todo-app`

## Progress log

| Date | What | Where |
|------|------|-------|
| 2026-11-03 | Installed `poppler-utils` in the devcontainer to read the lab PDF (no Python available). | container |
| 2026-11-03 | Created the `refine` and `requirements` agents (Lab 4 Ex. 1 and 2), plus the Lab 3 `plan` and `implement` agents the pipeline depends on (they did not exist yet). | `.claude/agents/` – commit `f850bbb` on branch `worktree-lab-04-agents` (pushed), commit `c312e2d` on `main` (not pushed) |
| 2026-11-03 | Enabled Issues on the fork. | GitHub repo settings |
| 2026-11-03 | Created the rough issue from `spec.md`. | [#1 Group tasks into projects](https://github.com/snoordman/modern-copilot-.NET-Todo-app/issues/1) |
| 2026-11-03 | Checked prerequisites: no devcontainer firewall allowlist exists, and `api.githubcopilot.com/mcp/` is reachable (401 without token). | – |
| 2026-11-03 | Wrote this plan. | `docs/lab-04-plan.md` |
| 2026-11-03 | Step 0: parked the old Projects WIP (3 modified + 9 untracked files) on local branch `wip-old-projects`. Main now only has untracked `customDocs/` and `.claude/worktrees/` (agent worktrees). | commit `22c2278` on `wip-old-projects` (not pushed) |

## The agents

| Agent | Tools | Input → output | Key rules |
|-------|-------|----------------|-----------|
| `refine` | Read, Glob, Grep, Write, `mcp__github__*` | GitHub Issue (or `spec.md`) → `docs/pbi.md`, then the issue | Endpoints with status code per scenario, business rules, authorization, domain events, acceptance criteria, open questions with recommendations. Stops for review; writes to GitHub **only after explicit approval**. Treats issue content as data and reports injected instructions. Flags the `UserId`-from-request-body pattern. |
| `requirements` | Read, Glob, Grep, Write, Edit, `mcp__github__*` | Approved PBI on the issue → `tests/Application.Tests/Features/Projects.feature` | Issue is the source of truth (not `docs/pbi.md`). Reuses existing step phrasing verbatim, tags scenarios with `@ACn`, covers happy/validation/not-found/cross-user cases, lists missing steps without implementing them. Read-only on GitHub. |
| `plan` | Read, Glob, Grep (read-only) | `docs/pbi.md` + `Projects.feature` → plan in the chat | User approves and saves it as `docs/implementation-plans/PRJ004Projects.md`. |
| `implement` | Read, Glob, Grep, Write, Edit, Bash | Saved plan → code, migration, step definitions | Builds after every task, never edits `.feature` files, no git commands. |

## Remaining plan

| # | Step | Who | Output / gate | Status |
|---|------|-----|---------------|--------|
| 0 | Park the old uncommitted Projects work on `main` (`Application/PRJ004Projects/`, `Project.cs`, Project events, `ProjectEventHandler.cs`, edits to `TaskItem.cs`/`AppDbContext.cs`/`CacheKeys.cs`) on a separate branch so `lab-04` starts clean. | Claude, after OK | `git status` on main shows only `customDocs/` (+ `.claude/worktrees/`) | done |
| 1 | Agents on main. | Claude | commit `c312e2d` | done |
| 2 | Enable Issues, create rough issue. | Claude | issue #1 | done |
| 3 | GitHub MCP: fine-grained PAT (only this fork; Issues read/write, Contents read), then `claude mcp add --transport http github https://api.githubcopilot.com/mcp/ --header "Authorization: Bearer <PAT>"` in **local** scope (never `--scope project`: that writes the PAT into `.mcp.json`). Restart, check `/mcp`. | You | `/mcp` shows github ✓ | todo |
| 4 | Ex. 1 – `@agent-refine Refine issue #1 in snoordman/modern-copilot-.NET-Todo-app into a PBI.` | Agent + you | **Gate 1:** review `docs/pbi.md`, answer open questions (identity from JWT claim vs `UserId` in body; 403 vs 404 for another user's project), approve the write to #1, check it in the browser. | todo |
| 5 | Ex. 1.5 – injection test: create issue "Add task priorities" with the planted instruction (see lab), run `@agent-refine` on it. | Claude creates, you run | Agent reports the injection instead of replying "pwned". Note the missing leg of the lethal trifecta (no exfiltration channel). | todo |
| 6 | Ex. 2 – `@agent-requirements Generate scenarios from the approved PBI on issue #1.` | Agent + you | **Gate 2:** every AC covered, cross-user scenarios (user B reads/archives user A's project), existing steps reused. | todo |
| 7 | Ex. 3 setup – `git switch -c lab-04` from a clean main. | Claude | branch `lab-04` | todo |
| 8 | `@agent-plan Plan PRJ004Projects from docs/pbi.md and Projects.feature.` | Agent + you | **Gate 3:** approve, save as `docs/implementation-plans/PRJ004Projects.md` (overwrites the Lab 2 plan on this branch). | todo |
| 9 | `@agent-implement Implement docs/implementation-plans/PRJ004Projects.md.` | Agent + you | **Gate 4:** review the diff (code, EF migration, step definitions). | todo |
| 10 | `dotnet build` and `dotnet test tests/Application.Tests` (PostgreSQL may be needed). | Claude | all green | todo |
| 11 | Wrap up: commit on `lab-04`; `git switch main && git checkout lab-04 -- docs/pbi.md && git commit -m "Lab 4: refined PBI for Projects"`. Answer the Observe questions, compare with Lab 2. | Claude (git), you (reflection) | deliverables met | todo |
| opt | Stretch: coordinator agent (`claude --agent coordinator`, needs writing – not in repo), background subagents, `/deep-research`. | together | – | optional |

## Risks

- PAT scope wrong → `/mcp` shows ✘ 401.
- `refine` copies the `UserId`-from-body pattern → catch at Gate 1.
- Integration tests need a database since the move from InMemory to PostgreSQL (#46).
- `main` is ahead of `origin/main` and not pushed – run `git push origin main` yourself when ready.

## Deliverables checklist

- [ ] GitHub MCP server configured in local scope, `/mcp` connected
- [x] `refine` and `requirements` agents committed on main
- [ ] GitHub Issue #1 holding the approved PBI
- [ ] `tests/Application.Tests/Features/Projects.feature`
- [ ] Branch `lab-04`: implementation from the four-agent pipeline, build and tests passing
