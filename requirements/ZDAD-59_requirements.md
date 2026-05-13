# Requirement: Configure and Deliver Notification Alerts for DevSecOps Dashboard

## Project: DevSecOps
## Story Name: Configure and Deliver Notification Alerts for DevSecOps Dashboard
## Story Id: ZDAD-59

---

## 1. Overview

The DevSecOps Dashboard Settings module allows users to configure notification alert settings per specialization. This includes:

- Configuring the "at risk" threshold (number of days since onboarding without a repository before a project is considered at risk).
- Enabling/disabling email digest reports and at-risk alerts.
- Setting the report frequency (daily, weekly, monthly).
- Managing email recipients (add/remove) who receive the scheduled reports.

All errors encountered during service execution must be logged to the `error_log` table in the database.

Stack: Python

---

## 2. Scope

### In Scope

- Retrieve settings for a given specialization (including specialization name and email recipients).
- Update settings partially (only changed fields are sent).
- Manage email recipients via action-based approach (add/remove).
- Log all service errors to the `error_log` table.
- User authentication/authorization implementation (assumes JWT middleware is in place).

### Out of Scope

- Report generation and email delivery logic (covered in a separate story).
- Azure DevOps sync operations.
- Cron job tracking for email delivery schedules.

---

## 3. Technical Details

### 3.1 Database Details

Refer the er_diagram.mmd file.

**Required Tables:**

| Table              | Purpose                                                                 |
|--------------------|-------------------------------------------------------------------------|
| `specializations`  | Stores specialization records (id, name, active status)                 |
| `settings`         | Stores per-specialization settings (at_risk_threshold, email_digest, at_risk_alert) |
| `email_recipient`  | Stores email recipients linked to a setting and specialization          |
| `error_log`        | Logs service errors (error_message, error_function, error_file, stack_trace) |
| `cron_jobs`        | Tracks scheduled sync/email jobs per specialization (type, sync_status) |

### 3.2 API Endpoints

#### Service 1: SettingDetails (Subtask: ZDAD-73)

| Method | Path                                                  | Description                          |
|--------|-------------------------------------------------------|--------------------------------------|
| GET    | `/api/v1/specializations/{specializationId}/settings` | Retrieve settings for a specialization |

**Path Parameters:**
- `specializationId` (UUID, required) — UUID of the specialization

**Response (200):**
```json
{
  "status_code": 200,
  "status": "success",
  "message": "Settings retrieved successfully",
  "data": {
    "setting_id": "uuid",
    "specialization_id": "uuid",
    "specialization_name": "Cloud Engineering",
    "at_risk_threshold": 10,
    "email_digest": "weekly",
    "at_risk_alert": "daily",
    "report_frequency": "weekly",
    "last_synced": "2026-05-10T14:30:00Z",
    "email_recipients": [
      {
        "email_recipient_id": "uuid",
        "alert_recipient": "manager@org.com"
      }
    ]
  }
}
```

**Error Responses:**
- `404` — Specialization not found
- `500` — Server error (logged to `error_log`)

---

#### Service 2: ManageSettings (Subtask: ZDAD-73)

| Method | Path                                        | Description                              |
|--------|---------------------------------------------|------------------------------------------|
| PUT    | `/api/v1/specializations/settings/manage`   | Partially update settings for a specialization |

**Request Body:**
```json
{
  "specialization_id": "uuid (required)",
  "at_risk_threshold": 10,
  "email_digest": "weekly",
  "at_risk_alert": "daily",
  "report_frequency": "weekly",
  "email_recipients": [
    {
      "action": "add",
      "alert_recipient": "newuser@org.com"
    },
    {
      "action": "remove",
      "email_recipient_id": "uuid"
    }
  ]
}
```

**Field Details:**
- `specialization_id` — Required. UUID of the specialization to update.
- `at_risk_threshold` — Optional. Integer (min: 1). Days threshold for at-risk determination.
- `email_digest` — Optional. String. Email digest schedule value (e.g., "weekly", "daily", "monthly").
- `at_risk_alert` — Optional. String. At-risk alert schedule value (e.g., "weekly", "daily", "monthly").
- `report_frequency` — Optional. Enum: `daily`, `weekly`, `monthly`.
- `email_recipients` — Optional. Array of actions:
  - `add`: requires `alert_recipient` (email address)
  - `remove`: requires `email_recipient_id` (UUID)

**Response (200):**
Returns the updated `SettingsSuccessResponse` (same structure as GET, includes `last_synced`).

**Error Responses:**
- `400` — Validation error (invalid fields or values)
- `404` — Specialization not found
- `500` — Server error (logged to `error_log`)

### 3.3 Affected Components

