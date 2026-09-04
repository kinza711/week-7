# Week 7 — Assignment 3: Architecture Decision Record

**CMIT Internship Program · Week 7**

## Overview

Two Architecture Decision Records documenting real decisions made across `DESIGN.md` and `api-spec.md`, following the Context / Options Considered / Decision / Consequences format.

## What's included

### `ADR-001.md` — Cursor Pagination for the Task List Endpoint
Documents why the task list uses cursor pagination while every other list endpoint in the API uses offset pagination — the one place in the domain where high read frequency and high write frequency coincide. Adds the cursor payload's `{sortValue, id}` tie-breaker, a detail not previously written down in either `DESIGN.md` or `api-spec.md`.

### `ADR-002.md` — Two-Tier Authorization (Challenge)
Documents a different decision, in a different layer: why authorization uses both a route-level `RolesGuard` (fast pre-check) and an authoritative Service-layer transactional re-check, rather than relying on the guard alone — a decision forced by NestJS's actual request lifecycle (guards run before interceptors), not by style preference.

## Consistency with Assignments 1 and 2

Both records were cross-checked against `DESIGN.md` and `api-spec.md` line by line:
- No entity, endpoint, parameter, or role name in either ADR contradicts the other two documents.
- ADR-001's cursor decision matches `DESIGN.md` Trade-off 1 and `api-spec.md` Section 1 exactly, including parameter names.
- ADR-002's guard/service split matches `DESIGN.md`'s Trade-off 2 and the Concurrency note in Section 3 — and documents *why* an earlier draft's claim (that a guard alone could be atomic with a later transaction) was incorrect, since NestJS runs guards before interceptors.

## Files

- `ADR-001.md`
- `ADR-002.md`
- `README.md`

## Author

Kinza Shahzadi — CMIT Full-Stack Internship, Coding Pixel