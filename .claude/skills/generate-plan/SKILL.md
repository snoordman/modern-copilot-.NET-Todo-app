---
name: generate-plan
description: Generates an implementation plan for a feature request in this Todo app. Use when the user asks to plan a feature described in a spec file.
argument-hint: "[spec-file]"
disable-model-invocation: true
---

# Generate an implementation plan

Produce a plan that an implementing agent (see the `implement-plan` skill) can follow **without doing its own research**.
The implementing agent has a limited attention budget: include what it needs, leave out what it doesn't.
Don't write production code. Only write the plan file.

## 1. Read the spec

Read the feature request in `$ARGUMENTS`. If no file was given, read `spec.md` in the repository root.

Number every requirement in the spec before you start researching, so you can trace them later:

- `R1`, `R2`, ... for each item under "What we need" **and** each bullet under "Additional context".
- `BR1`, `BR2`, ... for each business rule hidden inside those requirements. Look for words like *must*, *only*, *unique*, *at most*, *no longer*, *may*, *own*. One requirement often holds several rules. For example: "names must be unique per user" is one rule, and implies that two different users *can* use the same name.

Take the slice name from the spec (for the current `spec.md` that is `PRJ004Projects`).

## 2. Research

Read the code. Don't rely on memory, and record `path:line` for everything you want the implementer to copy.

1. **Reference slices**: `Application/TSK001Tasks/` and `Application/CMT003Comments/`. Read the commands, queries, validators, handlers and `*FeatureSetup.cs`. Note how DI registration happens and how the setup gets called from `Program.cs`.
2. **Entities**: `Domain/Entities/` (`TaskItem.cs`, `Comment.cs`). Note factory methods, private setters, how domain events are raised and how ownership (`UserId`) is stored.
3. **Events**: `Domain/Events/` and the handlers in `Application/EventHandlers/`. Note how events get published after `SaveChanges`.
4. **Persistence**: `Infrastructure/AppDbContext.cs` (DbSets, entity configuration, indexes) and how migrations are added.
5. **Caching**: `Application/Behaviours/` (`CacheKeys.cs`, `ICacheableQuery`, `ICacheInvalidatingCommand`). Note which keys a command has to invalidate.
6. **Controllers**: `API/Controllers/TasksController.cs` and `CommentsController.cs`. Note routes, authorization, how the current user is resolved, and which HTTP status code maps to which outcome (201, 204, 400, 403/404, 409).
7. **BDD tests**: `tests/Application.Tests/Features/*.feature` (skip the generated `*.feature.cs` files), `StepDefinitions/` (especially `CommonSteps.cs`), `Hooks/` and `Support/`. Note the Gherkin style and the step phrases you can reuse.
8. **Existing work for the new slice**: check whether files for the slice already exist (for example `Application/PRJ004Projects/`, `Domain/Entities/Project.cs`, events). If they do, the plan says per file whether to **keep**, **change** or **replace** it. Don't silently plan to overwrite them.
9. **Frontend**: only if the spec asks for frontend work. Look at `frontend/src/api/`, `pages/` and `components/` for the task list and how it calls the API.

## 3. Write the plan

Save it to `docs/implementation-plans/<SliceName>.md`. For the current spec that is `docs/implementation-plans/PRJ004Projects.md`. If the file already exists, overwrite it, and mention that in your summary.

Use exactly these sections:

1. **Summary**: the feature in at most three sentences, plus the spec file the plan was made from.
2. **Conventions to mirror**: a table of *convention → `path:line` → one-line note*. Cover at least: slice layout, command/query + handler, validation, ownership check, domain event raise + publish, cache keys and invalidation, DbContext configuration, controller + status codes, feature file + step definitions.
3. **Tasks**: an ordered, numbered list. Every task has:
   - **Files**: concrete paths to create or change (no "follow existing patterns").
   - **Do**: what to change, with a pointer to the convention it copies.
   - **Covers**: the `R`/`BR` ids it implements.
   - **Validate**: one concrete check, such as `dotnet build TodoApp.sln`, `dotnet test tests/Application.Tests --filter "FullyQualifiedName~Projects"`, a migration command or `npm run build` in `frontend/`.
4. **Test scenarios**: one Gherkin scenario per business rule (`BR` id in the scenario title), in the style of the existing `.feature` files. Include the happy path **and** the "not allowed" case for every rule: other user's project, duplicate name, archived project, task that isn't yours, and so on. State the expected status code or error.
5. **Traceability**: two tables. *Requirement → task(s)* and *business rule → scenario(s)*.
6. **Open questions**: everything the spec leaves undecided, each with why it matters and which task it blocks. Don't guess. If a task needs a decision, the task says "blocked by Q*n*" instead of picking an answer.

## 4. Self-validation (run before saving)

Don't save the plan until every box is checked. Re-read the spec file **now** (don't work from memory), then check the draft against it:

- [ ] **Requirements → tasks**: every `R` id, including every "Additional context" bullet, appears in the *Covers* field of at least one task. No requirement is skipped because it is "implicit" (ownership, domain events, frontend).
- [ ] **Rules → scenarios**: every `BR` id has at least one test scenario, and every rule that forbids or limits something has a "not allowed" scenario with the expected status code.
- [ ] **Dependency order**: no task uses a file, type or table that a later task creates. The usual order is entity + events → DbContext + migration → cache keys → commands/queries/handlers → feature setup / DI → controller → BDD feature + steps → frontend.
- [ ] **Concrete tasks**: every task names files and has a runnable validation step.
- [ ] **Conventions are real**: every `path:line` you cite exists and points at what you claim. Spot-check them with a read.
- [ ] **No silent decisions**: every place where you had to choose something the spec doesn't state (status codes for forbidden access, unarchiving, renaming, what happens to tasks when a project is archived, etc.) is listed under Open questions instead of decided.
- [ ] **Traceability tables** match the tasks and scenarios exactly.

If a check fails, fix the plan and run the whole checklist again. Then save the plan.

## 5. Report

Print a short summary: the plan path, the number of tasks and scenarios, the open questions (verbatim) and any existing files the plan changes or replaces.
