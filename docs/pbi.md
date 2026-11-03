# PBI: Group tasks into projects (PRJ004Projects)

Source: GitHub issue snoordman/modern-copilot-.NET-Todo-app#1 "Group tasks into projects".

## 1. Title and user story

**Projects: group my tasks**

As a signed-in user with a growing list of tasks, I want to create projects, put my tasks into them and see per project how many tasks are done, so that I can organise my work and see progress per project.

## 2. Context

Today a task (`Domain/Entities/TaskItem.cs`) only has a title, a completion flag and an owner; there is no way to group tasks. This PBI adds a `Project` entity owned by a user, an optional link from a task to one project, a per-project task overview with a completed count, archiving, and a project filter in the frontend task list. It is a new vertical slice `Application/PRJ004Projects`, following `Application/TSK001Tasks` and `Application/CMT003Comments`.

## 3. Endpoints

General conventions (taken from the existing controllers):

- Route prefix `api/v{version:apiVersion}/...`, API version 1.0 (`API/Controllers/TasksController.cs:15-16`).
- `[Authorize]`, `[EnableRateLimiting("fixed")]`, `application/json` (`TasksController.cs:13-18`).
- Success returns **200 OK** with the resource in the body, including for creates (`TasksController.cs:29-35`, `CommentsController.cs:29-35`). See open question Q1.
- Errors are mapped by `API/Middleware/ExceptionHandlingMiddleware.cs:32-38` to `application/problem+json`: `ArgumentException` = 400, `KeyNotFoundException` = 404, `InvalidOperationException` = 409.
- Missing/invalid bearer token = **401** (from `[Authorize]`, verified by the existing auth tests).
- **No endpoint in this feature accepts a `userId`** in the body, query or route. The current user always comes from the authenticated principal (see section 5).

Project representation (`ProjectDto`), used in several responses:

| Field | Type | Notes |
|---|---|---|
| `id` | int | |
| `name` | string | Trimmed name as stored |
| `isArchived` | bool | |
| `createdAt` | string (ISO 8601 UTC) | |
| `archivedAt` | string (ISO 8601 UTC) or null | null while active |

Task representation is the existing `TaskItem` JSON (`id`, `title`, `isCompleted`, `userId`) extended with `projectId` (int or null).

### 3.1 Create a project

`POST /api/v1/projects`

Request body:

| Field | Type | Required | Constraints |
|---|---|---|---|
| `name` | string | yes | After trimming leading/trailing whitespace: 1 to 100 characters. Unique per user (BR2). |

Response: `200 OK` with `ProjectDto` (`isArchived: false`, `archivedAt: null`).

| Scenario | Status |
|---|---|
| Valid, unique name | 200 |
| `name` missing, null, empty or whitespace only | 400 |
| `name` longer than 100 characters after trimming | 400 |
| Current user already has a project with the same name (case-insensitive, after trimming), active or archived | 409 |
| Another user already has a project with that name | 200 (names are unique per user only) |
| Request contains a `userId` field | Ignored; project is owned by the current user (200) |
| No / invalid bearer token | 401 |
| Authenticated principal has no matching local user | 403 (see BR/AR and Q4) |

### 3.2 List my projects

`GET /api/v1/projects`

Request: no parameters.

Response: `200 OK` with an array of `ProjectDto`, only the current user's **non-archived** projects, ordered by `name` (case-insensitive, ascending).

| Scenario | Status |
|---|---|
| User has active projects | 200, array of their active projects only |
| User has no projects, or only archived ones | 200, empty array |
| Other users have projects | 200, their projects never appear |
| No / invalid bearer token | 401 |

### 3.3 List a project's tasks with completed count

`GET /api/v1/projects/{projectId}/tasks`

Request: `projectId` route parameter (int).

Response: `200 OK`

```json
{
  "project": { "id": 3, "name": "Home", "isArchived": false, "createdAt": "...", "archivedAt": null },
  "totalCount": 4,
  "completedCount": 1,
  "tasks": [ { "id": 10, "title": "Paint door", "isCompleted": true, "userId": 1, "projectId": 3 } ]
}
```

