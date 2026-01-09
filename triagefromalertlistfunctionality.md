# Triage Functionality Documentation

This document outlines the implementation of the Alert Triage workflow in the RoboI application.

## Overview
The triage functionality allows users to mark specific security/safety alerts as "reviewed" or "handled" by adding a comment. This action updates the event status in the database and prevents it from being flagged as an active critical issue in subsequent queries.

## Frontend Implementation

### Components
1.  **AlertsList** (`roboi-frontend/src/components/widgets/alerts/AlertsList.tsx`)
    -   Displays the list of alerts.
    -   Contains the "Mark as Triage" button for each alert.
    -   Manages the state for the selected alert to be triaged.
    -   Handles the API call to submit triage data.
2.  **TriageModal** (`roboi-frontend/src/components/widgets/alerts/TriageModal.tsx`)
    -   A modal component that appears when "Mark as Triage" is clicked.
    -   collects:
        -   **Comment/Notes** (Mandatory).
    -   Displays a caution warning and disclaimer.
3.  **ToastContext** (`roboi-frontend/src/components/ui/Toast/ToastContext.tsx`)
    -   Provides user feedback (Success/Error) after the API call.

### Workflow
1.  User clicks "Mark as Triage" on an alert card.
2.  `TriageModal` opens with the alert details.
3.  User enters a comment and confirms.
4.  `AlertsList` sends a `POST` request to `/api/v1/events/triage`.
5.  On success, a green Toast notification appears, and the modal closes.

## Backend Implementation

### JSON Schema
**TriageRequest** (`roboi-backend/app/models/schemas.py`)
```json
{
  "site_id": "string (ro001)",
  "event_timestamp": "int (Unix Epoch)",
  "triaged_by": "string (Username)",
  "triage_notes": "string (User comment)",
  "triage_timestamp": "float (Current Unix Epoch)"
}
```

### API Endpoint
-   **URL**: `/api/v1/events/triage`
-   **Method**: `POST`
-   **File**: `roboi-backend/app/api/endpoints/events.py`

### logic
1.  Receives the `TriageRequest` payload.
2.  Sanitizes input strings (notes, username) to prevent SQL injection.
3.  Executes an asynchronous ClickHouse mutation (`ALTER TABLE UPDATE`) to update the specific event row.
    -   Updates columns: `triaged_by`, `triage_notes`, `triage_timestamp`.
    -   Updates `event_status` to `'TRIAGED'`.
4.  Returns a success message `{"status": "ok"}`.

### Database
-   **Table**: `video_analytics_logs`
-   **Query**:
    ```sql
    ALTER TABLE video_analytics_logs
    UPDATE
        triaged_by = '...',
        triage_notes = '...',
        triage_timestamp = ...,
        event_status = 'TRIAGED'
    WHERE site_id = '...' AND event_timestamp = ...
    ```
