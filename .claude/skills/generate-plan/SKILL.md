---
name: generate-plan
description: Generates an implementation plan for a feature request in this Todo app. Use when the user asks to plan a feature described in a spec file.
argument-hint: "[spec-file]"
disable-model-invocation: true
---

# Generate an implementation plan

Produce a plan that an implementing agent can follow **without doing its own research**.
The implementing agent has a limited attention budget: include what it needs, leave out what it doesn't.

## Input

Read the feature request in `$ARGUMENTS`. If no file was given, read `spec.md` in the repository root.

## Research

1. Read the reference slices `Application/TSK001Tasks/` and `Application/CMT003Comments/`: commands, queries, handlers and `*FeatureSetup.cs`.
2. Read the matching entities in `Domain/Entities/`, events in `Domain/Events/`, controllers in `API/Controllers/` and the DbContext in `Infrastructure/AppDbContext.cs`.
3. Read the BDD features in `tests/Application.Tests/Features/` and their step definitions.
4. Note the conventions you must mirror: naming, folder layout, caching (`ICacheInvalidatingCommand`), domain events, error handling and HTTP status codes.

## Output

Save the plan as `docs/implementation-plans/<SliceName>.md`, for example `docs/implementation-plans/PRJ004Projects.md`. Use these sections:

1. **Summary**: the feature in three sentences.
2. **Conventions to mirror**: file references (`path:line`) to the code the implementer should copy the style of.
3. **Tasks**: an ordered list. Each task names the files to create or change, what to do, and how to validate it (`dotnet build TodoApp.sln`, a specific test).
4. **Test scenarios**: one Gherkin scenario per business rule, including the failure cases.
5. **Open questions**: everything the spec leaves undecided. Don't guess; list it.

## Self-validation (before you save)

Re-read the spec, then check the plan against it:

- [ ] Every requirement and every rule in "Additional context" maps to at least one task.
- [ ] Every business rule has a test scenario, including the "not allowed" cases.
- [ ] The task order respects dependencies: entity, then DbContext, then handlers, then controller, then tests.
- [ ] Every task names concrete files, not "follow existing patterns".
- [ ] Ambiguities are listed under Open questions instead of being decided silently.

Fix any gap, then save the plan and print a short summary with the open questions.
