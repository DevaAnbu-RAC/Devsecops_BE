# Requirements Document

## Introduction

This document defines the requirements for a Repository Detail API endpoint that returns comprehensive information about a specific repository within the DevSecOps Jira Dashboard. The endpoint consolidates all repository-related data into a single response, including header metadata, pipeline metrics, adoption timeline, pipeline activity, commits, pull requests, security scan findings, and build artifacts.

## Glossary

- **API**: The DevSecOps Jira Dashboard backend REST API
- **Repository**: A source code repository tracked within the DevSecOps onboarding system
- **Pipeline**: A CI/CD pipeline associated with a repository in Azure DevOps (ADO)
- **Adoption_Timeline**: A multi-step progression tracking a repository's onboarding journey from kickoff through adoption
- **Pipeline_Activity**: A historical record of pipeline execution runs for a repository
- **Security_Scan**: An automated security analysis (SCA, SAST, or DAST) performed against a repository
- **Artifact**: A build output file produced by a pipeline run
- **SCA**: Software Composition Analysis — scanning for vulnerabilities in third-party dependencies
- **SAST**: Static Application Security Testing — scanning source code for security flaws
- **DAST**: Dynamic Application Security Testing — scanning running applications for vulnerabilities
- **BaseResponse**: The standard API response envelope containing status, message, and data fields

## Requirements

### Requirement 1: Retrieve Repository Detail

**User Story:** As a dashboard user, I want to retrieve all detail information for a specific repository in a single API call, so that I can view the complete repository status without multiple requests.

#### Acceptance Criteria

1. WHEN a GET request is made to `/api/v1/repositories/{repositoryId}`, THE API SHALL return the complete repository detail response wrapped in the BaseResponse envelope.
2. THE API SHALL identify the repository using a UUID path parameter named `repositoryId`.
3. IF the `repositoryId` does not correspond to an existing repository, THEN THE API SHALL return a 404 status code with a descriptive error message.
4. IF the `repositoryId` is not a valid UUID format, THEN THE API SHALL return a 400 status code with a validation error message.
5. IF the request lacks a valid authentication token, THEN THE API SHALL return a 401 status code with an unauthorized error message.

### Requirement 2: Repository Header Information

**User Story:** As a dashboard user, I want to see the repository's metadata and status tags, so that I can quickly understand the repository's identity and current state.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include the repository name in the response.
2. WHEN the repository detail is retrieved, THE API SHALL include an array of status tags associated with the repository (e.g., "PASSED", "ADO Linked"). Overdue information SHALL NOT be included in the tags array.
3. WHEN the repository detail is retrieved, THE API SHALL include an `overdue` field as a nullable integer representing the number of days the repository is overdue. THE field SHALL be null when the repository is not overdue.
4. WHEN the repository detail is retrieved, THE API SHALL include the client name associated with the repository.
5. WHEN the repository detail is retrieved, THE API SHALL include the onboarding date as an ISO 8601 date string.
6. WHEN the repository detail is retrieved, THE API SHALL include the expected completion date (closed by) as an ISO 8601 date string or null if not set.
7. WHEN the repository detail is retrieved, THE API SHALL include the project type label describing the specialization and category.
8. WHEN the repository detail is retrieved, THE API SHALL include the parent project ID and name.

### Requirement 3: Pipeline Metrics

**User Story:** As a dashboard user, I want to see pipeline execution metrics for the repository, so that I can assess the CI/CD health and activity level.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include the total pipeline runs count since the first connection.
2. WHEN the repository detail is retrieved, THE API SHALL include the pipeline success rate as a percentage calculated from the last 30 pipeline runs.
3. WHEN the repository detail is retrieved, THE API SHALL include the time elapsed since the last pipeline run as a human-readable string.
4. WHEN the repository detail is retrieved, THE API SHALL include the ISO 8601 timestamp of the last pipeline run or null if no runs exist.

### Requirement 4: Adoption Timeline

