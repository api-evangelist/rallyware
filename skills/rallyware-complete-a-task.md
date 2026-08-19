---
name: Drive a Rallyware task from assignment to completion
description: >-
  Read a field user's task feed, start an assigned task, save and submit its unit
  results, and complete it — the core Rallyware enablement loop, and the only
  write surface Rallyware publishes.
api: mcp/rallyware-mcp.yml
generated: '2026-08-14'
method: generated
source: >-
  npm @rallyware/sdk-react-native-components@1.2.1 —
  lib/module/services/task-service.js, task-program-service.js,
  lib/typescript/dto/*.d.ts, lib/typescript/enums/*.d.ts
operations:
  - listMyTaskPrograms
  - getTaskProgram
  - listAvailableTasks
  - getUserTask
  - startUserTask
  - saveUnitResultDraft
  - submitUnitResult
  - completeUserTask
---

# Drive a Rallyware task from assignment to completion

This is Rallyware's product in one loop: a field seller or distributor is assigned
work, does it, submits evidence, and is scored. Every operation below is read from
Rallyware's own published SDK — see `mcp/rallyware-mcp.yml` for the path and method of
each. Authenticate first using the companion skill,
`rallyware-authenticate-and-page.md`.

## The two-entity model you must not confuse

- **`Task`** is the *template* — what may be done. It lives in the tenant catalogue.
- **`UserTask`** is the *assignment* — this task, for this user, with `status`,
  `due_date`, `points_awarded` and history.

**You only ever act on a `UserTask`.** There is no operation that writes to a `Task`.
Getting this backwards is the most likely way to fail against this API.

## Step 1 — read the feed

`listAvailableTasks` → `GET /api/me/tasks_models`

This returns a **polymorphic collection**. Each member is a discriminated union:

```json
{ "type": "Task",     "item": { ...TaskDto... } }
{ "type": "UserTask", "item": { ...UserTaskDto... } }
```

Branch on `type` before touching `item`. Catalogue tasks and assignments arrive
interleaved in the same Hydra collection — page it as described in the companion
skill.

Optional narrowing (parameter names are **not** published; Rallyware's component layer
exposes these and forwards them opaquely, so treat them as low-confidence):
`taskProgramId`, `status` (a `UserTaskStatusesEnum` array), `featured`,
`items_per_page`.

For program context, use `listMyTaskPrograms` → `GET /api/users/me/task_programs` and
`getTaskProgram` → `GET /api/task_programs/{id}`. Progress is **pre-computed on the
program** — `task_count`, `completed_task_count`, `completed_user_task_count`,
`not_failed_user_task_count`. Read those counters; do not recompute them client-side.

## Step 2 — read the assignment

`getUserTask` → `GET /api/user_tasks/{userTaskId}`

Returns the `UserTask` with its `unit_configs` (the steps to complete) and any existing
`unit_results` (work already saved). Check `status` first:

| `status` | What to do |
|---|---|
| `available` | proceed to Step 3 |
| `in_progress` | skip Step 3, resume at Step 4 |
| `completed` | stop — nothing to do |
| `failed` | stop and read `fail_reason` |
| `locked` | **never appears from the API.** It is a synthetic client-side status, annotated as such in Rallyware's own source. If you are computing it, do not send it back. |

When `status` is `failed`, `fail_reason` is one of `business_rule` (a tenant rule
blocked it), `expired` (passed `due_date`), `rejected` (a reviewer rejected the
submission), or `""` (no failure). None of these is retryable by the API — all require
tenant-side action.

## Step 3 — start it

`startUserTask` → `PUT /api/user_tasks/{userTaskId}/start`

Moves the assignment to `in_progress` and returns the updated `UserTask`.

## Step 4 — save drafts, then submit each unit

A task is made of units. Each unit result is posted against **two IRI references**, not
ids — this API is JSON-LD, so relationships travel as paths:

```json
{
  "user_task":   "/api/user_tasks/{userTaskId}",
  "unit_config": "/api/unit_configs/{unitConfigId}",
  "data":        { ...unit result payload... }
}
```

Create vs. update is expressed by **verb and path**, which is the part most easily got
wrong:

| Intent | Call |
|---|---|
| First draft | `POST /api/unit_results/save_draft` |
| Update a draft | `PUT /api/unit_results/{unitResultId}/save_draft` |
| First submit | `POST /api/unit_results/submit` |
| Resubmit | `PUT /api/unit_results/{unitResultId}/submit` |

The `data` payload is shaped by the unit's config. Field types come from
`UnitReportItemTypesEnum`: `text`, `checkbox`, `editor`, `email`, `phone_number`,
`date`, `number`, `positive_number`, `url`, `vertical_radiobutton_list`,
`checkbox_list`, `file`, `image`, `video`, `slider`, `link`.

After submission the unit enters review. `UnitResultStatusesEnum` runs
`new → draft → in_review → approved | rejected`. A `rejected` result must be
resubmitted with `PUT .../submit`; it does not fail the task by itself.

## Step 5 — complete it

`completeUserTask` → `PUT /api/user_tasks/{userTaskId}/complete`

Returns the updated `UserTask` with `points_awarded` set. Completion may unlock a
`Badge` — re-read `listBadges` (`GET /api/badges`) and compare `is_unlocked` /
`unlocked_at` if you need to surface that.

## Retry safety — read this before you retry anything

**Rallyware supports no idempotency.** There is no `Idempotency-Key` header, no dedupe
parameter, and no retry-safety contract on any write. `startUserTask`,
`submitUnitResult` and `completeUserTask` are all unguarded.

Consequences you must design around:

- **Never blind-retry a write on timeout or 5xx.** Re-read the `UserTask` with
  `getUserTask` and branch on its `status` to decide whether the write landed.
- The one automatic retry that does exist — the `401` refresh-and-replay — replays the
  original request **verbatim**, including a `POST /api/unit_results/submit`. Refresh
  your token proactively before a write rather than letting a write trigger the
  refresh path.
- Prefer `save_draft` before `submit`. A duplicated draft is recoverable; a duplicated
  submission enters a human review queue.

## Consent is not yours to give

`agreeRules` → `POST /api/users/me/rules_agree` records the user's acceptance of the
tenant's terms and conditions. It is a legal act. Never call it autonomously — surface
the terms (resolved from `getPublicConfig`, key
`core.security.static_page.terms_of_service`) and require an explicit human action.
