# Heatmap API Reference

> **Total APIs Needed: 2**

---

## API 1: Zone Dashboard

**Endpoint:**
```
GET /api/v1/dashboard/{site_name}?date=YYYY-MM-DD
```

**Purpose:** Zone-level occupancy counts, hourly activity, critical events.

**Response:**
```json
{
  "site_name": "HEAD_OFFICE",
  "date": "2026-01-05",
  "zones": {
    "EMPLOYEE_AREA": {"peak": 8, "events": 17, "intensity": "high"},
    "RECEPTION_AREA": {"peak": 6, "events": 312, "intensity": "medium"},
    "CAFETERIA": {"peak": 2, "events": 16, "intensity": "low"},
    "BOSS_CABIN": {"peak": 2, "events": 382, "intensity": "low"}
  },
  "hourly": [
    {"hour": 5, "count": 1, "intensity": "low"},
    {"hour": 10, "count": 8, "intensity": "high"},
    {"hour": 12, "count": 8, "intensity": "high"}
  ],
  "critical_events": {
    "UNAUTHORIZED_STRANGER": 23,
    "RESTRICTED_ACCESS": 18,
    "CROWD_VIOLATION": 3,
    "total_events": 2844
  }
}
```

**ClickHouse Query:**
```sql
-- Zones
SELECT cam_name, max(people_count), count() 
FROM camera_events WHERE site_name=? AND toDate(toDateTime(event_timestamp))=? 
GROUP BY cam_name;

-- Hourly
SELECT toHour(toDateTime(event_timestamp)), max(people_count) 
FROM camera_events WHERE ... GROUP BY 1 ORDER BY 1;

-- Events
SELECT trigger, count() FROM camera_events 
ARRAY JOIN event_triggers AS trigger WHERE ... GROUP BY trigger;
```

---

## API 2: Spatial Heatmap (BBox-Based)

**Endpoint:**
```
GET /api/v1/heatmap/{site_name}/{cam_name}?date=YYYY-MM-DD
```

**Purpose:** WHERE people stand in camera view (density grid from bbox centroids).

**Response:**
```json
{
  "site_name": "HEAD_OFFICE",
  "cam_name": "EMPLOYEE_AREA",
  "date": "2026-01-05",
  "resolution": 100,
  "frame_size": {"width": 1920, "height": 1080},
  "data": {
    "format": "sparse",
    "cells": [
      {"x": 45, "y": 72, "v": 1.0},
      {"x": 46, "y": 72, "v": 0.87},
      {"x": 23, "y": 45, "v": 0.65}
    ]
  }
}
```

**ClickHouse Query:**
```sql
-- Centroid from bbox: x = left + width/2
WITH 100 AS grid_size, 1920 AS w, 1080 AS h
SELECT
    intDiv((bbox_left + bbox_width/2) * grid_size, w) AS grid_x,
    intDiv((bbox_top + bbox_height/2) * grid_size, h) AS grid_y,
    count() AS density
FROM (
    SELECT 
        arrayJoin(`detections.bbox_left`) AS bbox_left,
        arrayJoin(`detections.bbox_width`) AS bbox_width,
        arrayJoin(`detections.bbox_top`) AS bbox_top,
        arrayJoin(`detections.bbox_height`) AS bbox_height
    FROM camera_events
    WHERE site_name=? AND cam_name=? AND toDate(...)=?
      AND has(`detections.label`, 'person')
)
GROUP BY grid_x, grid_y;
```

---

## Intensity Classification

```python
def classify(peak):
    if peak >= 7: return "high"   # Red
    if peak >= 4: return "medium" # Yellow
    return "low"                   # Green
```

---

## Summary

| API | Returns | Query Table |
|-----|---------|-------------|
| `/dashboard/{site}` | Zone counts + hourly | `camera_events` (people_count) |
| `/heatmap/{site}/{cam}` | Spatial density grid | `camera_events` (detections.bbox_*) |