Tasks are ordered by task `id` ascending.

| Scenario | Status |
|---|---|
| Own active project with tasks | 200, counts match the tasks |
| Own project without tasks | 200, `tasks: []`, `totalCount: 0`, `completedCount: 0` |
| Own **archived** project | 200, tasks and counts returned, `project.isArchived: true` |
| Project does not exist | 404 |
| Project belongs to another user | 404 |
| `projectId` not an integer | 400 (model binding) |
| No / invalid bearer token | 401 |

### 3.4 Archive a project

`PUT /api/v1/projects/{projectId}/archive`

Matches the existing state-change style `PUT /tasks/{id}/complete` (`TasksController.cs:43`).

Request: no body.

Response: `200 OK` with the updated `ProjectDto` (`isArchived: true`, `archivedAt` set).

| Scenario | Status |
|---|---|
| Own active project | 200, project archived, tasks keep their `projectId` |
| Own project already archived | 200, no change, `archivedAt` unchanged, no event (idempotent, like completing an already completed task, `tests/Application.Tests/Features/TaskCompletion.feature:21-25`) |
| Project does not exist | 404 |
| Project belongs to another user | 404 |
| No / invalid bearer token | 401 |

### 3.5 Assign, move or remove a task's project

`PUT /api/v1/tasks/{taskId}/project`

Lives in `TasksController` (task route), handled by a command in the `PRJ004Projects` slice.

Request body:

| Field | Type | Required | Constraints |
|---|---|---|---|
| `projectId` | int or null | yes (key must be present) | `null` removes the task from its project. Otherwise must be one of the current user's non-archived projects. |

Response: `200 OK` with the updated task (including `projectId`).

| Scenario | Status |
|---|---|
| Own task without project, own active project | 200, `projectId` set |
| Own task in project A, move to own active project B | 200, `projectId` = B |
| Own task in a project, `projectId: null` | 200, `projectId` null |
| Own task without project, `projectId: null` | 200, no change, no event |
| Own task already in project A, `projectId` = A | 200, no change, no event |
| Body missing or `projectId` key missing | 400 |
| Task does not exist | 404 |
| Task belongs to another user | 404 |
| Project does not exist | 404 |
| Project belongs to another user | 404 |
| Target project is archived (own) | 409 |
| Task is currently in an archived project and is moved out (`null`) or to an active project | 200 (allowed) |
| No / invalid bearer token | 401 |

### 3.6 Frontend: filter task list by project

