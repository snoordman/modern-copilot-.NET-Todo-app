# PRJ004Projects: implementation plan

Source: `Feature/feature.txt` (Feature Request: Projects)

## 1. Summary

Users can create named projects (names unique per user) and group their own tasks into them. A task belongs to at most one project, can be moved between the user's own projects or removed from a project, and a project's task list reports how many of its tasks are completed. Projects can be archived: they disappear from the project list but keep their tasks. Every state change publishes a domain event, users can only see and change their own projects, and the frontend task list gets a project filter.

## 2. Conventions to mirror

| What | Copy the style of |
|---|---|
| Slice file layout: one `*Feature.cs` holding records and handlers, plus one `*FeatureSetup.cs` | `Application/TSK001Tasks/TasksFeature.cs:1-87`, `Application/TSK001Tasks/TasksFeatureSetup.cs:1-16` |
| Command record + `ICacheInvalidatingCommand` | `Application/TSK001Tasks/TasksFeature.cs:24-31` |
| Query record + `ICacheableQuery` (30 s TTL) | `Application/TSK001Tasks/TasksFeature.cs:10-14` |
| Handler constructor style (`AppDbContext`, `IPublisher`) and publishing after `SaveChangesAsync` | `Application/TSK001Tasks/TasksFeature.cs:33-49` |
| Input validation that throws `ArgumentException` | `Application/CMT003Comments/CommentsFeature.cs:25-31` |
| Not-found handling that throws `KeyNotFoundException` | `Application/TSK001Tasks/TasksFeature.cs:79-80` |
| Exception to HTTP status mapping: `ArgumentException`→400, `KeyNotFoundException`→404, `InvalidOperationException`→409 | `API/Middleware/ExceptionHandlingMiddleware.cs:32-38` |
| Cache key constants | `Application/Behaviours/CacheKeys.cs:6-13` |
| Domain event record (`INotification`) | `Domain/Events/TaskCreatedEvent.cs:1-6` |
| Event handler that logs | `Application/EventHandlers/TaskEventHandler.cs:1-29` |
| Entity style: public get/set props, nullable navigation | `Domain/Entities/Comment.cs:1-15` |
| Controller attributes, XML docs, `ProducesResponseType` | `API/Controllers/TasksController.cs:10-50` |
| Feature registration in DI | `Program.cs:41-43` |
| Gherkin style + Background | `tests/Application.Tests/Features/Comments.feature:1-21` |
| Step definitions (`ScenarioState`, `JsonElement` assertions) | `tests/Application.Tests/StepDefinitions/TaskSteps.cs:1-91` |
| Shared scenario state | `tests/Application.Tests/Support/ScenarioState.cs:1-12` |
| Frontend API module | `frontend/src/api/tasks.ts:1-23` |
| Frontend filter dropdown | `frontend/src/pages/TasksPage.tsx:80-95` |

**Identifying the current user (important).** No existing controller reads the user from the JWT. Every slice takes `UserId` in the request body (commands) or the query string (GETs). See `tests/Application.Tests/StepDefinitions/TaskSteps.cs:23-24` and `API/Controllers/TasksController.cs:59`. The test auth handler always returns `NameIdentifier = "1"` (`tests/Application.Tests/Infrastructure/TestWebApplicationFactory.cs:196-201`). The frontend uses the Keycloak `sub` claim as its `userId` (`frontend/src/context/AuthContext.tsx:50-55`), which does not map to the integer `User.Id`. This plan **follows the existing convention**: the handlers take `UserId` and enforce ownership by comparing it with `Project.UserId` / `TaskItem.UserId`. Only the controller decides where `UserId` comes from, so switching to claims later (see Open question Q1) changes the controller alone.

**Ownership failures** return **404** (`KeyNotFoundException`) in this plan, so a project's existence is not revealed to other users. The middleware has no 403 mapping. See Q2.

