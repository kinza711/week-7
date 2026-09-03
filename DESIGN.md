# Task Management Platform — System Design Document

## 1. Overview and Requirements

### Overview
The Task Management platform is a collaborative work-tracking system where teams organize their work into projects. Users can create projects, invite other users as members with specific roles (owner, admin, member, viewer), and manage tasks within those projects. Tasks can be assigned to a single user, tagged for categorization, and discussed through threaded comments. Access to project data is controlled by each user's role, so the system supports both open collaboration and controlled visibility.

### Functional Requirements
1. Users can create, update, and delete projects, with each project belonging to exactly one owner.
2. Project owners and admins can invite users as project members and assign them a role (owner, admin, member, or viewer).
3. Members can create tasks within a project, assign a task to one user, tag it with one or more tags, and comment on it.

### Non-Functional Requirements
1. **Performance** — Read endpoints must respond within 200ms and write endpoints within 500ms under normal load.
2. **Security** — All authenticated endpoints must reject requests without a valid access token, and passwords must be stored using a one-way hash (bcrypt), never plaintext.
3. **Availability** — The system should target 99.5% uptime, with read endpoints remaining available even during a brief write-path outage.

---

## 2. Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ PROJECTS : owns
    USERS ||--o{ PROJECT_MEMBERS : "is member"
    PROJECTS ||--o{ PROJECT_MEMBERS : has
    PROJECTS ||--o{ TASKS : contains
    USERS |o--o{ TASKS : "assigned to (optional)"
    TASKS ||--o{ TASK_TAGS : has
    TAGS ||--o{ TASK_TAGS : has
    TASKS ||--o{ COMMENTS : has
    USERS ||--o{ COMMENTS : writes

    USERS {
        int id PK
        string name
        string email UK
        datetime created_at
    }
    PROJECTS {
        int id PK
        string name
        int owner_id FK
        datetime created_at
    }
    PROJECT_MEMBERS {
        int user_id PK_FK
        int project_id PK_FK
        string role "owner/admin/member/viewer"
    }
    TASKS {
        int id PK
        string title
        string description
        string status "todo/in_progress/done"
        int priority "1-5"
        int project_id FK
        int assignee_id FK "nullable"
        date due_date
        datetime created_at
    }
    TAGS {
        int id PK
        string name UK
    }
    TASK_TAGS {
        int task_id PK_FK
        int tag_id PK_FK
    }
    COMMENTS {
        int id PK
        int task_id FK
        int author_id FK
        string body
        datetime created_at
    }
```

All 7 tables from the fixed domain are represented. `task_tags` is the many-to-many join between tasks and tags. `project_members` uses a composite primary key on `(user_id, project_id)`. `tasks.assignee_id` is optional (nullable), shown with the `|o` optional notation.

---