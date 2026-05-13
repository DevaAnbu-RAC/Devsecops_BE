## SOLDEF-ZDAD-34 - Overview API

### <u>Project Details</u>
- **Project ID:** ZDAD  
- **Project Name:** DevSecOps Jira Dashboard  

### <u>Story Details</u>
- **Story ID:** ZDAD-34  
- **Story Name:** Overview API  
- **Story Description:**  
  Provide real-time dashboard metrics for stakeholders to monitor project health. The Overview API serves as the landing page data source, delivering KPI metrics and status distribution for all onboarded projects filtered by period and specialization.

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

The Overview API is the primary data source for the DevSecOps Jira Dashboard landing page. It provides stakeholders with a consolidated view of project health across the organization by delivering KPI metrics and status distribution data. The API exposes two endpoints: `GET /api/v1/overview` for retrieving dashboard metrics and `GET /api/v1/specializations` for populating the specialization filter dropdown. The KPI metrics include Total Projects, Completed, Active, Inactive, At Risk, and Not Applicable counts along with trend indicators comparing current values against the previous period. The status distribution provides a percentage-based breakdown of projects across all statuses. Both endpoints require JWT Bearer token authentication and return standardized JSON responses. The KPI data is pre-computed and stored in the `kpi_history` table, which is populated by the ADO Sync API, ensuring the Overview API performs efficient read operations without complex on-the-fly calculations. All errors encountered during request processing are logged to the `error_log` database table for traceability and debugging purposes.

#### <u> 1.2 Requirement Details </u>

- **ZDAD-34-FR01: Overview Dashboard Metrics Retrieval**
- **ZDAD-34-FR02: Specializations Filter Endpoint**
- **ZDAD-34-FR03: Query Parameter Validation**
- **ZDAD-34-FR04: Error Logging**

##### <u> 1.2.1 ZDAD-34-FR01: Overview Dashboard Metrics Retrieval </u>

##### Description:
The system shall expose a `GET /api/v1/overview` endpoint that returns KPI metrics and status distribution data for the dashboard landing page. The endpoint accepts optional query parameters for filtering by time period and specialization. The KPI data is read from the `kpi_history` table which is pre-computed by the ADO Sync API.

##### Request Parameters:
- `period` (optional, string enum): Defines the time window for metric calculations. Accepted values: `last_week` (7 days), `last_month` (30 days, default), `last_3_months` (90 days), `last_6_months` (180 days), `last_year` (365 days).
- `specialization` (optional, string CSV): Comma-separated list of specialization IDs to filter metrics by (e.g., `devsecops,devops`). When omitted, returns metrics across all specializations.

##### Response Structure - KPI Metrics:
The response `data.metrics` object contains six KPI cards:
- `totalProjects` — Total number of projects with `count`, `newThisMonth` (trend direction), `change` (numeric difference), and `source` (contextual label).
- `completed` — Fully onboarded projects with `count`, `description`, `trend`, and `change`.
- `active` — Projects with pipeline activity within last 10 days with `count`, `description` ("pipeline run ≤ 10d"), `trend`, and `change`.
- `inactive` — Projects with no pipeline activity for 10+ days with `count`, `description` ("no activity 10+ d"), `trend`, and `change`.
- `atRisk` — Projects with overdue onboarding with `count`, `description` ("overdue onboarding"), `trend`, and `change`.
- `notApplicable` — Projects marked as exceptions with `count`, `description` ("exception raised"), `trend`, and `change`.

##### Response Structure - Status Distribution:
The response `data.statusDistribution` object contains:
- `total` — Total project count.
- `breakdown[]` — Array of objects each with `status` (name), `count`, and `percentage` (float).

##### Data Source:
- KPI metrics are read from the `kpi_history` table joined with `specializations` table for filtering.
- Status distribution is computed from the `projects` table joined with `statuses` table.
- The `projects` table includes `sn_project_id` (ServiceNow project identifier) for cross-system traceability.
- The `kpi_history` table is populated and maintained by the ADO Sync API (out of scope for this story).

##### Acceptance Criteria:
- The endpoint returns HTTP 200 with correct KPI metrics and status distribution when called with valid parameters.
- Default period is `last_month` when no period parameter is provided.
- When specialization filter is provided, only metrics for matching specializations are returned.
- When specialization filter is omitted, metrics across all specializations are aggregated.
- Trend values are one of: `increase`, `decrease`, `flat`, or `null`.
- Response follows the standardized `BaseResponse` schema with `status`, `message`, and `data` fields.

##### <u> 1.2.2 ZDAD-34-FR02: Specializations Filter Endpoint </u>

##### Description:
The system shall expose a `GET /api/v1/specializations` endpoint that returns the list of available specializations for the filter dropdown on the dashboard. This endpoint provides the valid specialization IDs that can be used as filter values in the Overview API.

