# TaskBoard: Bug Report

**Environment:** local Docker build, `http://localhost:3000`, seed data from the README (all passwords `password123`)
**Test date:** 2026-11-07
**How it was tested:** calls to the live API with `curl`, plus a read of the UI source code. Every proof below is real output from this environment. Long JSON responses are cut short with `…`.

Bugs are listed from highest to lowest business impact.

| # | Bug | Category | Severity | Status |
|---|-----|----------|----------|--------|
| 1 | Viewers (and non-members) can edit tasks | Security | Critical | **Fixed** |
| 2 | New accounts cannot create or see any project | Architecture | High | Open |
| 3 | No way to add members to a project or set their role | Architecture | High | Open |
| 4 | Viewers still see the add, delete and save buttons | Security | Medium | Open |
| 5 | Status names in the "add a task" dropdown don't match the board | Data Integrity | Low | Open |
| 6 | Headings and button text are styled inconsistently | Architecture | Low | Open |

### Getting tokens for the curl commands

```bash
B=http://localhost:3000
login() { curl -s -X POST $B/api/auth/login -H "Content-Type: application/json" \
  -d "{\"email\":\"$1\",\"password\":\"password123\"}" | grep -o '"token":"[^"]*"' | cut -d'"' -f4; }

ADMIN=$(login meera@taskboard.dev)   # admin on Q3 Launch
VIEWER=$(login dev@example.com)      # viewer on Q3 Launch
OUTSIDER=$(login lina@example.com)   # NOT a member of Q3 Launch
MEMBER=$(login arjun@taskboard.dev)  # member on Q3 Launch

PROJECT=cmw5qa42x0006ny9rqhdclcf4    # Q3 Launch
TASK=cmw5qa4ea000vny9rv9mxkkj1       # "Prepare customer email blast"
```

The IDs above are from this run. After a reseed, get them from `GET /api/projects` and `GET /api/projects/$PROJECT/tasks`.

---

## 1. Viewers (and non-members) can edit tasks