**User Story:** As a dashboard user, I want to see the adoption timeline for the repository, so that I can track the onboarding progress through each milestone.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include an ordered array of adoption timeline steps.
2. THE API SHALL include exactly four timeline steps: "kicked_off", "pipeline_detected", "first_run", and "adopted".
3. WHEN a timeline step has been completed, THE API SHALL include the completion date as an ISO 8601 date string.
4. WHEN a timeline step has not been completed, THE API SHALL set the completion date to null.
5. THE API SHALL include a status field for each timeline step with a value of "completed", "in_progress", or "pending".
6. THE API SHALL include a description field for each timeline step providing contextual information about the step's state.

### Requirement 5: Pipeline Activity

**User Story:** As a dashboard user, I want to see the list of recent pipeline runs, so that I can review execution history and identify failures.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include a paginated list of pipeline runs sorted by triggered time in descending order.
2. THE API SHALL include for each pipeline run: run number, status, branch name, duration, and triggered timestamp.
3. THE API SHALL represent pipeline run status as one of: "passed", "failed", "running", or "cancelled".
4. THE API SHALL represent duration as a human-readable string.
5. THE API SHALL support pagination for pipeline activity using offset and limit query parameters.
6. THE API SHALL default to returning the 10 most recent pipeline runs when no pagination parameters are provided.

### Requirement 6: Commits

**User Story:** As a dashboard user, I want to see the list of recent commits, so that I can review development activity in the repository.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include a paginated list of commits sorted by commit time in descending order.
2. THE API SHALL include for each commit: abbreviated commit hash, full commit message, author name, and commit timestamp.
3. THE API SHALL support pagination for commits using offset and limit query parameters.
4. THE API SHALL default to returning the 10 most recent commits when no pagination parameters are provided.

### Requirement 7: Pull Requests

**User Story:** As a dashboard user, I want to see pull request activity and summary counts, so that I can understand the code review workflow for the repository.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include summary counts of pull requests grouped by status: open, merged, and declined.
2. WHEN the repository detail is retrieved, THE API SHALL include a paginated list of pull requests sorted by updated time in descending order.
3. THE API SHALL include for each pull request: title, author name, status, and last updated timestamp.
4. THE API SHALL represent pull request status as one of: "open", "merged", or "declined".
5. THE API SHALL support pagination for pull requests using offset and limit query parameters.
6. THE API SHALL default to returning the 10 most recent pull requests when no pagination parameters are provided.

### Requirement 8: Security Scans

**User Story:** As a dashboard user, I want to see security scan findings for the repository, so that I can assess the security posture and identify vulnerabilities.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include a summary object with finding counts for each scan type: SCA, SAST, and DAST.
2. WHEN the repository detail is retrieved, THE API SHALL include the total findings count across all scan types.
3. THE API SHALL represent each scan type count as a non-negative integer.

### Requirement 9: Artifacts

**User Story:** As a dashboard user, I want to see build artifacts produced by the repository's pipelines, so that I can access and download build outputs.

#### Acceptance Criteria

1. WHEN the repository detail is retrieved, THE API SHALL include a paginated list of artifacts sorted by creation date in descending order.
2. THE API SHALL include for each artifact: filename, file size in bytes, associated build number, creation date, and download URL.
3. THE API SHALL support pagination for artifacts using offset and limit query parameters.
4. THE API SHALL default to returning the 10 most recent artifacts when no pagination parameters are provided.

### Requirement 10: Tab-Based Query Parameters

**User Story:** As a frontend developer, I want to request only the data for the active tab, so that the API can optimize response size and query performance.

#### Acceptance Criteria

1. THE API SHALL accept an optional `tab` query parameter to specify which detail section to include in the response.
2. THE API SHALL accept the following values for the `tab` parameter: "pipeline_activity", "commits", "pull_requests", "security_scans", and "artifacts".
3. WHEN the `tab` parameter is not provided, THE API SHALL include all sections in the response.
4. WHEN the `tab` parameter is provided, THE API SHALL include only the header, pipeline metrics, adoption timeline, and the specified tab's data in the response.
5. IF the `tab` parameter contains an invalid value, THEN THE API SHALL return a 400 status code with a validation error message.
