# Part 2 Test Output

**Test file:** [src/tests/part2.test.ts](src/tests/part2.test.ts)
**Environment:** Docker (`web` + `db`), Next.js dev server at `http://localhost:3000`, seed data from the README

**Command used for both runs:**

```bash
docker compose exec web npx vitest run src/tests/part2.test.ts --hookTimeout=60000 --testTimeout=30000
```

> **Why the longer timeouts:** with Vitest's default 10s `hookTimeout`, the `beforeAll` hook timed out before any test ran. It sends three logins at once, and on a freshly started container each one took about 10s while Next.js compiled the API routes (`POST /api/auth/login 200 in 10013ms` in the server log). Even after the routes are compiled, each API call takes about 1.5–3s in this Docker-on-Windows setup. Passing the timeouts on the command line keeps the test file's setup unchanged.

## Summary

| Test | Run 1: original tests | Run 2: after Test B fix |
|------|-----------------------|-------------------------|
| A: a viewer cannot update a task | ❌ expected 403, got **200** | ❌ expected 403, got **200** (app bug, still open) |
| B: a viewer cannot create a task | ❌ expected 401, got **403** | ✅ 403 |
| C: a member can create a task | ✅ 201 | ✅ 201 |
| D: a viewer can list tasks (read-only access) | ✅ 200 | ✅ 200 |
| **Total** | **2 failed, 2 passed** | **1 failed, 3 passed** |

## Diagnosis

| Test | Diagnosis | Action |
|------|-----------|--------|
| A | **App bug.** `PATCH /api/tasks/:id` never checks project membership or role, so any logged-in user can edit any task ([BUGS.md #1](BUGS.md#1-viewers-and-non-members-can-edit-tasks)). The test is correct: a viewer should get `403 Forbidden`. | **Not fixed; the test fails on purpose.** The assertion is unchanged, because it correctly catches the bug. The test will pass once the update handler adds the membership and `canEditTasks` checks that create and delete already use. |
| B | **Test bug.** The test expected `401 Unauthorized`, but the viewer *is* logged in; they just aren't allowed to create tasks. The correct status is `403 Forbidden`, which the API already returned. | Changed the expected status in Test B from `401` to `403`. The app code was not changed for this test. |
| D | New test: confirms that a viewer has read access to the project's tasks. | No fix needed. |

---

## Run 1: Original tests

This run used the original test file (Test B expecting `401`) plus the new Test D. The app code is unchanged in both runs.

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

## Run 2: After Test B fix

This run was made after changing the Test B assertion from `401` to `403`. The app code is unchanged, so Test A still fails because of the app bug. The `web` container was restarted first, because the dev server running on a Windows bind mount does not pick up file changes automatically. The restart also reseeded the database.

```text
 RUN  v2.1.8 /app

 ❯ src/tests/part2.test.ts (4 tests | 1 failed) 29967ms
   × task access control > a viewer cannot update a task 6421ms
     → expected 200 to be 403 // Object.is equality
   ✓ task access control > a viewer cannot create a task 1751ms
   ✓ task access control > a member can create a task 1605ms
   ✓ task access control > a viewer can list tasks (read-only access) 1842ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯

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

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯

 Test Files  1 failed (1)
      Tests  1 failed | 3 passed (4)
   Start at  02:46:13
   Duration  34.98s (transform 405ms, setup 746ms, collect 205ms, tests 29.97s, environment 1ms, prepare 652ms)
```
