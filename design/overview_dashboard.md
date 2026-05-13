## SOLDEF-ZDAD-34 - Implement Overview Dashboard API

### <u>Project Details</u>
- **Project ID:** ZDAD  
- **Project Name:** DevSecOps Jira Dashboard  

### <u>Story Details</u>
- **Story ID:** ZDAD-34  
- **Story Name:** Implement Overview Dashboard API  
- **Story Description:**  
  When accessing the DevSecOps dashboard, the ability to view a consolidated overview of project KPIs, status distribution and specialization filters is needed so that delivery leads can quickly assess portfolio health and take timely action on at-risk projects.
- **Scope:**  
  The goal is to implement the Overview Dashboard API for the DevSecOps Dashboard. The `GET /api/v1/overview` endpoint will return KPI tiles, status distribution, attention banner and most overdue projects list in a single call. The `GET /api/v1/specializations` endpoint will provide specialization dropdown values for filtering. This consolidation streamlines data retrieval, improves performance and enhances the user experience for team members accessing the dashboard.
- **Acceptance Criteria:**
  - The `GET /api/v1/overview` endpoint returns KPI tiles (Total Projects, Adopted, Adoption Rate, At Risk, Active, Inactive), status distribution data, attention banner details, and the most overdue projects list in one response.
  - The `GET /api/v1/specializations` endpoint returns specialization values to populate the dropdown filter on the Overview page.
  - The overview data accurately reflects the current state of all onboarded projects.
  - Both endpoints handle errors like database unavailability and invalid requests, returning meaningful responses.
  - Unit tests cover positive and negative cases for both endpoints; all defects found during testing are fixed before release.

