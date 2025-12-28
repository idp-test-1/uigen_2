# GitHub Workflows for Pipeline Testing (Minimal Version)

This document contains specifications for 6 test workflows to validate pipeline execution scenarios.

## Overview

Each workflow simulates different execution patterns (success/failure, different durations) to test the pipeline execution engine's behavior with dependencies, trigger conditions, and recovery paths.

**Minimal Approach:** Workflows only report final status (SUCCESS/FAILURE) to the IDP API, keeping implementation simple.

---

## Workflow 1: quick-success.yml

**Purpose:** Quick successful steps (test-unit, notify)

**Duration:** 15 seconds

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 15 seconds
- At completion, authenticates with IDP API using client credentials (see API Call #1 below)
- Reports SUCCESS status with progress=100 using API Call #2 below
- Includes error handling: if any step fails, report FAILURE status in a final cleanup step

---

## Workflow 2: quick-fail.yml

**Purpose:** Fast failing steps (test-integration failure)

**Duration:** 15 seconds (fails at ~8 seconds)

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 8 seconds
- Intentionally fails with `exit 1` and error message "Quick failure occurred"
- In a final step that always runs (using `if: always()`), authenticates with IDP API (see API Call #1 below)
- Reports FAILURE status with error message using API Call #2 below

---

## Workflow 3: slow-success.yml

**Purpose:** Longer operations (deploy, rollback success)

**Duration:** 45 seconds

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 45 seconds
- At completion, authenticates with IDP API using client credentials (see API Call #1 below)
- Reports SUCCESS status with progress=100 using API Call #2 below
- Includes error handling: if any step fails, report FAILURE status in a final cleanup step

---

## Workflow 4: slow-fail.yml

**Purpose:** Deploy failures, rollback failures

**Duration:** 45 seconds (fails at ~30 seconds)

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 30 seconds
- Intentionally fails with `exit 1` and error message "Deployment failed: resource quota exceeded"
- In a final step that always runs (using `if: always()`), authenticates with IDP API (see API Call #1 below)
- Reports FAILURE status with error message using API Call #2 below

---

## Workflow 5: medium-success.yml

**Purpose:** Medium-duration operations

**Duration:** 30 seconds

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 30 seconds
- At completion, authenticates with IDP API using client credentials (see API Call #1 below)
- Reports SUCCESS status with progress=100 using API Call #2 below
- Includes error handling: if any step fails, report FAILURE status in a final cleanup step

---

## Workflow 6: medium-fail.yml

**Purpose:** Medium-duration failures

**Duration:** 30 seconds (fails at ~15 seconds)

### Prompt for Claude:

Create a GitHub Action workflow that:
- Accepts workflow inputs: `runId`, `stepId`, `client-id`, `client-secret`, `api-url`
- Simulates work by waiting 15 seconds
- Intentionally fails with `exit 1` and error message "Integration test failed"
- In a final step that always runs (using `if: always()`), authenticates with IDP API (see API Call #1 below)
- Reports FAILURE status with error message using API Call #2 below

---

## API Calls Reference

All workflows must make these two API calls to the IDP backend:

---

### API Call #1: Authenticate and Get Token

**Purpose:** Obtain a JWT token for authenticating subsequent API requests

**Specifications:**
- **Method:** POST
- **Endpoint:** `{api-url}/api/auth/token`
- **Headers:** `Content-Type: application/json`
- **Request Body:** JSON with `clientId` and `clientSecret`
- **Response:** JSON with `token` field containing the JWT
- **Implementation:** Store the token for use in API Call #2

---

### API Call #2: Update Step Status

**Purpose:** Report the final status of the workflow step back to IDP

**Specifications:**
- **Method:** PATCH
- **Endpoint:** `{api-url}/api/action-runs/{runId}/steps/{stepId}`
- **Headers:**
  - `Authorization: Bearer {token}` (from API Call #1)
  - `Content-Type: application/json`
- **Request Body for SUCCESS:** JSON with `status: "SUCCESS"` and `progress: 100`
- **Request Body for FAILURE:** JSON with `status: "FAILURE"` and `error: "error message"`
- **Implementation:** This call must be made in a final step that always runs (even on workflow failure)

---

### Workflow Inputs Required

All workflows must accept these inputs via `workflow_dispatch`:

| Input Name | Type | Required | Description |
|-----------|------|----------|-------------|
| `api-url` | string | Yes | Base URL of IDP API (example: `http://localhost:8080`) |
| `client-id` | string | Yes | Service account client ID for authentication |
| `client-secret` | string | Yes | Service account client secret for authentication |
| `runId` | string | Yes | Unique identifier for the pipeline execution run |
| `stepId` | string | Yes | Unique identifier for this specific step in the pipeline |

---

## Testing Scenarios

These workflows map to failure scenarios:

| Scenario | Workflows Used |
|----------|---------------|
| **Simple Failure (No Recovery)** | slow-fail → quick-success (skipped) |
| **Successful Recovery** | slow-fail → slow-success (recovery) |
| **Failed Recovery** | slow-fail → medium-fail (failed recovery) |
| **Parallel Execution Failure** | quick-success + quick-fail (parallel) → slow-success (skipped) |

---

## Implementation Notes

1. All workflows should be placed in `.github/workflows/` directory
2. Workflows are triggered via `workflow_dispatch` event
3. Authentication uses service account credentials (not user tokens)
4. Error handling ensures status is reported even on workflow failure
5. Use `if: always()` on final status reporting step to ensure it runs