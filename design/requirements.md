# DevSecOps Jira Dashboard - Requirements

## 1. Overview API

### 1.1 Description

The Overview API provides dashboard-level KPI metrics and status distribution for all onboarded projects. It serves as the landing page data source, giving stakeholders a quick snapshot of project health across the organization.

### 1.2 Endpoint

`GET /api/v1/overview`

### 1.3 Filters

| Filter | Parameter | Type | Required | Default | Description |
|--------|-----------|------|----------|---------|-------------|
| Period | `period` | string (enum) | No | `last_month` | Time period for metric calculations |
| Specialization | `specialization` | string (CSV) | No | All | Comma-separated specialization IDs to filter by |

#### Period Options

- `last_week` — Last 7 days
- `last_month` — Last 30 days
- `last_3_months` — Last 90 days
- `last_6_months` — Last 180 days
- `last_year` — Last 365 days

#### Specialization Filter

- Accepts a comma-separated list of specialization IDs (e.g., `devsecops,devops`)
- When empty or omitted, returns metrics across all specializations
- Valid IDs are retrieved from `GET /api/v1/specializations`

### 1.4 Response — KPI Metrics

The API returns the following KPI cards:

| KPI | Field | Description |
|-----|-------|-------------|
| Total Projects | `metrics.totalProjects` | Total number of projects in the system |
| Completed | `metrics.completed` | Projects that are fully onboarded |
| Active | `metrics.active` | Projects with pipeline activity within the last 10 days |
| Inactive | `metrics.inactive` | Projects with no pipeline activity for 10+ days |
| At Risk | `metrics.atRisk` | Projects with overdue onboarding |
| Not Applicable | `metrics.notApplicable` | Projects marked as exceptions |

Each KPI includes:

- `count` — Current value for the selected period
- `trend` — Direction compared to previous period (`increase`, `decrease`, `flat`, or `null`)
- `change` — Numeric difference from the previous period
- `description` or `source` — Contextual label explaining the metric

### 1.5 Response — Status Distribution

The API also returns a status distribution breakdown:

- `statusDistribution.total` — Total project count
- `statusDistribution.breakdown[]` — Array of status groups, each with:
  - `status` — Status name (Completed, Active, Inactive, At Risk, Not Applicable)
  - `count` — Number of projects in that status
  - `percentage` — Percentage of total projects

### 1.6 Error Responses

| Status Code | Condition |
|-------------|-----------|
| 400 | Invalid query parameter value (e.g., unsupported period) |
| 401 | Missing or invalid authentication token |
| 500 | Unexpected server error |

### 1.7 Authentication

- Requires a valid JWT Bearer token in the `Authorization` header
- Returns 401 if the token is missing, expired, or invalid