Not a new API endpoint. `frontend/src/pages/TasksPage.tsx` gets a project dropdown (options: "All projects" plus the user's active projects from 3.2). Selecting a project loads `GET /api/v1/projects/{id}/tasks` (3.3) and shows its tasks and "completed X of Y". "All projects" keeps the current behaviour. New API client file `frontend/src/api/projects.ts`, following `frontend/src/api/tasks.ts`. See Q6 for a "No project" option.

## 4. Business rules

- **BR1 Name required and length.** The name is trimmed of leading/trailing whitespace before validation and storage. After trimming it must be 1 to 100 characters, otherwise 400. Internal whitespace is kept as entered.
- **BR2 Name unique per user.** Two projects of the same user may not have names that are equal after trimming and case-insensitive comparison (invariant culture). "Home", " home " and "HOME" are duplicates (409). Archived projects count towards uniqueness (409 when re-using an archived project's name; see Q3). Different users may use the same name.
- **BR3 Ownership on create.** A new project is owned by the current user. Any client-supplied owner is ignored.
- **BR4 At most one project per task.** `TaskItem` gets a nullable `ProjectId`. A task is in zero or one project.
- **BR5 Same owner.** A task may only be linked to a project with the same owner as the task.
- **BR6 Move.** Assigning a task that is already in project A to project B replaces A with B in one operation.
- **BR7 Remove.** Assigning `null` takes the task out of its project; the task itself is not deleted or changed otherwise.
- **BR8 No-op assignment.** Assigning the project a task already has (including `null` to `null`) returns 200 and raises no event.
- **BR9 Archived target rejected.** A task cannot be assigned to an archived project (409). Moving a task out of an archived project is allowed.
- **BR10 Archive hides from list.** Archived projects are excluded from `GET /projects` (3.2).
- **BR11 Archive keeps tasks.** Archiving does not delete, complete or unlink tasks; they keep their `projectId` and stay visible in the normal task list and via 3.3.
- **BR12 Archive is idempotent.** Archiving an already archived project returns 200, keeps the original `archivedAt` and raises no event.
- **BR13 Completed count.** `completedCount` = number of the project's tasks with `isCompleted = true`; `totalCount` = number of the project's tasks. A project with no tasks returns 0 and 0 (never null, never an error).
- **BR14 Count freshness.** Counts and project task lists must reflect task creation, completion and (re)assignment immediately on the next request. Note: `CompleteTaskCommand` currently only invalidates `CacheKeys.AllTasks` (`Application/TSK001Tasks/TasksFeature.cs:66-69`), so if the project-tasks query is cached, completing a task must also invalidate that project's cache key (or the query must not be cached).
- **BR15 No archive reversal / delete.** There is no unarchive or delete in this PBI (see Out of scope).

## 5. Authorization rules

- **AR1 Identity source.** The current user is derived only from the authenticated principal, never from request input. The JWT name claim is `preferred_username` (`Program.cs:90`, `NameClaimType = "preferred_username"`), exposed as `User.Identity.Name`; it is matched to the local `User.Username` (`Domain/Entities/User.cs`). The Keycloak `sub` claim is not used because it is a Keycloak id, not the local integer `User.Id`.
- **AR2 Unknown local user.** If the principal is authenticated but no local `User` with that username exists, every endpoint in this PBI returns 403 (see Q4).
- **AR3 Own projects only.** A user can only list, read, archive and assign to their own projects.
- **AR4 Own tasks only.** A user can only change the project of their own tasks.
- **AR5 404 for other users' resources.** Reading or changing another user's project, or assigning another user's task, returns **404**, the same as a non-existent id. Reason: it does not reveal that the id exists, and it matches the existing `KeyNotFoundException` to 404 mapping (`ExceptionHandlingMiddleware.cs:35`).
- **AR6 Unauthenticated.** All endpoints require authentication; without a valid token they return 401.

**Existing pattern flagged (not to be copied):** the current slices let a caller act on behalf of another user:

- `CreateTaskCommand(string Title, int UserId)` is bound from the request body (`TasksFeature.cs:24`, `TasksController.cs:32`): any caller can create tasks for any user.
- `GET /api/v1/tasks` returns all users' tasks, or any user's tasks via `?userId=` (`TasksController.cs:57-71`).
- `CompleteTaskCommand` has no ownership check (`TasksFeature.cs:77-85`).
- `AddCommentCommand(int TaskItemId, int UserId, string Text)` takes `UserId` from the body (`CommentsFeature.cs:11`, `CommentsController.cs:32`).
- `UsersController` has no `[Authorize]` (`API/Controllers/UsersController.cs:12-17`).

The new feature must use AR1 instead. Fixing the existing endpoints is out of scope for this PBI (recommended as a separate PBI, Q7).

**Test infrastructure gap:** `TestAuthHandler` always authenticates as `NameIdentifier = "1"`, `Name = "testuser"` regardless of which user the scenario logged in as (`tests/Application.Tests/Infrastructure/TestWebApplicationFactory.cs:91-96`), while `Given I am authenticated as "<name>"` only stores the token (`tests/Application.Tests/StepDefinitions/AuthSteps.cs:22-35`). Cross-user scenarios (AC on 404 for another user's project) cannot be tested until the test handler takes the username from the bearer token (the fake token already contains `preferred_username`, `TestWebApplicationFactory.cs:128-129`). This change is in scope as test support.

## 6. Domain events

Following `Domain/Events/*Event.cs`: positional `record ... : INotification`, past-tense name, ids plus key data (`TaskCreatedEvent(int TaskId, string Title, int UserId)`, `CommentAddedEvent(...)`). Published via `IPublisher` after `SaveChangesAsync`, as in `TasksFeature.cs:45-46`. A handler that logs the event, like `Application/EventHandlers/TaskEventHandler.cs`, is added as `ProjectEventHandler`.

| Event | Raised when | Payload |
|---|---|---|
| `ProjectCreatedEvent` | A project was created (3.1, 200) | `int ProjectId, string Name, int UserId` |
| `ProjectArchivedEvent` | An active project was archived (3.4). Not raised for an already archived project (BR12). | `int ProjectId, int UserId` |
| `TaskProjectChangedEvent` | A task's project actually changed: assigned, moved or removed (3.5). Not raised for no-ops (BR8). | `int TaskId, int? PreviousProjectId, int? NewProjectId, int UserId` |

No event is raised for failed requests.

## 7. Acceptance criteria

Unless stated otherwise: Given user "alice" exists and I am authenticated as "alice".

- **AC1 (BR1, BR3)** Given I have no projects, When I create a project named "Home", Then the response is 200, the project has name "Home", `isArchived` false, and it is owned by alice.
- **AC2 (BR1)** When I create a project named "  Garden  ", Then the response is 200 and the stored name is "Garden".
- **AC3 (BR1)** When I create a project with name "" or "   " or no name, Then the response is 400.
- **AC4 (BR1)** When I create a project with a 101-character name, Then the response is 400; with a 100-character name it is 200.
- **AC5 (BR2)** Given I have a project "Home", When I create a project named "home " , Then the response is 409.
- **AC6 (BR2)** Given user "bob" has a project "Home", When I create a project "Home", Then the response is 200.
- **AC7 (BR2, Q3)** Given my project "Home" is archived, When I create a project "Home", Then the response is 409.
- **AC8 (BR3, AR1)** When I create a project "Work" with a body that also contains `"userId": <bob's id>`, Then the response is 200 and the project is owned by alice, not bob.
- **AC9 (BR10, AR3)** Given I have active projects "Work" and "Home", an archived project "Old", and bob has a project "Bob's", When I list my projects, Then the response is 200 and contains exactly "Home" and "Work" in that order.
- **AC10 (BR10)** Given I have no projects, When I list my projects, Then the response is 200 with an empty array.
- **AC11 (BR4, BR5)** Given I have a task "Paint door" without a project and a project "Home", When I assign the task to "Home", Then the response is 200 and the task's `projectId` is Home's id.
- **AC12 (BR6)** Given my task is in project "Home" and I have project "Work", When I assign the task to "Work", Then the response is 200, the task is in "Work", and it no longer appears in "Home"'s task list.
- **AC13 (BR7)** Given my task is in project "Home", When I assign it to `null`, Then the response is 200, the task's `projectId` is null and the task still exists.
- **AC14 (BR8)** Given my task is in project "Home", When I assign it to "Home" again, Then the response is 200 and no `TaskProjectChangedEvent` is raised.
- **AC15 (BR9)** Given my project "Old" is archived and I have a task without a project, When I assign the task to "Old", Then the response is 409.
- **AC16 (BR9)** Given my task is in archived project "Old", When I assign it to `null`, Then the response is 200.
- **AC17 (AR4, AR5)** Given bob has a task, When I assign bob's task to my project, Then the response is 404 and bob's task is unchanged.
- **AC18 (BR5, AR5)** Given bob has a project "Bob's" and I have a task, When I assign my task to "Bob's", Then the response is 404.
- **AC19 (AR5)** When I assign my task to project id 9999, Then the response is 404; when I assign task id 9999 to my project, Then the response is 404.
- **AC20 (BR13)** Given my project "Home" has tasks "A" (completed), "B" and "C" (not completed), When I get Home's tasks, Then the response is 200 with 3 tasks, `totalCount` 3 and `completedCount` 1.
- **AC21 (BR13)** Given my project "Empty" has no tasks, When I get its tasks, Then the response is 200 with `tasks` empty, `totalCount` 0 and `completedCount` 0.
- **AC22 (BR14)** Given my project "Home" has 1 open task and I requested Home's tasks (`completedCount` 0), When I complete that task and request Home's tasks again, Then `completedCount` is 1.
- **AC23 (AR3, AR5)** Given bob has a project "Bob's", When I get the tasks of "Bob's", Then the response is 404; When I archive "Bob's", Then the response is 404 and it stays active.
- **AC24 (AR5)** When I get the tasks of project id 9999, Then the response is 404.
- **AC25 (BR10, BR11)** Given my project "Home" has 2 tasks, When I archive "Home", Then the response is 200 with `isArchived` true, "Home" no longer appears in my project list, both tasks still exist with `projectId` = Home's id, and getting Home's tasks returns 200 with both tasks.
- **AC26 (BR12)** Given my project "Home" is archived, When I archive it again, Then the response is 200, `archivedAt` is unchanged and no `ProjectArchivedEvent` is raised.
- **AC27 (AR6)** Given I am not authenticated, When I call any of the endpoints 3.1 to 3.5, Then the response is 401.
- **AC28 (Events)** When I create a project, Then a `ProjectCreatedEvent` with its id, name and my user id is published; when I archive an active project, a `ProjectArchivedEvent` is published; when I move a task from "Home" to "Work", a `TaskProjectChangedEvent` with PreviousProjectId = Home and NewProjectId = Work is published.
- **AC29 (Frontend, 3.6)** Given I have projects "Home" and "Work" with tasks, When I select "Home" in the task list's project filter, Then only Home's tasks are shown with "completed X of Y"; When I select "All projects", Then the unfiltered list is shown again. Archived projects do not appear in the dropdown.

## 8. Out of scope

- Unarchiving, renaming or deleting projects.
- Project descriptions, colours, ordering, sharing projects between users.
- Assigning a project while creating a task (`POST /tasks` stays as is); assignment is done with 3.5.
- Bulk assignment of several tasks at once.
- Fixing the existing on-behalf-of issues in Tasks, Comments and Users endpoints (section 5), apart from the test auth handler change.
- Pagination of project or task lists.

## 9. Open questions

| # | Question | Recommendation |
|---|---|---|
| Q1 | Should `POST /projects` return 201 Created with a `Location` header (REST convention) or 200 like the existing creates? | **200 OK**, consistent with `TasksController.cs:29-35` and `CommentsController.cs:29-35`. |
| Q2 | Maximum project name length? | **100 characters** after trimming. |
| Q3 | Does an archived project still block re-using its name? | **Yes, 409.** Avoids two "Home" projects once unarchive is added later. |
| Q4 | What happens when the authenticated user has no local `User` row (Keycloak user never registered via `POST /users`)? | **403 Forbidden** with a problem-details message. Auto-provisioning on first request is a cross-cutting change for a separate PBI. |
| Q5 | Should archived projects still be readable via `GET /projects/{id}/tasks`? | **Yes, 200** with `isArchived: true`, so users can still find the tasks that were kept. Optionally add `GET /projects?includeArchived=true` later. |
| Q6 | Should the frontend filter also offer "No project" (tasks without a project)? | **Not in this PBI**; only "All projects" plus active projects. Can follow when task listing is made user-scoped. |
| Q7 | Should the existing endpoints that take `userId` from the request (tasks, comments) and the unauthenticated `UsersController` be fixed? | **Yes, as a separate security PBI**, so this PBI stays focused. The new endpoints must not rely on or extend those patterns. |
| Q8 | Should `GET /projects` include task counts per project? | **No**; counts are returned by 3.3. Keeps the list query cheap. |
| Q9 | Should `PUT /projects/{id}/archive` on an already archived project return 200 or 409? | **200, idempotent**, matching `TaskCompletion.feature:21-25`. |
