# Part 2 Test Output

**Test file:** [src/tests/part2.test.ts](src/tests/part2.test.ts)
**Environment:** Docker (`web` + `db`), Next.js dev server at `http://localhost:3000`, seed data from the README

**Command used for both runs:**

```bash
docker compose exec web npx vitest run src/tests/part2.test.ts --hookTimeout=60000 --testTimeout=30000
```

> **Why the longer timeouts:** with Vitest's default 10s `hookTimeout`, the `beforeAll` hook timed out before any test ran. It sends three logins at once, and on a freshly started container each one took about 10s while Next.js compiled the API routes (`POST /api/auth/login 200 in 10013ms` in the server log). Even after the routes are compiled, each API call takes about 1.5–3s in this Docker-on-Windows setup. Passing the timeouts on the command line keeps the test file's setup unchanged.

## Summary

| Test | Before fixes | After fixes |
|------|--------------|-------------|
| A: a viewer cannot update a task | ❌ expected 403, got **200** | ✅ 403 |
| B: a viewer cannot create a task | ❌ expected 401, got **403** | ✅ 403 |
| C: a member can create a task | ✅ 201 | ✅ 201 |
| D: a viewer can list tasks (read-only access) | ✅ 200 | ✅ 200 |
| **Total** | **2 failed, 2 passed** | **4 passed** |

## What was fixed

| Test | Diagnosis | Fix |
|------|-----------|-----|
| A | **App bug.** `PATCH /api/tasks/:id` never checked project membership or role, so any logged-in user could edit any task ([BUGS.md #1](BUGS.md#1-viewers-and-non-members-can-edit-tasks)). | Added the membership and `canEditTasks` checks to the update handler in [src/app/api/tasks/[id]/route.ts](src/app/api/tasks/[id]/route.ts). This matches what create and delete already did. A viewer now gets 403 `viewers cannot update tasks`; a non-member gets 403 `you are not a member of this project`. |
| B | **Test bug.** The test expected `401 Unauthorized`, but the viewer *is* logged in; they just aren't allowed to create tasks. The correct status is `403 Forbidden`, which the API already returned. | Changed the expected status in Test B from `401` to `403`. The app code was not changed for this test. |
| D | New test: confirms that a viewer still has read access after the fix. | No fix needed. |

---

## Run 1: Before fixes

This run used the original test file (Test B expecting `401`) plus the new Test D, with no app changes.

```text
 RUN  v2.1.8 /app

 ❯ src/tests/part2.test.ts (4 tests | 2 failed) 14140ms
   × task access control > a viewer cannot update a task 1585ms
     → expected 200 to be 403 // Object.is equality
   × task access control > a viewer cannot create a task 2476ms
     → expected 403 to be 401 // Object.is equality
   ✓ task access control > a member can create a task 2633ms
   ✓ task access control > a viewer can list tasks (read-only access) 1700ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 2 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/tests/part2.test.ts > task access control > a viewer cannot update a task
AssertionError: expected 200 to be 403 // Object.is equality

- Expected
+ Received

- 403
+ 200

 ❯ src/tests/part2.test.ts:57:24
     55|       body: JSON.stringify({ title: "viewer update attempt" }),
     56|     });
     57|     expect(res.status).toBe(403);
       |                        ^
     58|   });
     59| 

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/2]⎯

 FAIL  src/tests/part2.test.ts > task access control > a viewer cannot create a task
AssertionError: expected 403 to be 401 // Object.is equality

- Expected
+ Received

- 401
+ 403

 ❯ src/tests/part2.test.ts:70:24
     68|       body: JSON.stringify({ title: "viewer create attempt" }),
     69|     });
     70|     expect(res.status).toBe(401);
       |                        ^
     71|   });
     72| 

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[2/2]⎯

 Test Files  1 failed (1)
      Tests  2 failed | 2 passed (4)
   Start at  01:52:46
   Duration  16.01s (transform 267ms, setup 391ms, collect 102ms, tests 14.14s, environment 3ms, prepare 287ms)
```

## Run 2: After fixes

This run was made after the route fix and the change to Test B. The `web` container was restarted first, because the dev server running on a Windows bind mount does not pick up file changes automatically. The restart also reseeded the database.

```text
 RUN  v2.1.8 /app

 ✓ src/tests/part2.test.ts (4 tests) 27314ms
   ✓ task access control > a viewer cannot update a task 8022ms
   ✓ task access control > a viewer cannot create a task 2024ms
   ✓ task access control > a member can create a task 2858ms
   ✓ task access control > a viewer can list tasks (read-only access) 3013ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Start at  01:59:46
   Duration  30.90s (transform 483ms, setup 1.05s, collect 260ms, tests 27.31s, environment 1ms, prepare 405ms)
```
