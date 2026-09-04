# Task Management Platform — REST API Contract

Base URL prefix for every route: **`/api/v1`**

This contract is what the Phase 4 NestJS backend implements. Entity and column names match `DESIGN.md` exactly.

---

## 1. Pagination, Filtering & Sorting Convention

Applies to every list (`GET` collection) endpoint. Defined once here; individual endpoints reference it instead of redefining it.

**Default: offset pagination.** Used by every list endpoint except the tasks list (see exception below).

| Param | Type | Default | Notes |
|---|---|---|---|
| `page` | int | `1` | 1-indexed |
| `pageSize` | int | `20` | max `100` — requests above the max are clamped, not rejected |
| `status` | string | — | endpoint-specific filter (e.g. task status) |
| `sort` | string | `-createdAt` | prefix `-` for descending, e.g. `sort=-createdAt`, `sort=priority` |

**Offset paged response shape:**

```json
{
  "data": [ /* array of resources */ ],
  "total": 132,
  "page": 1,
  "pageSize": 20
}
```

**Exception: cursor pagination for tasks.** `GET /api/v1/projects/:id/tasks` uses cursor pagination instead (see `DESIGN.md`, Trade-off 1 — task lists are inserted into frequently, and offset pagination shifts/duplicates rows under concurrent inserts).

| Param | Type | Default | Notes |
|---|---|---|---|
| `cursor` | string | — | opaque, base64-encoded; omit for the first page |
| `pageSize` | int | `20` | max `100`, same as above |
| `status` | string | — | same filter semantics as above |
| `sort` | string | `-createdAt` | same semantics as above |

**Cursor paged response shape:**

```json
{
  "data": [ /* array of tasks */ ],
  "nextCursor": "eyJpZCI6NDUsImNyZWF0ZWRBdCI6Ii4uLiJ9",
  "pageSize": 20
}
```

`nextCursor` is `null` when there is no further page. All other list endpoints (`projects`, `project members`, `tags`, `comments`) use the offset convention above.

---

## 2. Error Contract

Every error response, from every endpoint, uses this shape:

```json
{
  "statusCode": 404,
  "message": "Task not found",
  "error": "Not Found",
  "timestamp": "2026-09-03T10:15:00.000Z",
  "path": "/api/v1/tasks/45"
}
```

For validation failures (`400`), `message` is an **array of field errors** instead of a single string:

```json
{
  "statusCode": 400,
  "message": ["title should not be empty", "priority must be between 1 and 5"],
  "error": "Bad Request",
  "timestamp": "2026-09-03T10:15:00.000Z",
  "path": "/api/v1/projects/3/tasks"
}
```

**Worked examples:**

| Code | Example `message` | `error` |
|---|---|---|
| 400 | `["title should not be empty"]` | `Bad Request` |
| 401 | `"Invalid credentials"` | `Unauthorized` |
| 403 | `"You do not have permission to perform this action"` | `Forbidden` |
| 404 | `"Project not found"` | `Not Found` |

---

## 3. Auth Endpoints

| Method | Path | Access | Body → Response |
|---|---|---|---|
| POST | `/api/v1/auth/register` | Public | `{ name, email, password }` → `201` `{ id, name, email, createdAt }` |
| POST | `/api/v1/auth/login` | Public | `{ email, password }` → `200` `{ accessToken, refreshToken }` |
| POST | `/api/v1/auth/refresh` | Public (requires valid refresh token) | `{ refreshToken }` → `200` `{ accessToken, refreshToken }` |
| POST | `/api/v1/auth/logout` | Authenticated | `{ refreshToken }` → `204 No Content` |

**Rules:** `login` with a wrong password → `401`, never `400`/`404` (never confirm whether the email exists). `refresh` issues a brand-new token pair and revokes the old refresh token. `logout` revokes the given refresh token. **No response body anywhere returns `password` or `password_hash`.**

**Token contents.** The access token carries **identity only** (`userId`, `email`) — no role or project-membership claim, per `DESIGN.md` X2. Every protected endpoint re-checks the caller's role against `project_members` live on each request; nothing in this API trusts a role embedded in the token.

---

## 4. Users

| Method | Path | Access | Notes |
|---|---|---|---|
| GET | `/api/v1/users/me` | Authenticated | Returns the caller's own profile |
| PATCH | `/api/v1/users/me` | Authenticated | Update `name` only |
| GET | `/api/v1/users/:id` | Authenticated | Public profile fields only (no email) |

---

## 5. Projects

| Method | Path | Access | Body / Notes |
|---|---|---|---|
| POST | `/api/v1/projects` | Authenticated | `{ name }` → `201`, caller becomes `owner` |
| GET | `/api/v1/projects` | Authenticated | Lists projects the caller is a member of. Follows the list convention. |
| GET | `/api/v1/projects/:id` | Authenticated (member only) | `403` if not a member, `404` if project doesn't exist |
| PATCH | `/api/v1/projects/:id` | `owner`, `admin` | `{ name }` |
| DELETE | `/api/v1/projects/:id` | `owner` only | `204` |

---

## 6. Project Members

Nested under a project — a member is not a standalone resource.