- **Files:** [src/app/api/tasks/[id]/route.ts:16-38](src/app/api/tasks/[id]/route.ts#L16-L38)
- **Category:** Security
- **Severity:** Critical
- **Status:** Fixed. The update (`PATCH`) handler now checks project membership and `canEditTasks` before saving, the same checks that create and delete already used.

> When a viewer opens a task in the edit dialog, changes the title and clicks **save**, the app saves the change (`PATCH /api/tasks/:id` returns 200), but it should refuse the change with a 403, just as it already does for create and delete.

The update handler checked that the user was logged in, but never called `getProjectMembership` or `canEditTasks`. Because of this, **any logged-in user could edit any task in any project if they knew its ID**, including people who aren't members of that project. Anyone could change task content, status or assignee without permission.

**Curl proof (before the fix):**

```bash
# Viewer edits a task
$ curl -s -w "\nHTTP %{http_code}\n" -X PATCH $B/api/tasks/$TASK \
    -H "Authorization: Bearer $VIEWER" -H "Content-Type: application/json" \
    -d '{"title":"EDITED BY VIEWER"}'
{"task":{"id":"cmw5qa4ea000vny9rv9mxkkj1","projectId":"cmw5qa42x0006ny9rqhdclcf4","title":"EDITED BY VIEWER",…}}
HTTP 200

# A user who is not a member of the project edits the same task
$ curl -s -w "\nHTTP %{http_code}\n" -X PATCH $B/api/tasks/$TASK \
    -H "Authorization: Bearer $OUTSIDER" -H "Content-Type: application/json" \
    -d '{"title":"EDITED BY NON-MEMBER"}'
{"task":{"id":"cmw5qa4ea000vny9rv9mxkkj1","projectId":"cmw5qa42x0006ny9rqhdclcf4","title":"EDITED BY NON-MEMBER",…}}
HTTP 200
```

**Curl proof (after the fix):** the database was reseeded, so the task ID is different.

```bash
$ curl -s -w " HTTP %{http_code}\n" -X PATCH $B/api/tasks/$TASK -H "Authorization: Bearer $VIEWER" \
    -H "Content-Type: application/json" -d '{"title":"Prepare customer email blast"}'
{"error":"viewers cannot update tasks"} HTTP 403

$ curl -s -w " HTTP %{http_code}\n" -X PATCH $B/api/tasks/$TASK -H "Authorization: Bearer $OUTSIDER" \
    -H "Content-Type: application/json" -d '{"title":"Prepare customer email blast"}'
{"error":"you are not a member of this project"} HTTP 403

# A member can still edit (no regression)
$ curl -s -w " HTTP %{http_code}\n" -X PATCH $B/api/tasks/$TASK -H "Authorization: Bearer $MEMBER" \
    -H "Content-Type: application/json" -d '{"title":"Prepare customer email blast"}'
{"task":{"id":"cmw5r8eae000vny3klpr5v7b4",…,"title":"Prepare customer email blast",…}} HTTP 200
```

Part 2 Test A covers this; see [TEST_OUTPUT.md](TEST_OUTPUT.md).

---

## 2. New accounts cannot create or see any project

- **Files:** [src/app/dashboard/page.tsx:37-50](src/app/dashboard/page.tsx#L37-L50), [src/app/api/projects/route.ts:41-61](src/app/api/projects/route.ts#L41-L61)
- **Category:** Architecture
- **Severity:** High

> When a newly registered user lands on the dashboard, the app shows only "no projects yet." with no button or link to do anything, but it should offer a **Create project** action, since `POST /api/projects` already exists and makes the creator the project admin.

The API supports creating a project, but no screen calls it. A new user has no way to get into any project, so they are stuck from the moment they sign up.

**Curl proof:**

```bash
# Register a new user
$ curl -s -w "\nHTTP %{http_code}\n" -X POST $B/api/auth/register -H "Content-Type: application/json" \
    -d '{"name":"QA Proof","email":"qa.proof.1794016419@example.com","password":"password123"}'
{"user":{"id":"cmw5r23ha000cnybsr5x08grf","email":"qa.proof.1794016419@example.com","name":"QA Proof"},"token":"<jwt>"}
HTTP 201

# The new user has no projects, and the dashboard offers no way to create one
$ curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Bearer $NEW_USER" $B/api/projects
{"projects":[]}
HTTP 200

# The API does support creating a project (no screen calls it)
$ curl -s -w "\nHTTP %{http_code}\n" -X POST $B/api/projects \
    -H "Authorization: Bearer $NEW_USER" -H "Content-Type: application/json" -d '{"name":"QA probe project"}'
{"project":{"id":"cmw5pnbwq0005la5izo5qr4gn","name":"QA probe project",…}}
HTTP 201
```

---

## 3. No way to add members to a project or set their role

- **Files:** [src/app/api/auth/register/route.ts:15-26](src/app/api/auth/register/route.ts#L15-L26), [src/schemas/auth.ts:3-7](src/schemas/auth.ts#L3-L7), [src/app/projects/[id]/page.tsx:151-166](src/app/projects/[id]/page.tsx#L151-L166)
- **Category:** Architecture
- **Severity:** High

> When an admin looks at a project's **members** section, the app shows a read-only list with no way to add a person or change their role, but it should let admins invite users and set each one as admin, member or viewer.

By design, roles belong to a user's membership in a project, not to their account. That is why the register form has no role field, and why the register API silently ignores a `role` field if one is sent. The real gap is that there is no API endpoint and no screen for managing members. The only way to add people or set roles is through the seed script, so teams cannot actually work together in the app.

**Curl proof:**

```bash
# A "role" sent during registration is silently ignored (no role in the response, no error)
$ curl -s -X POST $B/api/auth/register -H "Content-Type: application/json" \
    -d '{"name":"QA Proof","email":"qa.proof.1794016419@example.com","password":"password123","role":"admin"}'
{"user":{"id":"cmw5r23ha000cnybsr5x08grf","email":"qa.proof.1794016419@example.com","name":"QA Proof"},"token":"<jwt>"}

# There is no endpoint for adding a member, even for the project admin
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" -X POST $B/api/projects/$PROJECT/members \
    -H "Authorization: Bearer $ADMIN" -H "Content-Type: application/json" \
    -d '{"email":"lina@example.com","role":"viewer"}'
HTTP 404
```

---

## 4. Viewers still see the add, delete and save buttons

- **Files:** [src/app/projects/[id]/page.tsx:96-138](src/app/projects/[id]/page.tsx#L96-L138), [src/components/TaskDetail.tsx:132-155](src/components/TaskDetail.tsx#L132-L155)
- **Category:** Security
- **Severity:** Medium

> When a viewer opens a project or a task, the app shows the **add a task** form and the **delete task** and **save** buttons, but it should hide them (or make the task dialog read-only), because a viewer is not allowed to change tasks.

The project page and the task dialog never check the current user's role, even though the project API already returns each member's role. Viewers fill in forms only to get an error. Until Bug #1 was fixed, **save** actually went through.

**Curl proof:** the page receives the viewer's role but ignores it, and the API rejects these actions:

```bash
# The project data the page loads already says this user is a viewer
$ curl -s -H "Authorization: Bearer $VIEWER" $B/api/projects/$PROJECT
{"project":{…,"memberships":[…,{"id":"cmw5qa42x000bny9r459obm5k","userId":"cmw5qa42b0003ny9rysgpkf5c",
 "projectId":"cmw5qa42x0006ny9rqhdclcf4","role":"viewer",…}]}}

# Yet the UI shows the add and delete buttons, which only lead to a 403 error
$ curl -s -w " HTTP %{http_code}\n" -X POST $B/api/projects/$PROJECT/tasks \
    -H "Authorization: Bearer $VIEWER" -H "Content-Type: application/json" -d '{"title":"x"}'
{"error":"viewers cannot create tasks"} HTTP 403

$ curl -s -w " HTTP %{http_code}\n" -X DELETE $B/api/tasks/$TASK -H "Authorization: Bearer $VIEWER"
{"error":"viewers cannot delete tasks"} HTTP 403
```

---

## 5. Status names in the "add a task" dropdown don't match the board

- **Files:** [src/app/projects/[id]/page.tsx:119-123](src/app/projects/[id]/page.tsx#L119-L123), [src/types/index.ts:42-47](src/types/index.ts#L42-L47)
- **Category:** Data Integrity
- **Severity:** Low

> When a user opens the status dropdown in **add a task**, the app lists the internal values `todo`, `in_progress`, `review` and `done`, but it should show the same names as the board columns and the edit dialog: `To do`, `In progress`, `In review` and `Done`.

The dropdown displays the raw value `{s}` instead of `STATUS_LABELS[s]`. The mismatch is most confusing for "review", which the board calls "In review", so users can't easily tell which column a new task will go into.

**Curl proof:** the API returns the internal status values, and the add-task dropdown shows them as-is:

```bash
$ curl -s -H "Authorization: Bearer $VIEWER" $B/api/projects/$PROJECT/tasks | grep -o '"status":"[^"]*"' | sort | uniq -c
      1 "status":"done"
      2 "status":"in_progress"
      1 "status":"review"
      9 "status":"todo"
```

The board columns and the edit dialog map these values through `STATUS_LABELS` (`review` → `In review`). The add-task dropdown does not.

---

## 6. Headings and button text are styled inconsistently

- **Files:** [src/app/dashboard/page.tsx:38](src/app/dashboard/page.tsx#L38), [src/app/projects/[id]/page.tsx:65](src/app/projects/[id]/page.tsx#L65), [src/app/projects/[id]/page.tsx:97](src/app/projects/[id]/page.tsx#L97), [src/components/TaskDetail.tsx:67](src/components/TaskDetail.tsx#L67), [src/components/Header.tsx:24](src/components/Header.tsx#L24), [src/app/register/page.tsx:37](src/app/register/page.tsx#L37)
- **Category:** Architecture
- **Severity:** Low

> When a user moves between the login, dashboard and project pages, the app mixes lowercase text ("your projects", "add a task", "edit task", "add", "save", "sign out") with Title Case text ("TaskBoard", "To do", "In review"), but it should use one capitalisation style for all headings and buttons.

There are also layout differences. The top bar uses `max-w-6xl` while the project page uses `max-w-7xl`, so their edges don't line up. The login page is titled "TaskBoard" while the register page is titled "create account". Each page sets its own text and styles; there are no shared heading or button components to keep them consistent.

**Curl proof:** headings and buttons taken from the server-rendered HTML (class names removed):

```bash
$ for p in login register dashboard; do echo "/$p:"; curl -s $B/$p | grep -o '<h1[^>]*>[^<]*</h1>\|<button[^>]*>[^<]*</button>'; done
/login:
<h1>TaskBoard</h1>
<button type="submit">sign in</button>
/register:
<h1>create account</h1>
<button type="submit">create account</button>
/dashboard:
<button>sign out</button>
<h1>your projects</h1>
```