| Layer        | File/Module                          | Change Description                                                              |
|--------------|--------------------------------------|---------------------------------------------------------------------------------|
| Route        | `src/routes/settings_route.py`       | Define GET and PUT endpoints for settings                                       |
| Service      | `src/services/settings_service.py`   | Business logic for retrieving and updating settings, email recipient management |
| Repository   | `src/repositories/settings_repository.py` | Data access for `settings`, `email_recipient`, and `specializations` tables |
| Schema       | `src/repositories/schema/settings.py` | SQLAlchemy ORM models for `settings` and `email_recipient`                     |
| Schema       | `src/repositories/schema/error_log.py` | SQLAlchemy ORM model for `error_log`                                          |
| Schema       | `src/repositories/schema/cron_jobs.py` | SQLAlchemy ORM model for `cron_jobs`                                          |
| Model        | `src/models/settings_model.py`       | Pydantic request/response models for settings                                   |
| Utility      | `src/utils/error_logger.py`          | Utility to log errors to the `error_log` table                                  |

---

## 4. Acceptance Criteria

### AC-1: Retrieve Settings
- **Given** a valid `specializationId` exists in the database,
- **When** a GET request is made to `/api/v1/specializations/{specializationId}/settings`,
- **Then** the API returns the settings record including `at_risk_threshold`, `email_digest`, `at_risk_alert`, `report_frequency`, specialization name, and the list of email recipients.

### AC-2: Specialization Not Found
- **Given** the `specializationId` does not exist,
- **When** a GET or PUT request is made,
- **Then** the API returns a `404` response with message "Specialization not found".

### AC-3: Update At-Risk Threshold
- **Given** a valid specialization with existing settings,
- **When** a PUT request is made with `at_risk_threshold` set to a new value,
- **Then** the `settings.at_risk_threshold` is updated and the response reflects the new value.

### AC-4: Update Email Digest and At-Risk Alert
- **Given** a valid specialization with existing settings,
- **When** a PUT request is made with `email_digest` or `at_risk_alert` set to a new string value,
- **Then** the corresponding field is updated in the `settings` table.

### AC-5: Update Report Frequency
- **Given** a valid specialization with existing settings,
- **When** a PUT request is made with `report_frequency` set to `daily`, `weekly`, or `monthly`,
- **Then** the `settings.email_digest` field is updated accordingly.

### AC-6: Add Email Recipient
- **Given** a valid specialization,
- **When** a PUT request includes `email_recipients` with action `add` and a valid email,
- **Then** a new record is inserted into `email_recipient` table linked to the setting and specialization.

### AC-7: Remove Email Recipient
- **Given** a valid specialization with existing email recipients,
- **When** a PUT request includes `email_recipients` with action `remove` and a valid `email_recipient_id`,
- **Then** the corresponding `email_recipient` record is soft-deleted (is_active = 0).

### AC-8: Error Logging
- **Given** any unhandled exception occurs during service execution,
- **When** the error is caught,
- **Then** the error details (message, function name, file name, stack trace) are logged to the `error_log` table.

### AC-9: Validation Error
- **Given** an invalid request body (e.g., `at_risk_threshold` < 1, invalid email format, invalid `report_frequency`),
- **When** a PUT request is made,
- **Then** the API returns a `400` response with a descriptive validation error message.

---

## 5. UI Reference

Refer to the attached "Settings - UI screenshots.docx" for the visual layout of:
- Settings configuration panel (at-risk threshold input, email digest toggle, at-risk alert toggle, report frequency selector)
- Email recipients management (add/remove recipients list)

---

## 6. Testing Requirements

- GET endpoint returns correct settings with email recipients for a valid specialization.
- GET endpoint returns 404 for non-existent specialization.
- PUT endpoint updates individual fields without affecting others (partial update).
- PUT endpoint adds new email recipients correctly.
- PUT endpoint removes email recipients via soft delete.
- PUT endpoint returns 400 for invalid input values.
- Error logging captures exceptions with full context (function, file, stack trace).
- Cron job records are created/updated for scheduled email deliveries.

---

## 7. Assumptions

- JWT authentication middleware is already in place and validates the Bearer token.
- Each specialization has at most one `settings` record (1:1 relationship).
- Email recipient removal is a soft delete (`is_active = 0`), not a hard delete.
- The `report_frequency` field is stored in the `settings` table as part of the `email_digest` column or a related mechanism.
- The `cron_jobs` table is used to track the status of scheduled email delivery jobs.

---

## 8. Dependencies

- Database with the required tables (`specializations`, `settings`, `email_recipient`, `error_log`, `cron_jobs`) already provisioned.
- JWT authentication middleware for route protection.
- OpenAPI specification (`api/openapi.yaml`) for contract validation.
- ER diagram (`design/er_diagram.mmd`) for schema reference.