| Method | Path | Access | Body / Notes |
|---|---|---|---|
| POST | `/api/v1/projects/:id/members` | `owner`, `admin` | `{ userId, role: "admin" \| "member" \| "viewer" }` → `201` — `owner` is **not** an assignable value here; rejected with `400` if sent |
| GET | `/api/v1/projects/:id/members` | Any member | Follows the list convention |
| PATCH | `/api/v1/projects/:id/members/:userId` | `owner`, `admin` | `{ role: "admin" \| "member" \| "viewer" }` — same restriction; changing to `owner` is always `400` here, regardless of caller's role |
| DELETE | `/api/v1/projects/:id/members/:userId` | `owner`, `admin` | `204` — rejected with `409 Conflict` if the target is the current `owner` (a project must always retain exactly one) |
| POST | `/api/v1/projects/:id/transfer-ownership` | `owner` only | `{ newOwnerUserId }` → `200`. Atomically updates `projects.owner_id` and swaps the `project_members.role='owner'` row to the new user, per `DESIGN.md`'s ownership invariant (row-locked, single transaction). This is the **only** endpoint that can produce a second `owner` role change — every other member-role write is rejected before it can touch ownership. |

---

## 7. Tasks

| Method | Path | Access | Body / Notes |
|---|---|---|---|
| POST | `/api/v1/projects/:id/tasks` | Any member except `viewer` | `{ title, description?, priority, assigneeId?, tagIds?: number[] }` → `201`, task returned with its attached tags |
| GET | `/api/v1/projects/:id/tasks` | Any member | Uses **cursor pagination** (Section 1 exception), not the default offset convention; `status` filters by task status |
| GET | `/api/v1/tasks/:id` | Any member of the parent project | `404` if not found or caller not a member |
| PATCH | `/api/v1/tasks/:id` | Any member except `viewer` | Partial update — `{ title?, description?, status?, priority?, assigneeId? }` |
| DELETE | `/api/v1/tasks/:id` | `owner`, `admin`, or the task's original creator | `204` |

**Status codes:** `201` create, `200` read/update, `204` delete, `400` invalid body **or any `tagIds` entry that doesn't exist in `tags`**, `403` insufficient role or not a member, `404` task or project not found.

---

## 8. Tags

| Method | Path | Access | Body / Notes |
|---|---|---|---|
| POST | `/api/v1/tags` | Authenticated | `{ name }` → `201`, `409` if name already exists |
| GET | `/api/v1/tags` | Authenticated | Follows the list convention |
| POST | `/api/v1/tasks/:id/tags` | Any member except `viewer` | `{ tagId }` → `201`, attaches tag to task |
| DELETE | `/api/v1/tasks/:id/tags/:tagId` | Any member except `viewer` | `204` |

---

## 9. Comments

Nested under a task — a comment is not a standalone resource.

| Method | Path | Access | Body / Notes |
|---|---|---|---|
| POST | `/api/v1/tasks/:id/comments` | Any member | `{ body }` → `201`, `authorId` set from the token, not the body |
| GET | `/api/v1/tasks/:id/comments` | Any member | Follows the list convention |
| PATCH | `/api/v1/comments/:id` | Comment author only | `{ body }` |
| DELETE | `/api/v1/comments/:id` | Comment author, or project `owner`/`admin` | `204` |

---

## 10. Access Level Summary

| Access level | Meaning |
|---|---|
| Public | No token required |
| Authenticated | Any valid access token |
| Role-restricted | Authenticated **and** holds the named `project_members.role` for that project |

No read that exposes another user's or another project's data is marked Public. Only `auth/register` and `auth/login` are Public.

---

## Challenge (Optional — Extra Marks)

### X1. Idempotent Create — Task Creation

`POST /api/v1/projects/:id/tasks` accepts an optional `Idempotency-Key` request header (a client-generated UUID).

- On first request with a given key: the task is created normally, and the response body is stored alongside the key.
- On a retry with the **same** key (e.g. after a network timeout): the server returns the **original** `201` response and creates **no second row**.
- Keys are stored per-user for **24 hours**, then expire and may be reused.
- A request with the same key but a **different** body is rejected with `422 Unprocessable Entity` (key reuse mismatch).

### X2. Full Status-Code Table — Tasks

| Code | Cause |
|---|---|
| 200 | Successful `GET` or `PATCH` |
| 201 | Task created |
| 204 | Task deleted |
| 400 | Invalid body (e.g. `priority` out of 1–5 range) |
| 401 | Missing or expired access token |
| 403 | Caller is a project member but lacks permission (e.g. a `viewer` trying to create) |
| 404 | Task or parent project doesn't exist, **or** caller is not a member of the project |
| 409 | Conflict — e.g. attaching a tag that's already attached to the task |
| 500 | Unhandled server error |

**403 vs 404:** if a caller is a **member** of the project but lacks the role to perform the action, return `403` — the resource's existence is not sensitive to them. If the caller is **not a member** of the project at all, return `404` instead of `403` for that project's tasks — confirming the project/task exists via a `403` would leak its existence to someone with no relationship to it. `404` leaks less and is used whenever membership itself is the gate.

### X3. OpenAPI Specification

See `openapi.yaml` (companion file) — describes the same routes, the pagination parameters, and the error contract as reusable `components`, validated against the OpenAPI 3.1 schema.