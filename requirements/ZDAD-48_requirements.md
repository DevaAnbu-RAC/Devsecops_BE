# Requirement: Implement API to Fetch and Update Project DevSecOps Applicability and Completion Status

## Project: DevSecOps
## Story Name: Implement API to Fetch and Update Project DevSecOps Applicability and Completion Status
## Story Id: ZDAD-48

---

## 1. Overview

The DevSecOps Dashboard Projects module provides the ability to:

- Retrieve a paginated list of projects with filtering capabilities (search, status, client, specialization, period).
- Expand each project to view its associated repositories with onboarded date and status.
- Sort projects by name or onboarded date.
- Perform actions on projects: mark as "Not Applicable" (with reason, comments, and optional evidence file) or mark as "Complete".

All errors encountered during service execution must be logged to the `error_log` table in the database.

Stack: Python

---

## 2. Scope

### In Scope

- Retrieve paginated project list with filtering (search, status, client, specialization, period).
- Sorting by project name or onboarded date (asc/desc).
- Expand project to show associated repositories (name, onboarded date, status).
- Mark a project as "Not Applicable" with reason category, comments, and optional evidence file attachment.
- Mark a project as "Complete" (updates status to Completed).
- Log all service errors to the `error_log` table.
- User authentication/authorization implementation (assumes JWT middleware is in place).

### Out of Scope

- Repository detail view (covered in a separate story).
- Jira ticket creation logic for "Not Applicable" action (external integration).
- Azure DevOps sync operations.
- Settings and notification configuration.

---

## 3. Technical Details

### 3.1 Database Details

Refer the er_diagram.mmd file.

**Required Tables:**

| Table                | Purpose                                                                      |
|----------------------|------------------------------------------------------------------------------|
| `projects`           | Stores project records (name, onboarded_date, status, client, is_applicable) |
| `statuses`           | Lookup table for project statuses (Active, Inactive, At Risk, Completed, Not Applicable) |
| `specializations`    | Stores specialization records for filtering                                  |
| `repositories`       | Stores repositories linked to projects via devsecops_tickets                 |
| `devsecops_tickets`  | Links projects to repositories and specializations                           |
| `jira_tickets`       | Stores Jira ticket details for "Not Applicable" actions                      |
| `error_log`          | Logs service errors (error_message, error_function, error_file, stack_trace) |

### 3.2 API Endpoints

#### Service 1: GetProjectList (Subtask: ZDAD-51)

| Method | Path               | Description                              |
|--------|--------------------|------------------------------------------|
| GET    | `/api/v1/projects` | Retrieve paginated list of projects with filters |

**Query Parameters:**

| Parameter       | Type    | Required | Default      | Description                                      |
|-----------------|---------|----------|--------------|--------------------------------------------------|
| `period`        | string  | No       | `last_month` | Enum: `last_week`, `last_month`, `last_3_months`, `last_6_months`, `last_year` |
| `search`        | string  | No       | —            | Free text search on project name                 |
| `status`        | string  | No       | —            | Comma-separated list of status IDs to filter     |
| `client`        | string  | No       | —            | Comma-separated list of client IDs to filter     |
| `specialization`| string  | No       | —            | Comma-separated list of specialization IDs       |
| `offset`        | integer | No       | `0`          | Number of items to skip for pagination           |
| `limit`         | integer | No       | `10`         | Number of items per page                         |
| `sort_by`       | string  | No       | `project`    | Enum: `project`, `onboarded_date`                |
| `sort_order`    | string  | No       | `asc`        | Enum: `asc`, `desc`                              |

**Response (200):**
```json
{
  "status_code": 200,
  "status": "success",
  "message": "Projects retrieved successfully",
  "data": {
    "projects": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "name": "Project Xenon",
        "onboardedDate": "2026-02-20",
        "repositoryCount": 3,
        "status": "COMPLETED",
        "repositories": [
          {
            "id": "880e8400-e29b-41d4-a716-446655440030",
            "name": "xenon-infra-core",
            "onboardedDate": "2026-02-27",
            "status": "PASSED"
          }
        ]
      }
    ],
    "pagination": {
      "offset": 0,
      "limit": 10,
      "totalItems": 44
    }
  }
}
```