##### Response Structure:
The response `data.specializations` is an array of objects each containing:
- `id` — Unique identifier for the specialization (e.g., "devsecops", "devops").
- `name` — Display name of the specialization (e.g., "DevSecOps", "DevOps").

##### Data Source:
- Data is read from the `specializations` table where `is_active = 1`.

##### Acceptance Criteria:
- The endpoint returns HTTP 200 with a list of all active specializations.
- Each specialization object contains `id` and `name` fields.
- Only specializations with `is_active = 1` are returned.
- Response follows the standardized `BaseResponse` schema.
- The endpoint requires JWT Bearer token authentication.

##### <u> 1.2.3 ZDAD-34-FR03: Query Parameter Validation </u>

##### Description:
The system shall validate all query parameters received by the Overview API endpoint. Invalid parameter values must result in a 400 Bad Request response with a descriptive error message indicating which parameter is invalid and what values are accepted.

##### Validation Rules:
- `period` must be one of: `last_week`, `last_month`, `last_3_months`, `last_6_months`, `last_year`. Any other value returns 400.
- `specialization` values are validated against existing specialization IDs in the database. Invalid IDs are ignored (non-strict filtering).

##### Error Response Format:
```json
{
  "status": "failed",
  "message": "Invalid query parameter: 'period' must be one of [last_week, last_month, last_3_months, last_6_months, last_year]",
  "data": []
}
```

##### Acceptance Criteria:
- Invalid `period` values return HTTP 400 with a descriptive error message.
- The error message clearly identifies the invalid parameter and accepted values.
- Valid requests with unknown specialization IDs do not fail but return filtered results (graceful handling).
- Response follows the standardized error response schema.

##### <u> 1.2.4 ZDAD-34-FR04: Error Logging </u>

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

The following tables are defined in the system as per the ER diagram (`design/er_diagram.mmd`):

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

##### settings
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

##### email_recipient
| Column | Type | Constraints |
|--------|------|-------------|
| email_recipient_id | UUID | PK |
| setting_id | UUID | FK → settings.setting_id |
| specialization_id | UUID | FK → specializations.specialization_id |
| alert_recipient | VARCHAR | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### email_templates
| Column | Type | Constraints |
|--------|------|-------------|
| email_template_id | UUID | PK |
| template_name | VARCHAR | NOT NULL |
| template_content | TEXT | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### email_history
| Column | Type | Constraints |
|--------|------|-------------|
| email_history_id | UUID | PK |
| setting_id | UUID | FK → settings.setting_id |
| report_frequency | VARCHAR | NOT NULL |
| email_status | VARCHAR | NOT NULL |
| email_type | VARCHAR | NOT NULL |
| report_url | TEXT | |
| last_synced | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### jira_tickets
| Column | Type | Constraints |
|--------|------|-------------|
| jira_ticket_id | UUID | PK |
| project_id | UUID | FK → projects.project_id |
| jira_id | VARCHAR | NOT NULL |
| type | VARCHAR | NOT NULL |
| assignee | VARCHAR | NOT NULL |
| priority | VARCHAR | NOT NULL |
| status | VARCHAR | NOT NULL |
| reason_category | VARCHAR | NOT NULL |
| comments | TEXT | NOT NULL |
| evidence_url | TEXT | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### pipeline_runs
| Column | Type | Constraints |
|--------|------|-------------|
| pipeline_run_id | UUID | PK |
| repository_id | UUID | FK → repositories.repository_id |
| run_number | INTEGER | NOT NULL |
| status | VARCHAR | NOT NULL |
| branch | VARCHAR | NOT NULL |
| duration_seconds | INTEGER | |
| triggered_at | DATETIME | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### commits
| Column | Type | Constraints |
|--------|------|-------------|
| commit_id | UUID | PK |
| repository_id | UUID | FK → repositories.repository_id |
| hash | VARCHAR | NOT NULL |
| message | TEXT | NOT NULL |
| author | VARCHAR | NOT NULL |
| committed_at | DATETIME | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### pull_requests
| Column | Type | Constraints |
|--------|------|-------------|
| pull_request_id | UUID | PK |
| repository_id | UUID | FK → repositories.repository_id |
| title | VARCHAR | NOT NULL |
| author | VARCHAR | NOT NULL |
| status | VARCHAR | NOT NULL |
| updated_at | DATETIME | |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### security_scans
| Column | Type | Constraints |
|--------|------|-------------|
| security_scan_id | UUID | PK |
| repository_id | UUID | FK → repositories.repository_id |
| scan_type | VARCHAR | NOT NULL |
| findings_count | INTEGER | DEFAULT 0 |
| description | VARCHAR | |
| scanned_at | DATETIME | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### artifacts
| Column | Type | Constraints |
|--------|------|-------------|
| artifact_id | UUID | PK |
| pipeline_run_id | UUID | FK → pipeline_runs.pipeline_run_id |
| artifact_name | VARCHAR | NOT NULL |
| size_bytes | BIGINT | NOT NULL |
| url | TEXT | NOT NULL |
| uploaded_at | DATETIME | NOT NULL |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

