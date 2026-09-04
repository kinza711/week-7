# Week 7 — Assignment 2: REST API Contract

**CMIT Internship Program · Week 7**

## Overview

This deliverable is the REST API contract (`api-spec.md` + `openapi.yaml`) for the Task Management platform, over the same fixed 7-table domain used in Assignment 1's `DESIGN.md`. It specifies every route the Phase 4 NestJS backend implements: methods, paths, request/response bodies, status codes, pagination, and access control.

## What's in `api-spec.md`

| Section | Contents |
|---|---|
| 1. Pagination Convention | Offset convention for most lists; the cursor-pagination exception for `GET /projects/:id/tasks`, matching `DESIGN.md` Trade-off 1 |
| 2. Error Contract | One shape (`statusCode`, `message`, `error`, `timestamp`, `path`) with worked examples for 400/401/403/404 |
| 3. Auth Endpoints | `register`, `login`, `refresh`, `logout` — plus the token-contents note (identity-only JWT, no role claim) |
| 4–9. Resources | `users`, `projects`, `project members` (incl. the dedicated `transfer-ownership` endpoint), `tasks` (with `tagIds`), `tags`, `comments` — full CRUD, nested where the domain requires it |
| 10. Access Level Summary | Public / Authenticated / Role-restricted, applied per route |

## Challenge (extra marks)

- **X1** — Idempotent task creation via `Idempotency-Key` header
- **X2** — Full status-code table for `tasks`, with the `403` vs `404` rule spelled out
- **X3** — `openapi.yaml`: the same contract as a validated OpenAPI 3.1 spec

## Consistency with Assignment 1

This contract is kept in sync with `DESIGN.md`'s decisions as they evolved:
- Cursor pagination applies only to the tasks list, matching Trade-off 1
- The ownership invariant (`project_members.role='owner'` as the single authoritative field) is enforced at the API boundary via a dedicated `transfer-ownership` endpoint and restricted role-assignment values
- `tagIds` on task creation matches the updated `DESIGN.md` trace, which now validates and inserts `task_tags` in the same transaction
- The JWT identity-only model matches `DESIGN.md` X2

## Files

- `api-spec.md`
- `openapi.yaml`

## Author

Kinza Shahzadi — CMIT Full-Stack Internship, Coding Pixel