### <u> Table of Contents </u>
- [Section 1: Functional Requirements](#Section-1:-Functional-Requirements)
    - [1.1 Overview](#1.1-Overview)
    - [1.2 Requirement Details](#1.2-Requirement-Details)
    - [1.3 Database Schema](#1.3-Database-Schema)
    - [1.4 Project Artifacts](#1.4-Project-Artifacts)
    - [1.5 Dependencies](#1.5-Dependencies)
- [Section 2: Non Functional Requirements](#Section-2:-Non-Functional-Requirements)
- [2.1 Infrastructure and Deployment](#2.1-Infrastructure-and-Deployment)
    - [2.1.1 Overview](#2.1-Overview)
    - [2.1.2 Requirement Details](#2.2-Requirement-Details)
    - [2.1.3 Project Artifacts](#2.3-Project-Artifacts)
- [2.2 Architecture and System Design](#2.2-Architecture-and-System-Design)
    - [2.2.1 Security and Compliance](#2.2.1-Security-and-Compliance)
    - [2.2.2 System Performance](#2.2.2-System-Performance)
    - [2.2.3 Availability and Reliability](#2.2.3-Availability-and-Reliability)
    - [2.2.4 Cost Efficiency](#2.2.4-Cost-Efficiency)
    - [2.2.5 Traceability and Observability](#2.2.5-Traceability-and-Observability)
- [Section 3: In Scope and Out Scope](#Section-3:-In-Scope-and-Out-Scope)
    - [3.1 In Scope Details](#3.1-In-Scope-Details)
    - [3.2 Out Scope Details](#3.2-Out-Scope-Details)
- [Section 4: Solution Diagrams](#Section-4:-Solution-Diagrams)
    - [4.1 UI/UX Design Diagram](#4.1-UI/UX-Design-Diagram)
    - [4.2 Architecture Design Diagram](#4.2-Architecture-Design-Diagram)
    - [4.3 Infrastructure Design Diagram](#4.3-Infrastructure-Design-Diagram)


### <u> Section 1: Functional Requirements </u>

#### <u> 1.1 Overview </u>

The Overview Dashboard API is the primary data source for the DevSecOps Jira Dashboard landing page. It provides delivery leads and stakeholders with a consolidated view of project health across the organization, enabling them to quickly assess portfolio status and take timely action on at-risk projects. The API exposes two endpoints: `GET /api/v1/overview` which returns KPI tiles (Total Projects, Adopted, Adoption Rate, At Risk, Active, Inactive), status distribution data, an attention banner with critical alerts, and the most overdue projects list — all in a single response. The `GET /api/v1/specializations` endpoint provides specialization dropdown values for filtering the overview data. Both endpoints require JWT Bearer token authentication and return standardized JSON responses. The KPI data is pre-computed and stored in the `kpi_history` table (populated by the ADO Sync API), ensuring the Overview API performs efficient read operations without complex on-the-fly calculations. All errors encountered during request processing are logged to the `error_log` database table for traceability and debugging purposes.

#### <u> 1.2 Requirement Details </u>

- **ZDAD-34-FR01: Overview Dashboard Metrics and Data Retrieval**
- **ZDAD-34-FR02: Specializations Filter Endpoint**
- **ZDAD-34-FR03: Attention Banner and Most Overdue Projects**
- **ZDAD-34-FR04: Query Parameter Validation**
- **ZDAD-34-FR05: Error Logging**

##### <u> 1.2.1 ZDAD-34-FR01: Overview Dashboard Metrics and Data Retrieval </u>

##### Description:
The system shall expose a `GET /api/v1/overview` endpoint that returns KPI tiles, status distribution data, attention banner details, and the most overdue projects list in a single consolidated response. The endpoint accepts optional query parameters for filtering by time period and specialization. The KPI data is read from the `kpi_history` table which is pre-computed by the ADO Sync API.

##### Request Parameters:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `period` | string (enum) | No | `last_month` | Time period for metric calculations |
| `specialization` | string (CSV) | No | All | Comma-separated specialization IDs to filter by |

##### Period Options:
- `last_week` — Last 7 days
- `last_month` — Last 30 days
- `last_3_months` — Last 90 days

##### Response Structure - KPI Tiles:
The response `data.metrics` object contains six KPI tiles:
- `totalProjects` — Total number of onboarded projects with `count`, `trend` (increase/decrease/flat/null), `change` (numeric difference from previous period), and `source` (contextual label e.g., "ServiceNow").
- `adopted` — Projects that have completed DevSecOps adoption (fully onboarded) with `count`, `description` ("fully onboarded"), `trend`, and `change`.
- `adoptionRate` — Percentage of projects that have been adopted with `rate` (float percentage), `trend`, and `change`.
- `atRisk` — Projects with overdue onboarding exceeding the configured threshold with `count`, `description` ("overdue onboarding"), `trend`, and `change`.
- `active` — Projects with pipeline activity within the last 10 days with `count`, `description` ("pipeline run ≤ 10d"), `trend`, and `change`.
- `inactive` — Projects with no pipeline activity for 10+ days with `count`, `description` ("no activity 10+ d"), `trend`, and `change`.

##### Response Structure - Status Distribution:
The response `data.statusDistribution` object contains:
- `total` — Total project count.
- `breakdown[]` — Array of objects each with `status` (name), `count`, and `percentage` (float).

##### Response Structure - Attention Banner:
The response `data.attentionBanner` object contains:
- `message` — Summary alert message (e.g., "6 projects are at risk and require immediate attention").
- `atRiskCount` — Number of at-risk projects.
- `overdueCount` — Number of projects past their onboarding deadline.
- `severity` — Alert severity level (`critical`, `warning`, `info`).

##### Response Structure - Most Overdue Projects:
The response `data.mostOverdueProjects` is an array of the top overdue projects (limited to 5), each containing:
- `projectId` — UUID of the project.
- `projectName` — Name of the project.
- `client` — Client name.
- `onboardedDate` — Date the project was onboarded.
- `daysOverdue` — Number of days past the expected completion date.
- `specialization` — Specialization name the project belongs to.

##### Data Source:
- KPI metrics are read from the `kpi_history` table joined with `specializations` table for filtering.
- Status distribution is computed from the `projects` table joined with `statuses` table.
- Attention banner is derived from the at-risk and overdue project counts.
- Most overdue projects are queried from `projects` table where status is "At Risk", ordered by overdue duration descending, limited to 5.
- The `kpi_history` table is populated and maintained by the ADO Sync API (out of scope for this story).

##### Acceptance Criteria:
- The endpoint returns HTTP 200 with correct KPI tiles, status distribution, attention banner, and most overdue projects when called with valid parameters.
- Default period is `last_month` when no period parameter is provided.
- When specialization filter is provided, only metrics for matching specializations are returned.
- When specialization filter is omitted, metrics across all specializations are aggregated.
- Trend values are one of: `increase`, `decrease`, `flat`, or `null`.
- The most overdue projects list contains a maximum of 5 entries sorted by days overdue descending.
- The attention banner severity is `critical` when at-risk count > 5, `warning` when 1-5, and `info` when 0.
- Response follows the standardized `BaseResponse` schema with `status_code`, `status`, `message`, and `data` fields.
- The overview data accurately reflects the current state of all onboarded projects.

##### <u> 1.2.2 ZDAD-34-FR02: Specializations Filter Endpoint </u>

##### Description:
The system shall expose a `GET /api/v1/specializations` endpoint that returns specialization values to populate the dropdown filter on the Overview page. This endpoint provides the valid specialization IDs and names that can be used as filter values in the Overview API.

##### Response Structure:
The response `data` object contains:
- `specializations` — Array of objects each containing:
  - `id` — Unique identifier for the specialization (UUID).
  - `name` — Display name of the specialization (e.g., "DevSecOps", "DevOps", "Cloud Security").

##### Data Source:
- Specializations are read from the `specializations` table where `is_active = 1`.

##### Acceptance Criteria:
- The endpoint returns HTTP 200 with all active specialization values.
- Each specialization object contains `id` and `name` fields.
- Only active records (`is_active = 1`) are returned.
- Response follows the standardized `BaseResponse` schema.
- The endpoint requires JWT Bearer token authentication.
- The response is cacheable with a TTL of 1 hour.


##### <u> 1.2.3 ZDAD-34-FR03: Attention Banner and Most Overdue Projects </u>

##### Description:
The system shall compute and return an attention banner summarizing critical project health alerts, along with a list of the most overdue projects. The attention banner provides delivery leads with an immediate visual indicator of portfolio risk, while the overdue projects list enables targeted action on the highest-priority items.

##### Attention Banner Logic:
1. Count projects with "At Risk" status linked to the requested specialization(s).
2. Count projects where `onboarded_date` + `at_risk_threshold` (from `settings` table) < current date.
3. Determine severity:
   - `critical` — More than 5 at-risk projects.
   - `warning` — 1 to 5 at-risk projects.
   - `info` — No at-risk projects.
4. Compose the banner message dynamically (e.g., "{count} projects are at risk and require immediate attention").

##### Most Overdue Projects Logic:
1. Query `projects` table joined with `statuses` where `status_name = 'At Risk'` and `is_active = 1`.
2. Filter by specialization if the query parameter is provided (via `devsecops_tickets` → `specialization_id`).
3. Calculate `daysOverdue` as the difference between the current date and the expected completion date (derived from `onboarded_date` + threshold).
4. Order by `daysOverdue` descending.
5. Limit results to 5.

##### Acceptance Criteria:
- The attention banner is included in the overview response with correct counts and severity.
- The most overdue projects list contains up to 5 projects sorted by overdue duration.
- Each overdue project entry includes project name, client, onboarded date, days overdue, and specialization.
- When no projects are at risk, the banner severity is `info` and the overdue list is empty.
- The banner message dynamically reflects the current at-risk count.

##### <u> 1.2.4 ZDAD-34-FR04: Query Parameter Validation </u>

##### Description:
The system shall validate all query parameters received by the Overview API endpoint. Invalid parameter values must result in a 400 Bad Request response with a descriptive error message indicating which parameter is invalid and what values are accepted.

##### Validation Rules:
- `period` must be one of: `last_week`, `last_month`, `last_3_months`. Any other value returns 400.
- `specialization` values are validated against existing specialization IDs in the database. Invalid IDs are ignored (non-strict filtering).

##### Error Response Format:
```json
{
  "status_code": 400,
  "status": "failed",
  "message": "Invalid query parameter: 'period' must be one of [last_week, last_month, last_3_months]",
  "data": []
}
```

##### Acceptance Criteria:
- Invalid `period` values return HTTP 400 with a descriptive error message.
- The error message clearly identifies the invalid parameter and accepted values.
- Valid requests with unknown specialization IDs do not fail but return filtered results (graceful handling).
- Response follows the standardized error response schema.
- Both endpoints handle errors like database unavailability and invalid requests, returning meaningful responses.

##### <u> 1.2.5 ZDAD-34-FR05: Error Logging </u>

##### Description:
The system shall log all unhandled exceptions and application errors to the `error_log` database table. This provides a persistent audit trail for debugging and incident investigation. Errors are logged with the function name, file name, error message, and full stack trace.

##### Error Log Table Fields:
- `error_id` — Auto-generated UUID primary key.
- `error_message` — The exception message text.
- `error_function` — The function name where the error occurred.
- `error_file` — The file path where the error originated.
- `stack_trace` — Full Python stack trace for debugging.
- `created_at` — Timestamp when the error was logged.
- `created_by` — System identifier (e.g., "overview_service").

##### Acceptance Criteria:
- All unhandled exceptions in the Overview API and Specializations API are captured and logged to the `error_log` table.
- The error log entry contains the function name, file name, error message, and stack trace.
- Error logging does not interfere with the error response returned to the client (non-blocking).
- HTTP 500 responses are always accompanied by an `error_log` entry.

#### <u> 1.3 Database Schema </u>

The following tables are directly involved in the Overview Dashboard API (as defined in `design/er_diagram.mmd`):

##### kpi_history
| Column | Type | Constraints |
|--------|------|-------------|
| kpi_history_id | UUID | PK |
| specialization_id | UUID | FK → specializations.specialization_id |
| projects_count | INTEGER | DEFAULT 0 |
| projects_increase_count | INTEGER | DEFAULT 0 |
| projects_decrease_count | INTEGER | DEFAULT 0 |
| completed_count | INTEGER | DEFAULT 0 |
| completed_increase_count | INTEGER | DEFAULT 0 |
| completed_decrease_count | INTEGER | DEFAULT 0 |
| inactive_count | INTEGER | DEFAULT 0 |
| inactive_increase_count | INTEGER | DEFAULT 0 |
| inactive_decrease_count | INTEGER | DEFAULT 0 |
| at_risk_count | INTEGER | DEFAULT 0 |
| at_risk_increase_count | INTEGER | DEFAULT 0 |
| at_risk_decrease_count | INTEGER | DEFAULT 0 |
| not_applicable_count | INTEGER | DEFAULT 0 |
| not_applicable_increase_count | INTEGER | DEFAULT 0 |
| not_applicable_decrease_count | INTEGER | DEFAULT 0 |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### specializations
| Column | Type | Constraints |
|--------|------|-------------|
| specialization_id | UUID | PK |
| specialization_name | VARCHAR | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### statuses
| Column | Type | Constraints |
|--------|------|-------------|
| status_id | UUID | PK |
| status_name | VARCHAR | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### projects
| Column | Type | Constraints |
|--------|------|-------------|
| project_id | UUID | PK |
| status_id | UUID | FK → statuses.status_id |
| sn_project_id | VARCHAR | ServiceNow project identifier |
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
| requested_at | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### settings (read-only reference for at_risk_threshold)
| Column | Type | Constraints |
|--------|------|-------------|
| setting_id | UUID | PK |
| specialization_id | UUID | FK → specializations.specialization_id |
| at_risk_threshold | INTEGER | NOT NULL |
| email_digest | VARCHAR | |
| at_risk_alert | VARCHAR | |
| last_synced | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
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
- `devsecops_tickets.specialization_id` → `specializations.specialization_id` (links projects to specializations)
- `devsecops_tickets.sn_project_id` → `projects.sn_project_id` (links tickets to projects)
- `kpi_history.specialization_id` → `specializations.specialization_id` (KPI data per specialization)
- `settings.specialization_id` → `specializations.specialization_id` (threshold config per specialization)


#### <u> 1.4 Project Artifacts </u>

- `api/openapi.yaml` — Full OpenAPI 3.0.3 specification defining all endpoints, request/response schemas, and error formats.
- `design/er_diagram.mmd` — Mermaid ER diagram showing all database tables and relationships.
- `design/requirements.md` — Section 1 (Overview API) contains the high-level requirements for this story.
- `diagram/ZDAD-34_overview_api_flow.mmd` — Sequence diagram showing the Overview API request flow.

#### <u> 1.5 Dependencies </u>

- **Python 3.12+** — Runtime environment
- **FastAPI** — Web framework for building the REST API
- **SQLAlchemy** — ORM for PostgreSQL database access
- **Pydantic v2** — Request/response model validation and serialization
- **PostgreSQL** — Primary database storing projects, KPI history, specializations, and error logs
- **python-jose / PyJWT** — JWT token decoding and validation
- **Uvicorn** — ASGI server for running the FastAPI application
- **ADO Sync API** — External dependency that populates the `kpi_history` table (out of scope)


### <u> Section 2: Non Functional Requirements </u>

### 2.1 Infrastructure and Deployment

#### <u> 2.1.1 Overview </u>

The Overview Dashboard API is deployed as part of the DevSecOps Jira Dashboard backend application on Azure App Service. The application is a Python FastAPI service running on Uvicorn, connected to a PostgreSQL database for persistent storage. The deployment follows a containerized approach with the application packaged as a Docker image and deployed to Azure App Service. The infrastructure leverages Azure-managed services for database hosting (Azure Database for PostgreSQL), application hosting (Azure App Service), and secret management. The deployment pipeline ensures zero-downtime deployments with health check validation before traffic routing. Environment-specific configurations are managed through Azure App Service application settings and environment variables, ensuring no secrets are hardcoded in the application code.

#### <u> 2.1.2 Requirement Details </u>

- **ZDAD-34-NFR01: Azure App Service Deployment**
- **ZDAD-34-NFR02: Environment Configuration**
- **ZDAD-34-NFR03: Health Check Endpoint**

##### <u> 2.1.2.1 ZDAD-34-NFR01: Azure App Service Deployment </u>

##### Description:
The application shall be deployed to Azure App Service as a containerized Python FastAPI application. The deployment uses a Docker image built from the project's Dockerfile and pushed to a container registry. Azure App Service is configured to pull the latest image and run the application with Uvicorn as the ASGI server.

##### Deployment Configuration:
- **Runtime:** Python 3.12+ container image
- **ASGI Server:** Uvicorn with configurable workers
- **Port:** Application listens on port 8080 (configurable via environment variable)
- **Startup Command:** `uvicorn main:app --host 0.0.0.0 --port 8080`

##### Acceptance Criteria:
- The application starts successfully on Azure App Service without errors.
- The health check endpoint responds within the configured startup timeout.
- Environment variables are correctly loaded from Azure App Service configuration.
- The application connects to PostgreSQL using connection string from environment variables.

##### <u> 2.1.2.2 ZDAD-34-NFR02: Environment Configuration </u>

##### Description:
All application configuration shall be managed through environment variables. Sensitive values such as database connection strings and JWT secrets are stored in Azure App Service application settings. A `.env.sample` file documents all required environment variables for local development.

##### Required Environment Variables:
| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | PostgreSQL connection string | Yes |
| `JWT_SECRET_KEY` | Secret key for JWT token validation | Yes |
| `JWT_ALGORITHM` | Algorithm used for JWT (default: HS256) | No |
| `APP_ENV` | Environment identifier (development, staging, production) | Yes |
| `LOG_LEVEL` | Application log level (default: INFO) | No |
| `PORT` | Application port (default: 8080) | No |

##### Acceptance Criteria:
- The application fails to start with a clear error message if required environment variables are missing.
- No secrets or credentials are hardcoded in the application source code.
- A `.env.sample` file exists documenting all required variables with placeholder values.

##### <u> 2.1.2.3 ZDAD-34-NFR03: Health Check Endpoint </u>

##### Description:
The application shall expose a `/health` endpoint for Azure App Service liveness probes and a `/ready` endpoint for readiness checks. The readiness endpoint validates database connectivity before reporting the application as ready to serve traffic.

##### Acceptance Criteria:
- `GET /health` returns HTTP 200 with `{"status": "healthy"}` when the application process is running.
- `GET /ready` returns HTTP 200 when the database connection is active and responsive.
- `GET /ready` returns HTTP 503 when the database is unreachable.
- Both endpoints do not require authentication.

#### <u> 2.1.3 Project Artifacts </u>

- `api/openapi.yaml` — API specification including health and readiness endpoints
- `design/er_diagram.mmd` — Database schema reference for readiness check validation

### 2.2 Architecture and System Design

#### <u> 2.2.1 Security and Compliance </u>

##### JWT Authentication:
All API endpoints (except `/health` and `/ready`) require a valid JWT Bearer token in the `Authorization` header. The middleware validates the token signature, expiration, and required claims before allowing the request to proceed to the route handler. Invalid or expired tokens result in an HTTP 401 Unauthorized response.

##### Token Validation Flow:
1. Extract the `Authorization` header from the incoming request.
2. Verify the header contains a `Bearer` prefix followed by the token.
3. Decode and validate the JWT token using the configured secret key and algorithm.
4. Check token expiration (`exp` claim) — reject if expired.
5. Attach decoded token claims to the request context for downstream use.
6. If validation fails at any step, return HTTP 401 with standardized error response.

##### Input Validation:
- All query parameters are validated using Pydantic models with strict enum constraints.
- SQL injection is prevented by using SQLAlchemy ORM with parameterized queries (no raw SQL).
- Request payloads are validated against Pydantic schemas before processing.

#### <u> 2.2.2 System Performance </u>

##### Database Query Optimization:
- The Overview API reads pre-computed KPI data from the `kpi_history` table, avoiding expensive aggregation queries at request time.
- Database indexes are maintained on frequently queried columns (`specialization_id`, `is_active`, `status_id`).
- SQLAlchemy connection pooling is configured to reuse database connections efficiently.
- The most overdue projects query uses indexed `status_id` and `onboarded_date` columns for efficient sorting.

##### Response Efficiency:
- The API returns all overview data (KPI tiles, status distribution, attention banner, overdue projects) in a single response to minimize round trips.
- Status distribution is computed with a single aggregation query on the `projects` table.
- The specializations endpoint response is cacheable with a TTL of 1 hour.

#### <u> 2.2.3 Availability and Reliability </u>

##### Error Resilience:
- All unhandled exceptions are caught by a global exception handler that returns HTTP 500 and logs the error to the `error_log` table.
- Database connection failures are handled gracefully with appropriate error responses.
- The application implements connection retry logic for transient database failures.

##### Deployment Reliability:
- Azure App Service provides built-in auto-restart on application crashes.
- Health check endpoints enable Azure to detect and replace unhealthy instances.

#### <u> 2.2.4 Cost Efficiency </u>

##### Resource Optimization:
- Pre-computed KPI metrics in `kpi_history` reduce database CPU usage compared to on-the-fly aggregation.
- Azure App Service scaling is configured based on actual traffic patterns.
- PostgreSQL connection pooling minimizes the number of active database connections.
- Single consolidated response reduces network overhead and client-side complexity.

#### <u> 2.2.5 Traceability and Observability </u>

##### Structured Logging:
- All application logs use JSON-formatted structured logging with fields: `timestamp`, `level`, `logger`, `filename`, `line_number`, `message`.
- Each request is assigned a `trace_id` for end-to-end request tracing.
- Log levels: DEBUG for development, INFO for production request/response logging, ERROR for exceptions.

##### Error Persistence:
- All application errors are persisted to the `error_log` database table with full context (function name, file name, stack trace).
- Error log entries include `created_at` timestamp and `created_by` identifier for audit purposes.

##### Request Logging:
- Incoming requests are logged at INFO level with method, path, and query parameters.
- Response status codes and latency are logged for monitoring purposes.
- Sensitive data (JWT tokens, credentials) is never included in log output.



### <u> Section 3: In Scope and Out Scope </u>

#### <u> 3.1 Inscope Details </u>

- Implementation of `GET /api/v1/overview` endpoint returning KPI tiles (Total Projects, Adopted, Adoption Rate, At Risk, Active, Inactive), status distribution, attention banner, and most overdue projects list in one response
- Implementation of `GET /api/v1/specializations` endpoint returning specialization values for the dropdown filter
- JWT Bearer token authentication middleware for both endpoints
- Query parameter validation for `period` (enum) and `specialization` (CSV) filters
- Reading pre-computed KPI data from the `kpi_history` table
- Computing status distribution from the `projects` and `statuses` tables
- Computing attention banner with severity levels based on at-risk project count
- Querying and returning the top 5 most overdue projects sorted by days overdue
- Standardized response format following `BaseResponse` schema (`status_code`, `status`, `message`, `data`)
- Error handling with HTTP 400, 401, and 500 responses with meaningful messages
- Error logging to the `error_log` database table for all unhandled exceptions
- Health check (`/health`) and readiness (`/ready`) endpoints
- SQLAlchemy ORM models for `kpi_history`, `specializations`, `projects`, `statuses`, `devsecops_tickets`, `settings`, and `error_log` tables
- Pydantic v2 request/response models for validation and serialization
- Structured JSON logging with trace_id for request tracing
- Environment variable-based configuration with validation at startup
- Layered architecture: Routes → Services → Repositories → Data Store
- Unit tests covering positive and negative cases for both endpoints

#### <u> 3.2 Outscope Details </u>

- `GET /api/v1/projects` — Projects list endpoint (separate story)
- `POST /api/v1/projects/action` — Project action endpoint (mark_not_applicable, mark_complete)
- `GET /api/v1/repositories/{repository_id}` — Repository detail endpoint
- `POST /api/v1/sync/servicenow` — ServiceNow sync endpoints (ZDAD-58)
- `POST /api/v1/reports/generate` — Email notification service (ZDAD-60)
- ADO Sync API that populates the `kpi_history` table
- KPI metric computation logic (handled by ADO Sync API)
- UI/Frontend implementation
- Database schema creation and migrations (assumed pre-existing)
- Caching layer implementation (Redis or in-memory)
- Rate limiting and throttling
- Role-based access control (RBAC) beyond JWT authentication
- Email notifications and alert system
- Settings management endpoints
- Client and status filter endpoints (consolidated into specializations only for this story)

### <u> Section 4: Solution Diagrams </u>

#### <u> 4.1 UI/UX Design Diagram </u>

**Diagram Location:** Not applicable for this story (backend API only)

#### <u> 4.2 Architecture Design Diagram </u>

**Diagram Location:** `diagram/ZDAD-34_overview_api_flow.mmd`

#### <u> 4.3 Infrastructure Design Diagram </u>

**Diagram Location:** Not applicable for this story