**InMemory does not enforce unique indexes** (tests and `Testing` env use `UseInMemoryDatabase`, `Program.cs:27-31`). The name-uniqueness rule must therefore be checked in the handler as well as by the database index.

## 3. Tasks

Run `dotnet build TodoApp.sln` after every task. It must succeed with no new warnings.

### Task 1: Domain entity `Project` and `TaskItem.ProjectId`
- **Create** `Domain/Entities/Project.cs` (namespace `TodoApp.Domain.Entities`):
  `int Id`, `string Name = string.Empty`, `int UserId`, `User? User`, `bool IsArchived = false`, `DateTime CreatedAt`.
  Do not add a `Tasks` collection navigation: the JSON responses would serialise it and it could create cycles.
- **Change** `Domain/Entities/TaskItem.cs`: add `int? ProjectId { get; set; }` and `Project? Project { get; set; }`.
- Validate: `dotnet build TodoApp.sln`.

### Task 2: Domain events
- **Create** in `Domain/Events/` (namespace `TodoApp.Domain.Events`, same shape as `TaskCreatedEvent.cs`):
  - `ProjectCreatedEvent.cs`: `record ProjectCreatedEvent(int ProjectId, string Name, int UserId) : INotification;`
  - `TaskAssignedToProjectEvent.cs`: `record TaskAssignedToProjectEvent(int TaskId, int ProjectId, int? PreviousProjectId, int UserId) : INotification;` (covers both first assignment and a move)
  - `TaskRemovedFromProjectEvent.cs`: `record TaskRemovedFromProjectEvent(int TaskId, int ProjectId, int UserId) : INotification;`
  - `ProjectArchivedEvent.cs`: `record ProjectArchivedEvent(int ProjectId, string Name, int UserId) : INotification;`
- **Create** `Application/EventHandlers/ProjectEventHandler.cs`: implements `INotificationHandler<>` for all four events and logs each with `_logger.LogInformation` (copy `TaskEventHandler.cs`). MediatR assembly scanning (`Program.cs:38`) registers it automatically.
- Validate: `dotnet build TodoApp.sln`.

### Task 3: DbContext and migration
- **Change** `Infrastructure/AppDbContext.cs`:
  - Add `public DbSet<Project> Projects => Set<Project>();`
  - Add `protected override void OnModelCreating(ModelBuilder modelBuilder)` that calls `base.OnModelCreating`, then:
    - `modelBuilder.Entity<Project>().Property(p => p.Name).IsRequired().HasMaxLength(100);` (max length: see Q5)
    - `modelBuilder.Entity<Project>().HasIndex(p => new { p.UserId, p.Name }).IsUnique();`
    - `modelBuilder.Entity<TaskItem>().HasOne(t => t.Project).WithMany().HasForeignKey(t => t.ProjectId).OnDelete(DeleteBehavior.SetNull);`
- **Create migration** from the repo root (no live database needed):
  ```bash
  dotnet tool install --global dotnet-ef   # not installed in the devcontainer yet
  ASPNETCORE_ENVIRONMENT=Development dotnet ef migrations add AddProjects --project TodoApp.csproj --output-dir Infrastructure/Migrations
  ```
  `ASPNETCORE_ENVIRONMENT=Development` is required. Without a connection string, `Program.cs:27-31` falls back to InMemory, which cannot generate migrations. Check that the generated `Infrastructure/Migrations/*_AddProjects.cs` creates the `Projects` table, adds the unique index `(UserId, Name)`, adds a nullable `ProjectId` column with FK and index to `Tasks`, and that `AppDbContextModelSnapshot.cs` is updated.
- Validate: `dotnet build TodoApp.sln`, then `dotnet test tests/Application.Tests/Application.Tests.csproj`. All existing tests must stay green; `TaskItem` JSON now also contains `projectId: null`.

