---
name: implement-plan
description: Implements a feature by following an implementation plan from docs/implementation-plans/ step by step. Use after a plan has been generated and reviewed.
argument-hint: "[plan-file]"
disable-model-invocation: true
---

# Implement a plan

Implement the plan in `$ARGUMENTS`. If no file was given, use the most recent file in `docs/implementation-plans/`.

## How to work

1. Read the whole plan first. If it has open questions that block a task, stop and ask the user.
2. Work through the tasks **one at a time**, in order.
3. Mirror the conventions the plan points to. Don't invent new patterns.
4. After each task, run `dotnet build TodoApp.sln`. Fix compile errors before you start the next task.
5. Don't change files the plan doesn't mention, unless the build forces you to. Say so when you do.

## Self-validation (before you declare done)

Re-read the spec the plan was made from (`spec.md` unless the plan says otherwise), then check:

- [ ] Every endpoint exists with the planned route, HTTP method and status codes.
- [ ] Every business rule is enforced in the handler, not only in the controller.
- [ ] Every domain event from the plan is published.
- [ ] Every test scenario from the plan exists in `tests/Application.Tests/Features/` with step definitions.
- [ ] `dotnet build TodoApp.sln` succeeds.
- [ ] `dotnet test tests/Application.Tests` passes.

Fix anything that's missing. Then report what you did, what you deviated from and why, and anything you couldn't finish.
