## SOLDEF-ZDAD-58 - ServiceNow Integration for Project Ingestion and DevSecOps Ticket Fetching

### <u>Project Details</u>
- **Project ID:** ZDAD  
- **Project Name:** DevSecOps Jira Dashboard  

### <u>Story Details</u>
- **Story ID:** ZDAD-58  
- **Story Name:** ServiceNow Integration for Project Ingestion and DevSecOps Ticket Fetching  
- **Story Description:**  
  Enable automated data ingestion from ServiceNow into the DevSecOps Jira Dashboard. ServiceNow acts as the source of truth for project onboarding and DevSecOps ticket creation. When a project is created in ServiceNow, a script triggers the Projects Sync API to push project data into the dashboard database. When a DevSecOps ticket is completed in ServiceNow, a script triggers the DevSecOps Tickets Sync API to push ticket and repository data. Both endpoints perform upsert operations, authenticate via the same JWT mechanism used across the application (with an additional `servicenow_sync` role check), and return standardized responses.

### <u> Table of Contents </u>
- [Section 1: Functional Requirements](#Section-1:-Functional-Requirements)
    - [1.1 Overview](#1.1-Overview)
    - [1.2 Requirement Details](#1.2-Requirement-Details)
    - [1.3 Project Artifacts](#1.3-Project-Artifacts)
    - [1.4 Dependencies](#1.4-Dependencies)
- [Section 2: Non Functional Requirements](#Section-2:-Non-Functional-Requirements)
- [2.1 Infrastructure and Deployment](#2.1-Infrastructure-and-Deployment)
    - [2.1.1 Overview](#2.1-Overview)
    - [2.1.2 Requirement Details](#2.2-Requirement-Details)
    - [2.1.3 Project Artifacts](#2.3-Project-Artifacts)
    - [2.1.4 Dependencies](#2.4-Dependencies)
- [2.2 Architecture and System Design](#2.2-Architecture-and-System-Design)
    - [2.2.1 Security and Compliance](#2.2.1-Security-and-Compliance)
    - [2.2.2 System Performance](#2.2.2-System-Performance)
    - [2.2.3 Availability and Reliability](#2.2.3-Availability-and-Reliability)
    - [2.2.4 Cost Efficiency](#2.2.4-Cost-Efficiency)
    - [2.2.5 Traceability and Observability](#2.2.5-Traceability-and-Observability)
- [Section 3: In Scope and Out Scope](#Section-3:-In-Scope-and-Out-Scope)
    - [3.1 In Scope Details](#1.1-In-Scope-Details)
    - [3.2 Out Scope Details](#1.2-Out-Scope-Details)
- [Section 4: Solution Diagrams](#Section-4:-Solution-Diagrams)
    - [4.1 UI/UX Design Diagram](#4.1-UI/UX-Design-Diagram)
    - [4.2 Architecture Design Diagram](#4.2-Architecture-Design-Diagram)
    - [4.3 Infrastructure Design Diagram](#4.3-Infrastructure-Design-Diagram)

### <u> Section 1: Functional Requirements </u>

#### <u> 1.1 Overview </u>

The ServiceNow Integration story implements two inbound sync endpoints that allow ServiceNow to push project and DevSecOps ticket data into the DevSecOps Jira Dashboard database. ServiceNow is the source of truth for project onboarding — when a new project is created in ServiceNow, a scheduled or event-driven script calls `POST /api/v1/sync/servicenow/projects` to create or update the project record. Similarly, when a DevSecOps ticket is completed in ServiceNow, a script calls `POST /api/v1/sync/servicenow/devsecops-tickets` to sync ticket and associated repository data. Both endpoints authenticate using the same JWT Bearer token mechanism as the rest of the application, with an additional authorization check ensuring the caller has the `servicenow_sync` role. The sync operations are idempotent upserts keyed on `sn_project_id` for projects and the combination of `sn_project_id` + `specialization_name` for tickets. Partial success is supported for batch ticket syncing — valid tickets are processed while failures are logged and reported in the response. All errors encountered during processing are logged to the `error_log` database table for traceability.

#### <u> 1.2 Requirement Details </u>

- **ZDAD-58-FR01: Project Sync from ServiceNow**
- **ZDAD-58-FR02: DevSecOps Tickets Sync from ServiceNow**
- **ZDAD-58-FR03: ServiceNow Role-Based Authorization**
- **ZDAD-58-FR04: Request Payload Validation**
- **ZDAD-58-FR05: Error Logging for Sync Operations**

##### <u> 1.2.1 ZDAD-58-FR01: Project Sync from ServiceNow </u>

##### Description:
The system shall expose a `POST /api/v1/sync/servicenow/projects` endpoint that receives project data from ServiceNow and performs an upsert operation in the `projects` table. The `sn_project_id` field serves as the unique identifier to determine whether to insert a new project or update an existing one. This endpoint is triggered by a ServiceNow script when a new project is created or an existing project is modified in ServiceNow.

##### Request Payload:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sn_project_id` | string | Yes | ServiceNow project identifier (unique key for upsert) |
| `project_name` | string | Yes | Name of the project |
| `onboarded_date` | date (ISO 8601) | Yes | Date when the project was onboarded (format: YYYY-MM-DD) |
| `project_type` | string | Yes | Type/category of the project (e.g., "Infrastructure") |
| `is_applicable` | boolean | No | Whether the project is applicable for DevSecOps (default: true) |
| `client` | string | No | Client name associated with the project |
| `approver` | string | No | Name or email of the approver for the project |

##### Processing Logic:
1. Authenticate the request using JWT Bearer token validation (same middleware as other endpoints).
2. Verify the authenticated user has the `servicenow_sync` role — return 401 if not authorized.
3. Validate the request payload against the Pydantic schema (required fields, data types, date format).
4. Look up the project by `sn_project_id` in the `projects` table:
   - **If found** → Update the existing record with the new values (`project_name`, `onboarded_date`, `project_type`, `is_applicable`, `client`, `approver`). Set `modified_at` to current timestamp and `modified_by` to the authenticated service account identifier.
   - **If not found** → Create a new project record with:
     - A generated UUID as `project_id`
     - `status_id` set to the "Inactive" status (looked up from the `statuses` table by name)
     - `is_active` set to 1
     - `created_at` set to current timestamp
     - `created_by` set to the authenticated service account identifier
5. Return a success response with HTTP 200.

##### Success Response:
```json
{
  "status_code": 200,
  "status": "success",
  "message": "Sync completed successfully"
}
```

##### Error Responses:
| Status Code | Condition |
|-------------|-----------|
| 400 | Invalid or missing required fields (e.g., missing `sn_project_id`, invalid date format) |
| 401 | Missing/invalid JWT token or user does not have `servicenow_sync` role |
| 500 | Unexpected server error (logged to `error_log` table) |

##### Acceptance Criteria:
- A new project is created in the `projects` table when `sn_project_id` does not exist.
- An existing project is updated when `sn_project_id` already exists in the `projects` table.
- New projects are assigned the "Inactive" status by default (resolved from `statuses` table).
- The `created_by` and `modified_by` fields reflect the authenticated service account.
- Invalid payloads return HTTP 400 with a descriptive error message identifying the invalid field.
- Unauthorized requests (missing token or wrong role) return HTTP 401.
- The endpoint is idempotent — calling it multiple times with the same data produces the same result.

##### <u> 1.2.2 ZDAD-58-FR02: DevSecOps Tickets Sync from ServiceNow </u>

##### Description:
The system shall expose a `POST /api/v1/sync/servicenow/devsecops-tickets` endpoint that receives a batch of DevSecOps ticket data from ServiceNow and performs upsert operations in the `devsecops_tickets` and `repositories` tables. This endpoint is triggered by a ServiceNow script when a DevSecOps ticket is completed. Each ticket is linked to a project (via `sn_project_id`) and a specialization (via `specialization_name`). The endpoint supports partial success — valid tickets are processed while invalid ones are logged and reported.

##### Request Payload:
**Root object:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tickets` | array | Yes | List of ticket objects to sync |

**Each ticket object:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sn_project_id` | string | Yes | ServiceNow project identifier (must reference an existing project) |
| `project_name` | string | Yes | Name of the project the ticket belongs to |
| `client` | string | No | Client name associated with the ticket |
| `specialization_name` | string | Yes | Specialization name (must match an existing active specialization) |
| `repositories` | array | No | List of repository objects associated with the ticket |
| `requested_by` | string | No | Name or email of the person who raised the request |
| `approver` | string | No | Name or email of the approver for the ticket |
| `requested_at` | datetime (ISO 8601) | No | Timestamp when the ticket was requested in ServiceNow |

**Each repository object:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `repo_name` | string | Yes | Repository name |
| `ado_repo_id` | string | No | Azure DevOps repository identifier |

##### Processing Logic:
1. Authenticate the request using JWT Bearer token validation.
2. Verify the authenticated user has the `servicenow_sync` role — return 401 if not authorized.
3. Validate the root payload structure (must contain a non-empty `tickets` array).
4. For each ticket in the array:
   a. **Resolve `specialization_name`** → Query the `specializations` table for a record where `specialization_name` matches (case-insensitive) and `is_active = 1`. If not found, mark this ticket as failed and continue to the next ticket.
   b. **Resolve `sn_project_id`** → Query the `projects` table for a record where `sn_project_id` matches and `is_active = 1`. If not found, mark this ticket as failed and continue to the next ticket.
   c. **Upsert the ticket** → Look up existing ticket by `sn_project_id` + `specialization_id` combination in `devsecops_tickets`:
      - **If found** → Update the existing record with new values. Set `modified_at` and `modified_by`.
      - **If not found** → Create a new record with generated UUID as `ticket_id`, resolved `specialization_id`, resolved `project_id`, and `is_active = 1`. Set `created_at` and `created_by`.
   d. **Process repositories** → For each repository in the ticket:
      - Look up by `repo_name` + `ticket_id` in the `repositories` table.
      - **If found** → Update `ado_repo_id` if provided. Set `modified_at` and `modified_by`.
      - **If not found** → Create a new repository record with generated UUID, linked to the ticket via `ticket_id`. Set `created_at` and `created_by`.
5. Return a success response indicating the sync completed. Failed tickets are logged to the application logs with the reason for failure.

##### Success Response:
```json
{
  "status_code": 200,
  "status": "success",
  "message": "Sync completed successfully"
}
```

##### Error Responses:
| Status Code | Condition |
|-------------|-----------|
| 400 | Invalid payload structure (missing `tickets` array, empty array, or missing required fields in ticket objects) |
| 401 | Missing/invalid JWT token or user does not have `servicenow_sync` role |
| 500 | Unexpected server error (logged to `error_log` table) |

##### Partial Success Behavior:
- If some tickets in the batch fail validation (unknown specialization or project not found), the valid tickets are still processed.
- Failed tickets are logged at WARNING level with the reason (e.g., "Specialization 'Unknown' not found for ticket with sn_project_id 'SN-PRJ-999'").
- The response still returns HTTP 200 with "Sync completed successfully" as long as at least one ticket was processed.
- If ALL tickets in the batch fail, the response still returns HTTP 200 but the failures are logged.

##### Acceptance Criteria:
- A new ticket is created in `devsecops_tickets` when the `sn_project_id` + `specialization_id` combination does not exist.
- An existing ticket is updated when the combination already exists.
- Repositories are created or updated and linked to the correct ticket via `ticket_id`.
- Unknown `specialization_name` values cause the individual ticket to be skipped (not the entire batch).
- Unknown `sn_project_id` values cause the individual ticket to be skipped (not the entire batch).
- The `created_by` and `modified_by` fields reflect the authenticated service account.
- The endpoint handles empty `repositories` arrays gracefully (ticket is created without repositories).
- Failed tickets are logged with sufficient detail for debugging.

##### <u> 1.2.3 ZDAD-58-FR03: ServiceNow Role-Based Authorization </u>

##### Description:
The system shall enforce role-based access control on both ServiceNow sync endpoints. After standard JWT token validation (signature, expiration, claims), the system performs an additional authorization check to verify the authenticated user has the `servicenow_sync` role. This ensures only the dedicated ServiceNow service account can invoke the sync endpoints, preventing unauthorized access from other authenticated users.

##### Authorization Flow:
1. The JWT middleware validates the token (same as all other endpoints in the application).
2. After successful token validation, the sync route handler (or a dedicated dependency) checks the decoded token claims for the `servicenow_sync` role.
3. The role can be present in a `roles` claim (array) or a `scope` claim (space-separated string) within the JWT payload.
4. If the role is not present, the endpoint returns HTTP 401 with the message: "Insufficient permissions: servicenow_sync role required".

##### Service Account Details:
- The ServiceNow integration uses a dedicated service account (e.g., `svc_servicenow@zeb.co`).
- The service account is provisioned with minimal permissions — only the `servicenow_sync` role.
- The JWT token for this account is issued by the same identity provider used for all dashboard users.

##### Acceptance Criteria:
- Requests with a valid JWT token but without the `servicenow_sync` role receive HTTP 401.
- Requests with a valid JWT token that includes the `servicenow_sync` role are allowed to proceed.
- The role check is applied to both `/api/v1/sync/servicenow/projects` and `/api/v1/sync/servicenow/devsecops-tickets`.
- The error response clearly indicates the missing role/permission.

##### <u> 1.2.4 ZDAD-58-FR04: Request Payload Validation </u>

##### Description:
The system shall validate all incoming request payloads for the sync endpoints using Pydantic v2 models. Validation errors are returned as HTTP 400 responses with descriptive messages identifying which field failed validation and why.

##### Validation Rules — Project Sync:
- `sn_project_id`: Required, non-empty string.
- `project_name`: Required, non-empty string.
- `onboarded_date`: Required, valid ISO 8601 date format (YYYY-MM-DD).
- `project_type`: Required, non-empty string.
- `is_applicable`: Optional, must be boolean if provided.
- `client`: Optional, string.
- `approver`: Optional, string.

##### Validation Rules — DevSecOps Tickets Sync:
- `tickets`: Required, must be a non-empty array.
- Each ticket:
  - `sn_project_id`: Required, non-empty string.
  - `project_name`: Required, non-empty string.
  - `specialization_name`: Required, non-empty string.
  - `repositories`: Optional, must be an array if provided.
  - Each repository: `repo_name` is required and must be a non-empty string.
  - `requested_at`: Optional, must be valid ISO 8601 datetime if provided.

##### Error Response Format:
```json
{
  "status_code": 400,
  "status": "failed",
  "message": "Validation error: 'sn_project_id' is required",
  "data": []
}
```

##### Acceptance Criteria:
- Missing required fields return HTTP 400 with the field name in the error message.
- Invalid date formats return HTTP 400 with a descriptive message.
- Empty `tickets` array returns HTTP 400.
- Validation errors are returned before any database operations are attempted.
- The error response follows the standardized `BaseResponse` schema.

##### <u> 1.2.5 ZDAD-58-FR05: Error Logging for Sync Operations </u>

##### Description:
The system shall log all unhandled exceptions occurring during sync operations to the `error_log` database table. This provides a persistent audit trail for debugging sync failures. The error logging mechanism is the same as used by the Overview API (ZDAD-34-FR04) — a global exception handler captures unhandled exceptions and persists them.

##### Error Log Table Fields:
- `error_id` — Auto-generated UUID primary key.
- `error_message` — The exception message text.
- `error_function` — The function name where the error occurred (e.g., "sync_project", "sync_devsecops_tickets").
- `error_file` — The file path where the error originated.
- `stack_trace` — Full Python stack trace for debugging.
- `created_at` — Timestamp when the error was logged.
- `created_by` — System identifier (e.g., "servicenow_sync_service").

##### Acceptance Criteria:
- All unhandled exceptions in the sync endpoints are captured and logged to the `error_log` table.
- The error log entry contains the function name, file name, error message, and stack trace.
- Error logging does not interfere with the error response returned to ServiceNow (non-blocking).
- HTTP 500 responses are always accompanied by an `error_log` entry.
- Partial failures (individual ticket validation failures) are logged at application log level (WARNING), not in the `error_log` table.

#### <u> 1.3 Database Schema </u>

The following tables are directly involved in the ServiceNow sync operations (as defined in `design/er_diagram.mmd`):

##### projects
| Column | Type | Constraints |
|--------|------|-------------|
| project_id | UUID | PK |
| status_id | UUID | FK → statuses.status_id |
| sn_project_id | VARCHAR | ServiceNow project identifier (unique key for upsert) |
| project_name | VARCHAR | NOT NULL |
| onboarded_date | DATE | NOT NULL |
| project_type | VARCHAR | NOT NULL |
| is_applicable | BOOLEAN | DEFAULT TRUE |
| client | VARCHAR | |
| ado_project_id | VARCHAR | |
| completed_at | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### devsecops_tickets
| Column | Type | Constraints |
|--------|------|-------------|
| ticket_id | UUID | PK |
| specialization_id | UUID | FK → specializations.specialization_id |
| project_id | UUID | NULLABLE |
| sn_project_id | VARCHAR | FK → projects.sn_project_id |
| project_name | VARCHAR | NOT NULL |
| client | VARCHAR | |
| requested_by | VARCHAR | |
| approver | VARCHAR | |
| sync_method | VARCHAR | Indicates sync source (e.g., "servicenow") |
| requested_at | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### repositories
| Column | Type | Constraints |
|--------|------|-------------|
| repository_id | UUID | PK |
| ticket_id | UUID | FK → devsecops_tickets.ticket_id |
| repository_name | VARCHAR | NOT NULL |
| ado_repo_id | VARCHAR | |
| pipeline_runs_count | INTEGER | DEFAULT 0 |
| success_rate | FLOAT | DEFAULT 0 |
| last_run_at | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### specializations (read-only reference)
| Column | Type | Constraints |
|--------|------|-------------|
| specialization_id | UUID | PK |
| specialization_name | VARCHAR | NOT NULL |
| is_active | INT | DEFAULT 1 |

##### statuses (read-only reference)
| Column | Type | Constraints |
|--------|------|-------------|
| status_id | UUID | PK |
| status_name | VARCHAR | NOT NULL |
| is_active | INT | DEFAULT 1 |

##### error_log
| Column | Type | Constraints |
|--------|------|-------------|
| error_id | UUID | PK |
| error_message | TEXT | NOT NULL |
| error_function | TEXT | NOT NULL |
| error_file | TEXT | NOT NULL |
| stack_trace | TEXT | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### Key Relationships:
- `projects.status_id` → `statuses.status_id` (each project has one status)
- `devsecops_tickets.specialization_id` → `specializations.specialization_id` (each ticket belongs to one specialization)
- `devsecops_tickets.sn_project_id` → `projects.sn_project_id` (each ticket references a project)
- `repositories.ticket_id` → `devsecops_tickets.ticket_id` (each repository belongs to one ticket)

#### <u> 1.4 Project Artifacts </u>

- `api/openapi.yaml` — Full OpenAPI 3.0.3 specification defining the sync endpoints, request schemas (`SyncProjectRequest`, `SyncDevSecOpsTicketsRequest`, `SyncDevSecOpsTicketItem`), and response schemas (`SyncSuccessResponse`, `SyncConflictError`).
- `design/er_diagram.mmd` — Mermaid ER diagram showing all database tables and relationships including `projects`, `devsecops_tickets`, `repositories`, `specializations`, `statuses`, and `error_log`.
- `design/requirements.md` — Section 2 (ServiceNow Sync) contains the high-level requirements for this story.

#### <u> 1.5 Dependencies </u>

- **Python 3.12+** — Runtime environment
- **FastAPI** — Web framework for building the REST API
- **SQLAlchemy** — ORM for PostgreSQL database access (upsert operations, relationship queries)
- **Pydantic v2** — Request payload validation and serialization
- **PostgreSQL** — Primary database storing projects, tickets, repositories, specializations, and error logs
- **python-jose / PyJWT** — JWT token decoding and validation (shared with existing auth middleware)
- **Uvicorn** — ASGI server for running the FastAPI application
- **ServiceNow (External)** — Source system that triggers the sync via HTTP POST requests


### <u> Section 2: Non Functional Requirements </u>

### 2.1 Infrastructure and Deployment

#### <u> 2.1.1 Overview </u>

The ServiceNow Sync endpoints are deployed as part of the same DevSecOps Jira Dashboard backend application on Azure App Service. No additional infrastructure is required — the sync routes are registered within the existing FastAPI application alongside the Overview API routes. The application connects to the same PostgreSQL database instance. Since ServiceNow initiates the sync calls, no outbound network configuration is needed. The deployment leverages the existing Docker containerization, health check endpoints, and environment variable configuration established in ZDAD-34. The only infrastructure consideration is ensuring the Azure App Service network rules allow inbound traffic from the ServiceNow instance IP range.

#### <u> 2.1.2 Requirement Details </u>

- **ZDAD-58-NFR01: Shared Deployment with Existing Application**
- **ZDAD-58-NFR02: Network Access Configuration**

##### <u> 2.1.2.1 ZDAD-58-NFR01: Shared Deployment with Existing Application </u>

##### Description:
The ServiceNow sync endpoints shall be deployed as additional routes within the existing FastAPI application. No separate service or deployment is required. The sync routes are registered in the application startup alongside the Overview API routes and share the same database connection pool, JWT middleware, and error handling infrastructure.

##### Deployment Configuration:
- **Route Registration:** Sync routes are added via a dedicated `APIRouter` with prefix `/api/v1/sync/servicenow`.
- **Database:** Uses the same SQLAlchemy engine and session factory as the Overview API.
- **Authentication:** Uses the same JWT middleware — no additional auth infrastructure needed.
- **Error Handling:** Uses the same global exception handler that logs to `error_log`.

##### Acceptance Criteria:
- The sync endpoints are accessible after deployment without additional infrastructure changes.
- The existing Overview API and Filters API continue to function without disruption.
- The application starts successfully with the new routes registered.
- Health check and readiness endpoints remain unaffected.

##### <u> 2.1.2.2 ZDAD-58-NFR02: Network Access Configuration </u>

##### Description:
The Azure App Service network configuration shall allow inbound HTTPS traffic from the ServiceNow instance. This may require adding the ServiceNow IP range to the App Service access restrictions or ensuring the App Service is accessible from the corporate network where ServiceNow resides.

##### Acceptance Criteria:
- ServiceNow can reach the sync endpoints over HTTPS from its configured network.
- The network configuration does not expose the endpoints to unauthorized external traffic.
- Access restrictions are documented for the operations team.

#### <u> 2.1.3 Project Artifacts </u>

- `api/openapi.yaml` — API specification including sync endpoint definitions
- `design/er_diagram.mmd` — Database schema reference for sync operations

### 2.2 Architecture and System Design

#### <u> 2.2.1 Security and Compliance </u>

##### JWT Authentication with Role Check:
Both sync endpoints require a valid JWT Bearer token in the `Authorization` header. The existing JWT middleware validates the token signature, expiration, and required claims. After standard validation, a dedicated dependency function checks for the `servicenow_sync` role in the token claims. This two-layer approach ensures that even if a valid user token is used, only the ServiceNow service account can invoke the sync endpoints.

##### Minimal Privilege Principle:
- The ServiceNow service account is provisioned with only the `servicenow_sync` role.
- The service account cannot access any other endpoints (Overview, Projects, Settings, etc.).
- The role check is implemented as a FastAPI dependency that can be reused across all sync routes.

##### Input Validation and SQL Injection Prevention:
- All request payloads are validated using Pydantic v2 models with strict type constraints before any database operation.
- SQLAlchemy ORM with parameterized queries is used exclusively — no raw SQL is constructed from user input.
- String fields are validated for maximum length to prevent buffer-related issues.

##### Audit Trail:
- All sync requests are logged at INFO level with: caller identity (from JWT claims), timestamp, endpoint called, and a summary of the payload (number of tickets, project ID).
- Sensitive data (full JWT token) is never logged.
- Failed authorization attempts are logged at WARNING level with the caller identity.

#### <u> 2.2.2 System Performance </u>

##### Database Operation Efficiency:
- Project sync performs a single SELECT followed by either an INSERT or UPDATE — two database operations maximum per request.
- Ticket sync processes tickets sequentially within a single database session, committing after all tickets are processed (batch commit).
- Specialization and project lookups use indexed columns (`specialization_name`, `sn_project_id`) for efficient resolution.
- SQLAlchemy connection pooling is shared with the existing application, reusing connections efficiently.

##### Payload Size Considerations:
- The tickets endpoint accepts arrays of tickets. While no hard limit is enforced, the application handles reasonable batch sizes (up to 50 tickets per request) without timeout issues.
- Large payloads are processed within the default request timeout configured on Azure App Service.

#### <u> 2.2.3 Availability and Reliability </u>

##### Idempotent Operations:
- Both sync endpoints are idempotent — calling them multiple times with the same data produces the same database state.
- This ensures that ServiceNow can safely retry failed requests without causing data duplication or corruption.

##### Partial Failure Handling:
- The tickets endpoint processes each ticket independently. A failure in one ticket does not prevent processing of subsequent tickets.
- Database transactions are managed per-request — if an unhandled exception occurs mid-batch, the entire transaction is rolled back to maintain data consistency.

##### Error Resilience:
- Transient database connection failures are handled by SQLAlchemy's connection pool retry mechanism.
- The global exception handler ensures all unhandled errors return a proper HTTP 500 response and are logged.
- The application does not crash on invalid input — all validation errors are caught and returned as HTTP 400.

#### <u> 2.2.4 Cost Efficiency </u>

##### Resource Optimization:
- No additional infrastructure is required — the sync endpoints share the existing App Service and database.
- Sync operations are lightweight (simple upserts) and do not require additional compute resources.
- Database connection pooling prevents connection exhaustion from frequent sync calls.
- No caching is needed for sync operations as they are write-heavy and infrequent.

#### <u> 2.2.5 Traceability and Observability </u>

##### Structured Logging:
- All sync operations use the same JSON-formatted structured logging as the rest of the application.
- Each sync request is assigned a `trace_id` for end-to-end request tracing.
- Log entries include: `timestamp`, `level`, `logger`, `filename`, `line_number`, `message`, and contextual fields (`trace_id`, `sn_project_id`, `tickets_count`).

##### Sync-Specific Logging:
- INFO: Successful sync operations with summary (e.g., "Project SN-PRJ-00456 synced successfully", "3 tickets processed, 1 skipped").
- WARNING: Individual ticket failures within a batch (e.g., "Specialization 'Unknown' not found").
- ERROR: Unhandled exceptions (also persisted to `error_log` table).

##### Error Persistence:
- All unhandled exceptions are persisted to the `error_log` database table with full context.
- Error log entries include `created_by` set to "servicenow_sync_service" for easy filtering.


### <u> Section 3: In Scope and Out Scope </u>

#### <u> 3.1 Inscope Details </u>

- Implementation of `POST /api/v1/sync/servicenow/projects` endpoint for project data ingestion from ServiceNow
- Implementation of `POST /api/v1/sync/servicenow/devsecops-tickets` endpoint for ticket and repository data ingestion from ServiceNow
- JWT Bearer token authentication using the existing middleware (shared with Overview API)
- Role-based authorization check for `servicenow_sync` role on both sync endpoints
- Pydantic v2 request models for payload validation (`SyncProjectRequest`, `SyncDevSecOpsTicketsRequest`)
- Upsert logic for projects keyed on `sn_project_id`
- Upsert logic for tickets keyed on `sn_project_id` + `specialization_id` combination
- Upsert logic for repositories keyed on `repo_name` + `ticket_id` combination
- Resolution of `specialization_name` to `specialization_id` from the `specializations` table
- Resolution of `sn_project_id` to `project_id` from the `projects` table
- Default status assignment ("Inactive") for newly created projects
- Partial success handling for batch ticket sync (valid tickets processed, invalid ones logged)
- Standardized response format following `BaseResponse` schema
- Error handling with HTTP 400, 401, and 500 responses
- Error logging to the `error_log` database table for unhandled exceptions
- SQLAlchemy ORM models for `projects`, `devsecops_tickets`, `repositories` tables (if not already created in ZDAD-34)
- Structured JSON logging with trace_id for sync request tracing
- Layered architecture: Routes → Services → Repositories following existing project patterns

#### <u> 3.2 Outscope Details </u>

- ServiceNow script development (ServiceNow side is managed by the ServiceNow team)
- Azure DevOps (ADO) sync functionality (`POST /api/v1/sync/ado` is a separate story)
- KPI history computation and population (handled by ADO Sync API)
- Rate limiting implementation (to be addressed in a future infrastructure story)
- Concurrent request protection / 409 conflict handling (deferred to future iteration)
- UI/frontend changes (this is a backend-only story)
- Database migration scripts (schema is assumed to already exist from prior setup)
- ServiceNow service account provisioning (handled by identity/access management team)
- Network/firewall configuration on Azure (handled by infrastructure team)
- Report generation and email delivery
- Settings management endpoints
- Project action endpoints (mark_not_applicable, mark_complete)
- Repository detail endpoint

### <u> Section 4: Solution Diagrams </u>

#### <u> 4.1 UI/UX Design Diagram </u>

**Diagram Location:** N/A — This is a backend-only story with no UI components.

#### <u> 4.2 Architecture Design Diagram </u>

**Diagram Location:** `diagram/ZDAD-58_servicenow_sync_flow.mmd`

#### <u> 4.3 Infrastructure Design Diagram </u>

**Diagram Location:** N/A — Uses existing Azure App Service infrastructure established in ZDAD-34.