### Task 4: Cache keys
- **Change** `Application/Behaviours/CacheKeys.cs`: add `public static string ProjectsByUser(int userId) => $"projects:user:{userId}";`
- Do **not** cache the project-tasks query (Task 5). A move from project A to B could not invalidate A's key, because the command only knows the new project id when `CacheKeysToInvalidate` is evaluated. Also, `CompleteTaskCommand` (`TasksFeature.cs:66-69`) does not know the project, so the completed count would go stale.
- Validate: `dotnet build TodoApp.sln`.

### Task 5: Slice `Application/PRJ004Projects/ProjectsFeature.cs`
Namespace `TodoApp.Application.PRJ004Projects`. Same usings as `TasksFeature.cs:1-6`.

1. **`CreateProjectCommand(string Name, int UserId) : IRequest<Project>, ICacheInvalidatingCommand`**
   - Invalidates `CacheKeys.ProjectsByUser(UserId)`.
   - Handler `CreateProjectHandler`:
     - `string.IsNullOrWhiteSpace(Name)` → `ArgumentException("Project name cannot be empty.")` (400). Trim the name before the checks below and before saving.
     - User not found → `ArgumentException($"User with ID {UserId} not found.")` (400, same as `CreateTaskHandler`).
     - Name already used by **this user** (including archived projects, see Q3; exact compare, see Q4) → `InvalidOperationException($"A project named '{name}' already exists.")` (409).
     - Save `new Project { Name, UserId, CreatedAt = DateTime.UtcNow }` and publish `ProjectCreatedEvent`. Return the project.
2. **`GetProjectsByUserQuery(int UserId) : IRequest<List<Project>>, ICacheableQuery`**
   - Key `CacheKeys.ProjectsByUser(UserId)`, TTL 30 s.
   - Handler `GetProjectsByUserHandler`: `_db.Projects.Where(p => p.UserId == UserId && !p.IsArchived).OrderBy(p => p.Name).ToListAsync()`.
3. **`AssignTaskToProjectCommand(int TaskId, int ProjectId, int UserId) : IRequest<TaskItem>, ICacheInvalidatingCommand`**
   - Invalidates `CacheKeys.AllTasks` and `CacheKeys.TasksByUser(UserId)`.
   - Handler `AssignTaskToProjectHandler`:
     - Task not found **or** `task.UserId != UserId` → `KeyNotFoundException($"Task with ID {TaskId} not found.")` (404).
     - Project not found **or** `project.UserId != UserId` → `KeyNotFoundException($"Project with ID {ProjectId} not found.")` (404).
     - `project.IsArchived` → `InvalidOperationException("Cannot assign a task to an archived project.")` (409). See Q6.
     - If `task.ProjectId == ProjectId`: return the task without saving or publishing.
     - Otherwise keep `previous = task.ProjectId`, set `task.ProjectId = ProjectId`, save, publish `TaskAssignedToProjectEvent(task.Id, ProjectId, previous, UserId)`, and return the task.
4. **`RemoveTaskFromProjectCommand(int TaskId, int ProjectId, int UserId) : IRequest<TaskItem>, ICacheInvalidatingCommand`**
   - Same invalidation as 3.
   - Handler `RemoveTaskFromProjectHandler`:
     - Task not found, owned by someone else, or `task.ProjectId != ProjectId` → `KeyNotFoundException($"Task with ID {TaskId} is not in project {ProjectId}.")` (404).
     - Set `task.ProjectId = null`, save, publish `TaskRemovedFromProjectEvent`, and return the task. Removing from an archived project is allowed.
5. **`GetProjectTasksQuery(int ProjectId, int UserId) : IRequest<ProjectTasksResult>`** (not cacheable, see Task 4)
   - Define in the same file: `public record ProjectTasksResult(int ProjectId, string Name, bool IsArchived, int TotalCount, int CompletedCount, List<TaskItem> Tasks);`
   - Handler `GetProjectTasksHandler`:
     - Project not found or not owned → `KeyNotFoundException` (404).
     - Archived projects are still readable (their tasks are kept).
     - `tasks = _db.Tasks.Where(t => t.ProjectId == ProjectId).OrderBy(t => t.Id)`.
     - `CompletedCount = tasks.Count(t => t.IsCompleted)`.