**Error Responses:**
- `400` — Invalid query parameters
- `401` — Unauthorized (missing or invalid token)
- `500` — Server error (logged to `error_log`)

---

#### Service 2: ManageProjectDetail (Subtask: ZDAD-52)

| Method | Path                      | Description                              |
|--------|---------------------------|------------------------------------------|
| POST   | `/api/v1/projects/action` | Perform action on a project (mark_not_applicable or mark_complete) |

**Content Type:** `multipart/form-data`

**Request Body:**

| Field            | Type   | Required | Description                                              |
|------------------|--------|----------|----------------------------------------------------------|
| `project_id`     | UUID   | Yes      | UUID of the project to perform the action on             |
| `action`         | string | Yes      | Enum: `mark_not_applicable`, `mark_complete`             |
| `reason_category`| string | Conditional | Reason category (required for `mark_not_applicable`)  |
| `comments`       | string | Conditional | Comments explaining the reason (required for `mark_not_applicable`) |
| `evidence_file`  | binary | No       | Evidence file attachment (optional, for `mark_not_applicable`) |

**Response (200):**
```json
{
  "status_code": 200,
  "status": "success",
  "message": "Action performed successfully"
}
```

**Error Responses:**
- `400` — Invalid request body or validation error
- `401` — Unauthorized (missing or invalid token)
- `404` — Project not found
- `500` — Server error (logged to `error_log`)

### 3.3 Affected Components

| Layer        | File/Module                               | Change Description                                                                 |
|--------------|-------------------------------------------|------------------------------------------------------------------------------------|
| Route        | `src/routes/projects_route.py`            | Define GET `/projects` and POST `/projects/action` endpoints                       |
| Service      | `src/services/projects_service.py`        | Business logic for listing projects with filters and performing project actions     |
| Repository   | `src/repositories/projects_repository.py` | Data access for `projects`, `statuses`, `repositories`, `devsecops_tickets` tables |
| Schema       | `src/repositories/schema/projects.py`     | SQLAlchemy ORM models for `projects` and `statuses`                                |
| Schema       | `src/repositories/schema/repositories.py` | SQLAlchemy ORM model for `repositories`                                            |
| Schema       | `src/repositories/schema/jira_tickets.py` | SQLAlchemy ORM model for `jira_tickets`                                            |
| Schema       | `src/repositories/schema/error_log.py`    | SQLAlchemy ORM model for `error_log`                                               |
| Model        | `src/models/projects_model.py`            | Pydantic request/response models for projects                                      |
| Utility      | `src/utils/error_logger.py`               | Utility to log errors to the `error_log` table                                     |

---

## 4. Acceptance Criteria

### AC-1: Retrieve Project List
- **Given** projects exist in the database,
- **When** a GET request is made to `/api/v1/projects`,
- **Then** the API returns a paginated list of projects with `id`, `name`, `onboardedDate`, `repositoryCount`, `status`, and nested `repositories`.

### AC-2: Filter by Search
- **Given** projects exist in the database,
- **When** a GET request includes the `search` query parameter,
- **Then** only projects whose name matches the search text are returned.

### AC-3: Filter by Status
- **Given** projects exist with various statuses,
- **When** a GET request includes `status` as a comma-separated list,
- **Then** only projects matching the specified statuses are returned.

### AC-4: Filter by Client
- **Given** projects exist with various clients,
- **When** a GET request includes `client` as a comma-separated list,
- **Then** only projects matching the specified clients are returned.

### AC-5: Filter by Specialization
- **Given** projects are linked to specializations via `devsecops_tickets`,
- **When** a GET request includes `specialization` as a comma-separated list,
- **Then** only projects associated with the specified specializations are returned.

