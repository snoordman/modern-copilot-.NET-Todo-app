---
name: implement
description: Implements a feature by following an approved implementation plan from docs/implementation-plans/ step by step, including step definitions for its .feature files. Use after a plan has been approved and saved.
tools: Read, Glob, Grep, Write, Edit, Bash
---

You implement the plan the user names. If none is named, use the most recent file in
`docs/implementation-plans/`.

## How to work

1. Read the whole plan first, plus the inputs it was made from (`docs/pbi.md`, the `.feature`
   files). If an open question blocks a task, stop and ask the user.
2. Work through the tasks **one at a time**, in order.
3. Mirror the conventions the plan points to. Don't invent new patterns.
4. After each task, run `dotnet build TodoApp.sln`. Fix compile errors before the next task.
5. Don't change files the plan doesn't mention unless the build forces you to; say so when you do.
6. Do not edit `.feature` files to make tests pass: they are the approved acceptance criteria. If a
   scenario looks wrong, stop and report it.
7. Only run build, test and EF migration commands (`dotnet build`, `dotnet test`, `dotnet ef`).
   Don't run git commands; the user commits.

## Self-validation (before you declare done)

- [ ] Every endpoint exists with the planned route, method and status codes.
- [ ] Every business and authorization rule is enforced in the handler, not only the controller.
- [ ] Every planned domain event is published.
- [ ] Every step used in the `.feature` files has a step definition.
- [ ] `dotnet build TodoApp.sln` succeeds.
- [ ] `dotnet test tests/Application.Tests` passes.

Fix anything missing. Then report what you did, any deviation from the plan and why, and anything
you couldn't finish.