6. **`ArchiveProjectCommand(int ProjectId, int UserId) : IRequest<Project>, ICacheInvalidatingCommand`**
   - Invalidates `CacheKeys.ProjectsByUser(UserId)`.
   - Handler `ArchiveProjectHandler`:
     - Not found or not owned → `KeyNotFoundException` (404).
     - Already archived → return the project unchanged without publishing (idempotent, see Q7).
     - Otherwise set `IsArchived = true`, save, publish `ProjectArchivedEvent`, and return the project.
     - Tasks are **not** modified: they keep their `ProjectId`.

- Validate: `dotnet build TodoApp.sln`.

### Task 6: `Application/PRJ004Projects/ProjectsFeatureSetup.cs` and registration
- **Create** `ProjectsFeatureSetup.AddProjectsFeature(this IServiceCollection services)`. Register all six handlers with `AddScoped<IRequestHandler<TReq, TRes>, THandler>()`, as in `TasksFeatureSetup.cs:10-12`.
- **Change** `Program.cs`: add `using TodoApp.Application.PRJ004Projects;` and chain `.AddProjectsFeature()` after `.AddUsersFeature()` (`Program.cs:41-43`).
- Validate: `dotnet build TodoApp.sln`.

### Task 7: Controller `API/Controllers/ProjectsController.cs`
Same class attributes as `TasksController.cs:13-19`, route `api/v{version:apiVersion}/projects`, XML doc comments on every action. `userId` source: see the conventions note and Q1.

| Action | Route | Input | Sends | Success |
|---|---|---|---|---|
| `Create` | `POST /` | body `CreateProjectCommand { name, userId }` | `CreateProjectCommand` | 200 + project |
| `GetAll` | `GET /` | `[FromQuery] int userId` | `GetProjectsByUserQuery` | 200 + list (non-archived) |
| `GetTasks` | `GET /{id}/tasks` | `[FromQuery] int userId` | `GetProjectTasksQuery(id, userId)` | 200 + `ProjectTasksResult` |
| `AssignTask` | `PUT /{id}/tasks/{taskId}` | `[FromQuery] int userId` | `AssignTaskToProjectCommand(taskId, id, userId)` | 200 + task |
| `RemoveTask` | `DELETE /{id}/tasks/{taskId}` | `[FromQuery] int userId` | `RemoveTaskFromProjectCommand(taskId, id, userId)` | 200 + task |
| `Archive` | `PUT /{id}/archive` | `[FromQuery] int userId` | `ArchiveProjectCommand(id, userId)` | 200 + project |

Annotate with `[ProducesResponseType]` for 200, plus 400/404/409 where they apply.
- Validate: `dotnet build TodoApp.sln`, then run the app and check the endpoints in Swagger.