##### cron_jobs
| Column | Type | Constraints |
|--------|------|-------------|
| cron_id | UUID | PK |
| specialization_id | VARCHAR | NOT NULL |
| type | VARCHAR | NOT NULL, ENUM(azure, email) |
| sync_status | VARCHAR | NOT NULL, ENUM(pending, success, fail) |
| created_at | DATETIME | |
| created_by | VARCHAR | |
| modified_at | DATETIME | |
| modified_by | VARCHAR | |
| is_active | INT | DEFAULT 1 |

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

#### <u> 1.4 Project Artifacts </u>

- `api/openapi.yaml` — Full OpenAPI 3.0.3 specification defining all endpoints, request/response schemas, and error formats.
- `design/er_diagram.mmd` — Mermaid ER diagram showing all database tables and relationships.
- `design/requirements.md` — Detailed requirements document for the Overview API.

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

The Overview API is deployed as part of the DevSecOps Jira Dashboard backend application on Azure App Service. The application is a Python FastAPI service running on Uvicorn, connected to a PostgreSQL database for persistent storage. The deployment follows a containerized approach with the application packaged as a Docker image and deployed to Azure App Service. The infrastructure leverages Azure-managed services for database hosting (Azure Database for PostgreSQL), application hosting (Azure App Service), and secret management. The deployment pipeline ensures zero-downtime deployments with health check validation before traffic routing. Environment-specific configurations are managed through Azure App Service application settings and environment variables, ensuring no secrets are hardcoded in the application code.

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
- `DATABASE_URL` — PostgreSQL connection string
- `JWT_SECRET_KEY` — Secret key for JWT token validation
- `JWT_ALGORITHM` — Algorithm used for JWT (default: HS256)
- `APP_ENV` — Environment identifier (development, staging, production)
- `LOG_LEVEL` — Application log level (default: INFO)
- `PORT` — Application port (default: 8080)

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

##### Response Efficiency:
- The API returns only the required fields in the response payload (no over-fetching).
- Status distribution is computed with a single aggregation query on the `projects` table.

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

- Implementation of `GET /api/v1/overview` endpoint returning KPI metrics and status distribution
- Implementation of `GET /api/v1/specializations` endpoint returning filter dropdown options
- JWT Bearer token authentication middleware for both endpoints
- Query parameter validation for `period` (enum) and `specialization` (CSV) filters
- Reading pre-computed KPI data from the `kpi_history` table
- Computing status distribution from the `projects` and `statuses` tables
- Standardized response format following `BaseResponse` schema (`status`, `message`, `data`)
- Error handling with HTTP 400, 401, and 500 responses
- Error logging to the `error_log` database table for all unhandled exceptions
- Health check (`/health`) and readiness (`/ready`) endpoints
- SQLAlchemy ORM models for `kpi_history`, `specializations`, `projects`, `statuses`, `error_log`, `devsecops_tickets`, `settings`, `email_recipient`, `email_history`, `email_templates`, and `cron_jobs` tables
- Pydantic v2 request/response models for validation and serialization
- Structured JSON logging with trace_id for request tracing
- Environment variable-based configuration with validation at startup
- Layered architecture: Routes → Middleware → Services → Repository → Database

#### <u> 3.2 Outscope Details </u>

- `GET /api/v1/projects` — Projects list endpoint (separate story)
- `POST /api/v1/projects` — Project action endpoint (mark_not_applicable, mark_complete)
- `GET /api/v1/repositories/{repository_id}` — Repository detail endpoint
- `POST /api/v1/sync/servicenow` — ServiceNow sync endpoint
- `GET /api/v1/clients` — Clients filter endpoint
- `GET /api/v1/statuses` — Statuses filter endpoint
- ADO Sync API that populates the `kpi_history` table
- KPI metric computation logic (handled by ADO Sync API)
- UI/Frontend implementation
- Database schema creation and migrations (assumed pre-existing)
- Caching layer implementation (Redis or in-memory)
- Rate limiting and throttling
- Role-based access control (RBAC) beyond JWT authentication
- Email notifications and alert system
- Settings management endpoints

### <u> Section 4: Solution Diagrams </u>

#### <u> 4.1 UI/UX Design Diagram </u>

**Diagram Location:** Not applicable for this story (backend API only)

#### <u> 4.2 Architecture Design Diagram </u>

**Diagram Location:** `design/er_diagram.mmd`

#### <u> 4.3 Infrastructure Design Diagram </u>

**Diagram Location:** Not applicable for this story