### AC-6: Pagination
- **Given** more projects exist than the `limit`,
- **When** a GET request includes `offset` and `limit`,
- **Then** the response returns the correct subset and `pagination.totalItems` reflects the total count.

### AC-7: Sorting
- **Given** projects exist in the database,
- **When** a GET request includes `sort_by` and `sort_order`,
- **Then** the projects are returned sorted by the specified field and direction.

### AC-8: Expand Repositories
- **Given** a project has associated repositories,
- **When** the project is returned in the list,
- **Then** the `repositories` array includes each repository's `id`, `name`, `onboardedDate`, and `status`.

### AC-9: Mark Not Applicable
- **Given** a valid `project_id` exists,
- **When** a POST request is made with action `mark_not_applicable`, `reason_category`, and `comments`,
- **Then** the project's `is_applicable` is set to `FALSE`, status is updated to "Not Applicable", and a Jira ticket record is created in `jira_tickets`.

### AC-10: Mark Complete
- **Given** a valid `project_id` exists,
- **When** a POST request is made with action `mark_complete`,
- **Then** the project's status is updated to "Completed" and `completed_at` is set to the current timestamp.

### AC-11: Project Not Found
- **Given** the `project_id` does not exist,
- **When** a POST request is made to `/api/v1/projects/action`,
- **Then** the API returns a `404` response with message "Project not found".

### AC-12: Validation Error
- **Given** an invalid request body (e.g., missing required fields for `mark_not_applicable`, invalid UUID format),
- **When** a POST request is made,
- **Then** the API returns a `400` response with a descriptive validation error message.

### AC-13: Error Logging
- **Given** any unhandled exception occurs during service execution,
- **When** the error is caught,
- **Then** the error details (message, function name, file name, stack trace) are logged to the `error_log` table.

---

## 5. UI Reference

Refer to the attached UI screenshots for the visual layout of:
- Project Onboarding Tracker table (columns: Project, Onboarded Date, No. of Repositories, Status, Actions)
- Filter bar (Search projects, All Statuses dropdown, All Clients dropdown, Specialization dropdown)
- Expandable project row showing nested repositories (columns: Repositories, Onboarded Date, Status)
- Actions menu (three-dot menu for mark_not_applicable and mark_complete)

---

## 6. Testing Requirements

- GET endpoint returns paginated project list with correct structure.
- GET endpoint filters correctly by search, status, client, and specialization.
- GET endpoint supports sorting by project name and onboarded date (asc/desc).
- GET endpoint returns correct pagination metadata (offset, limit, totalItems).
- GET endpoint returns repositories nested within each project.
- GET endpoint returns 400 for invalid query parameters.
- POST endpoint marks project as "Not Applicable" with reason and comments.
- POST endpoint marks project as "Complete" and sets `completed_at`.
- POST endpoint accepts optional evidence file for `mark_not_applicable`.
- POST endpoint returns 404 for non-existent project.
- POST endpoint returns 400 for missing required fields.
- Error logging captures exceptions with full context (function, file, stack trace).

---

## 7. Assumptions

- JWT authentication middleware is already in place and validates the Bearer token.
- Projects are linked to repositories through the `devsecops_tickets` table.
- The `statuses` table is pre-populated with: Active, Inactive, At Risk, Completed, Not Applicable.
- "Mark Not Applicable" creates a record in `jira_tickets` table to track the reason and evidence.
- "Mark Complete" updates the project status and sets `completed_at` timestamp.
- Repository status (PASSED/FAILED) is determined by pipeline run results from Azure DevOps sync.
- Evidence file upload is handled via multipart/form-data and stored externally (e.g., S3).

---

## 8. Dependencies

- Database with the required tables (`projects`, `statuses`, `specializations`, `repositories`, `devsecops_tickets`, `jira_tickets`, `error_log`) already provisioned.
- JWT authentication middleware for route protection.
- OpenAPI specification (`api/openapi.yaml`) for contract validation.
- ER diagram (`design/er_diagram.mmd`) for schema reference.
- File storage service for evidence file uploads (mark_not_applicable action).