### Task 8: BDD tests
- **Change** `tests/Application.Tests/Support/ScenarioState.cs`: add `int OtherUserId`, `int CurrentProjectId`, and `Dictionary<string, int> ProjectIds = new()`, `Dictionary<string, int> TaskIds = new()`.
- **Change** `tests/Application.Tests/StepDefinitions/TaskSteps.cs` `GivenATaskExistsForTheUser`: also store `_state.TaskIds[title] = id`. This change is additive, so existing scenarios are unaffected.
- **Create** `tests/Application.Tests/Features/Projects.feature` with the scenarios in section 4 (feature title `Project Management`, generated class `ProjectManagementFeature`). Reqnroll generates `Projects.feature.cs` at build time.
- **Create** `tests/Application.Tests/StepDefinitions/ProjectSteps.cs` (copy the structure of `TaskSteps.cs`). The steps call the endpoints from Task 7 with `userId = _state.CurrentUserId` unless the step says "other user":
  - `Given another user {string} exists` → POST `/api/v1/users`, store `OtherUserId`.
  - `Given a project {string} exists for the user` / `... for the other user` → POST, `EnsureSuccessStatusCode`, store in `ProjectIds` and `CurrentProjectId`.
  - `Given a task {string} exists for the other user` → POST `/api/v1/tasks` with `OtherUserId`, store in `TaskIds`.
  - `Given task {string} is assigned to project {string}` → PUT assign, `EnsureSuccessStatusCode`.
  - `Given task {string} is completed` → PUT `/api/v1/tasks/{id}/complete`.
  - `Given project {string} is archived` → PUT archive, `EnsureSuccessStatusCode`.
  - `When I create a project {string}` → POST, store response.
  - `When I request my projects` / `When the other user requests their projects` → GET `/api/v1/projects?userId=`.
  - `When I assign task {string} to project {string}` → PUT.
  - `When I remove task {string} from project {string}` → DELETE.
  - `When I request the tasks of project {string}` → GET `/{id}/tasks`.
  - `When I archive project {string}` → PUT archive.
  - `When I request all tasks` already exists (`TaskSteps.cs:56`).
  - Then steps:
    - `the response should contain a project with name {string}`
    - `the response should contain {int} projects`
    - `the project list should not contain {string}`
    - `the task should belong to project {string}` (asserts `projectId` equals `ProjectIds[name]`)
    - `the task should not belong to any project` (asserts `projectId` is null)
    - `the project task list should contain {int} tasks of which {int} completed` (asserts `totalCount`, `completedCount` and the `tasks` array length)
- Validate: `dotnet test tests/Application.Tests/Application.Tests.csproj --filter "FullyQualifiedName~ProjectManagementFeature"`, then the full `dotnet test tests/Application.Tests/Application.Tests.csproj`.

### Task 9: Frontend project filter
- **Create** `frontend/src/api/projects.ts` (style of `frontend/src/api/tasks.ts`):
  - `interface Project { id: number; name: string; userId: number; isArchived: boolean; createdAt: string }`
  - `interface ProjectTasks { projectId: number; name: string; isArchived: boolean; totalCount: number; completedCount: number; tasks: Task[] }`
  - `getProjects(userId: string)`, `getProjectTasks(projectId: string, userId: string)`
- **Change** `frontend/src/api/tasks.ts`: add `projectId: number | null` to `Task`.
- **Change** `frontend/src/pages/TasksPage.tsx`:
  - Add `filterProjectId` state and a second `<select>` next to the user filter (`TasksPage.tsx:80-95`) with the option "All projects" plus `getProjects(filterUserId)`.
  - Enable it only when a user is selected, because projects are per user.
  - When a project is selected, use `useQuery({ queryKey: ['projectTasks', filterProjectId], queryFn: () => getProjectTasks(...) })`, show its `.tasks`, and show "X of Y completed".
  - Reset `filterProjectId` when `filterUserId` changes.
  - Also invalidate `['projectTasks']` in the existing `completeMutation.onSuccess`.
- **Change** `frontend/e2e/tasks.spec.ts`: add a test that checks the project filter dropdown is visible and lists "All projects".
- Validate: `cd frontend && npm run build && npm run lint`. Run `npm run test:e2e` if the stack is running.

### Task 10: Documentation
- **Change** `README.md`: add the `/api/v1/projects` endpoints to the API section, if there is one.
- Validate: review the rendered Markdown.

## 4. Test scenarios (`tests/Application.Tests/Features/Projects.feature`)

