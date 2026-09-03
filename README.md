# Week 7 — Assignment 1: System Design Document

**CMIT Internship Program · Phase 3 · Week 7**

## Overview

This deliverable is a system design document (`DESIGN.md`) for the Task Management platform — the fixed 7-table domain built in Week 5 (SQL) and Week 6 (TypeORM entities). It specifies the system as if handing it off to a new team: requirements, data model, architecture, non-functional plan, and the trade-offs behind each decision.

This document is what Phase 4 (NestJS backend, starting next week) will be implemented against.

## What's in `DESIGN.md`

| Section | Contents |
|---|---|
| 1. Overview & Requirements | System summary, 3 functional requirements, 3 non-functional requirements |
| 2. ERD | Mermaid `erDiagram` of all 7 tables with labelled cardinalities |
| 3. Architecture | Layered design (Controller → Service → Repository → Database) and a full trace of a "create task" request, including status-code mapping |
| 4. Non-Functional Plan | Caching strategy (what's cached, what invalidates it), read-replica scaling, and the consistency model |
| 5. Trade-offs | 4 documented decisions, each with the rejected alternative and the criterion that decided it |


## Domain

Fixed schema, unchanged from Week 5/6: `users`, `projects`, `project_members`, `tasks`, `tags`, `task_tags`, `comments`.



## Author

Kinza Shahzadi — CMIT Full-Stack Internship, Coding Pixel