```gherkin
Feature: Project Management
  As a user of the Todo application
  I want to group my tasks into projects
  So that I can organise my work

  Background:
    Given the application is running
    And a user "projectuser" exists
    And I am authenticated as "projectuser"

  # Rule 1: create a project with a name
  Scenario: Create a project
    When I create a project "Home"
    Then the response status code should be 200
    And the response should contain a project with name "Home"

  Scenario: Create a project with an empty name fails
    When I create a project ""
    Then the response status code should be 400

  # Rule 1: names unique per user
  Scenario: Create a project with a duplicate name fails
    Given a project "Home" exists for the user
    When I create a project "Home"
    Then the response status code should be 409

  Scenario: Two users may use the same project name
    Given another user "otheruser" exists
    And a project "Home" exists for the other user
    When I create a project "Home"
    Then the response status code should be 200

  # Rule 2: assign a task to a project
  Scenario: Assign a task to a project
    Given a project "Home" exists for the user
    And a task "Paint fence" exists for the user
    When I assign task "Paint fence" to project "Home"
    Then the response status code should be 200
    And the task should belong to project "Home"

  # Rule 2: a task belongs to at most one project, so assigning it again moves it
  Scenario: Move a task to another of my projects
    Given a project "Home" exists for the user
    And a project "Garden" exists for the user
    And a task "Paint fence" exists for the user
    And task "Paint fence" is assigned to project "Home"
    When I assign task "Paint fence" to project "Garden"
    Then the response status code should be 200
    And the task should belong to project "Garden"
    When I request the tasks of project "Home"
    Then the project task list should contain 0 tasks of which 0 completed

  # Rule 2: take a task out of a project
  Scenario: Remove a task from a project
    Given a project "Home" exists for the user
    And a task "Paint fence" exists for the user
    And task "Paint fence" is assigned to project "Home"
    When I remove task "Paint fence" from project "Home"
    Then the response status code should be 200
    And the task should not belong to any project

  Scenario: Remove a task that is not in the project fails
    Given a project "Home" exists for the user
    And a task "Paint fence" exists for the user
    When I remove task "Paint fence" from project "Home"
    Then the response status code should be 404

  # Rule 2 + ownership: only into the user's own projects, only the user's own tasks
  Scenario: Assigning a task to another user's project is not allowed
    Given another user "otheruser" exists
    And a project "Secret" exists for the other user
    And a task "Paint fence" exists for the user
    When I assign task "Paint fence" to project "Secret"
    Then the response status code should be 404

  Scenario: Assigning another user's task to my project is not allowed
    Given a project "Home" exists for the user
    And another user "otheruser" exists
    And a task "Their task" exists for the other user
    When I assign task "Their task" to project "Home"
    Then the response status code should be 404

  Scenario: Assigning a task to a non-existent project fails
    Given a task "Paint fence" exists for the user
    When I assign task "Paint fence" to project id 9999
    Then the response status code should be 404

  # Rule 3: list a project's tasks including the completed count
  Scenario: List a project's tasks with the completed count
    Given a project "Home" exists for the user
    And a task "Paint fence" exists for the user
    And a task "Fix tap" exists for the user
    And a task "Not in project" exists for the user
    And task "Paint fence" is assigned to project "Home"
    And task "Fix tap" is assigned to project "Home"
    And task "Paint fence" is completed
    When I request the tasks of project "Home"
    Then the response status code should be 200
    And the project task list should contain 2 tasks of which 1 completed

  # Ownership: users may only see their own projects
  Scenario: Listing projects shows only my own projects
    Given a project "Home" exists for the user
    And another user "otheruser" exists
    And a project "Secret" exists for the other user
    When I request my projects
    Then the response status code should be 200
    And the response should contain 1 projects
    And the response should contain a project with name "Home"

  Scenario: Viewing the tasks of another user's project is not allowed
    Given another user "otheruser" exists
    And a project "Secret" exists for the other user
    When I request the tasks of project "Secret"
    Then the response status code should be 404

  # Rule 4: archive a project
  Scenario: An archived project no longer shows up in the project list
    Given a project "Home" exists for the user
    And a project "Old" exists for the user
    When I archive project "Old"
    Then the response status code should be 200
    When I request my projects
    Then the response should contain 1 projects
    And the project list should not contain "Old"

  Scenario: Archiving a project keeps its tasks
    Given a project "Old" exists for the user
    And a task "Legacy task" exists for the user
    And task "Legacy task" is assigned to project "Old"
    When I archive project "Old"
    And I request the tasks of project "Old"
    Then the response status code should be 200
    And the project task list should contain 1 tasks of which 0 completed

  Scenario: Assigning a task to an archived project fails
    Given a project "Old" exists for the user
    And project "Old" is archived
    And a task "New task" exists for the user
    When I assign task "New task" to project "Old"
    Then the response status code should be 409

  Scenario: Archiving another user's project is not allowed
    Given another user "otheruser" exists
    And a project "Secret" exists for the other user
    When I archive project "Secret"
    Then the response status code should be 404
```

The "project id 9999" scenario needs one more step: `When I assign task {string} to project id {int}`. Add it to `ProjectSteps.cs`.

Domain events are not asserted in the BDD tests: the existing features do not assert events either. Q9 covers whether they should be.

## 5. Open questions

The defaults this plan uses are marked **[default]**. Change the matching task if the answer differs.

1. **Q1: Where does the current user's identity come from?** The spec says "users may only see and change their own projects", but today every endpoint trusts a `userId` from the request, and the Keycloak `sub` (GUID) does not map to `User.Id` (int). Without a claim-to-user mapping, the ownership rule can be bypassed by sending a different `userId`. **[default]** Follow the existing convention (`userId` in the request). Handlers already enforce ownership, so moving to claims later only changes `ProjectsController`. Should this feature introduce the claim-to-user mapping, and for all controllers?
2. **Q2: 403 or 404 for another user's project or task?** **[default]** 404, so existence isn't leaked and no new middleware mapping is needed.
3. **Q3: Does name uniqueness include archived projects?** For example, can a user create "Home" again after archiving "Home"? **[default]** Yes, archived names still count, enforced by a plain unique index `(UserId, Name)`. If archived names may be reused, use a filtered index `HasFilter("\"IsArchived\" = false")` and a matching handler check.
4. **Q4: Is uniqueness case-insensitive and whitespace-trimmed?** ("Home" vs "home " ) **[default]** Trimmed, case-sensitive.
5. **Q5: Name validation limits** (maximum length, allowed characters). **[default]** Required, maximum 100 characters.
6. **Q6: Can tasks be assigned to an archived project?** **[default]** No (409). Removing tasks from an archived project is allowed.
7. **Q7: Archiving an already archived project.** **[default]** Idempotent: 200, no second event. Alternative: 409.
8. **Q8: Can archived projects be un-archived, renamed or deleted, and can archived projects be listed somewhere?** These are not in the spec and not planned. **[default]** Their tasks are still reachable via `GET /projects/{id}/tasks`.
9. **Q9: Who consumes the domain events?** **[default]** Logging only (`ProjectEventHandler`), like the existing features. Should the tests assert that events are published?
10. **Q10: Frontend scope.** The spec only asks for a filter by project. Should the UI also let users create projects, assign or move tasks, and archive projects? Without that, projects can only be created through the API. Should the filter offer a "No project" option? **[default]** Filter only, no "No project" option.
11. **Q11: Should the main task list (`GET /api/v1/tasks`) accept `?projectId=`?** The alternative is the frontend using `GET /projects/{id}/tasks`. **[default]** The frontend uses the project endpoint and `TasksController` is unchanged.
12. **Q12: Does a task have to be created inside a project?** Should `CreateTaskCommand` get an optional `ProjectId`? **[default]** No. Assign after creation.
13. **Q13: Existing cache staleness.** `CompleteTaskCommand` does not invalidate `TasksByUser` (`TasksFeature.cs:66-69`). This is a bug outside this feature and is not fixed here. Should it be fixed as part of this work?
