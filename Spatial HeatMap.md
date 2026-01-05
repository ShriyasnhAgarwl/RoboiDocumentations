# Spatial Heatmap with BBox Coordinates

> **For: Fullstack Developer**  
> **Requirement:** True spatial heatmap showing WHERE people stand in camera view

---

## Overview

This document covers the logic for creating a **spatial density heatmap** using bounding box (bbox) coordinates from YOLO detections. Unlike simple zone counts, this shows the actual physical locations where people congregate within each camera's field of view.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPATIAL HEATMAP PIPELINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Camera Frame (1920x1080)                                       │
│  ┌──────────────────────────────┐                               │
│  │    ┌───┐     ┌───┐          │   YOLO detects people          │
│  │    │ P │     │ P │          │   Each has bbox:                │
│  │    │ 1 │     │ 2 │          │   (left, top, width, height)   │
│  │    └───┘     └───┘          │                                │
│  │         ┌───┐               │   Extract centroid:            │
│  │         │ P │               │   x = left + width/2           │
│  │         │ 3 │               │   y = top + height/2           │
│  │         └───┘               │                                │
│  └──────────────────────────────┘                               │
│              │                                                   │
│              ▼                                                   │
│  ┌──────────────────────────────┐                               │
│  │   Normalize to Grid (100x100)│                               │
│  │   grid_x = (centroid_x / frame_width) * 100                  │
│  │   grid_y = (centroid_y / frame_height) * 100                 │
│  └──────────────────────────────┘                               │
│              │                                                   │
│              ▼                                                   │
│  ┌──────────────────────────────┐                               │
│  │   Aggregate in ClickHouse    │                               │
│  │   count() per (grid_x, grid_y)│                              │
│  └──────────────────────────────┘                               │
│              │                                                   │
│              ▼                                                   │
│  ┌──────────────────────────────┐                               │
│  │   Render as Heatmap Canvas   │                               │
│  │   High density = Red         │                               │
│  │   Low density = Blue/Green   │                               │
│  └──────────────────────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## ClickHouse Queries

### Query 1: Extract Centroids from BBox Arrays

Your schema stores bbox as parallel arrays. To get person centroids:

```sql
SELECT
    cam_name,
    -- Calculate centroid X: left + width/2
    arrayMap(
        (l, w) -> toUInt16(l + w / 2),
        `detections.bbox_left`,
        `detections.bbox_width`
    ) AS centroid_x_array,
    
    -- Calculate centroid Y: top + height/2
    arrayMap(
        (t, h) -> toUInt16(t + h / 2),
        `detections.bbox_top`,
        `detections.bbox_height`
    ) AS centroid_y_array,
    
    event_timestamp
FROM camera_events
WHERE site_name = 'HEAD_OFFICE'
  AND cam_name = 'EMPLOYEE_AREA'
  AND toDate(toDateTime(event_timestamp)) = '2026-01-05'
  -- Filter only person detections
  AND has(`detections.label`, 'person')
```

### Query 2: Aggregate into Density Grid

```sql
-- Flatten arrays and aggregate into 100x100 grid
WITH 
    1920 AS frame_width,
    1080 AS frame_height,
    100 AS grid_size
SELECT
    -- Normalize to grid coordinates (0-99)
    intDiv(centroid_x * grid_size, frame_width) AS grid_x,
    intDiv(centroid_y * grid_size, frame_height) AS grid_y,
    count() AS density
FROM (
    SELECT
        arrayJoin(
            arrayMap(
                (l, w) -> toUInt16(l + w / 2),
                `detections.bbox_left`,
                `detections.bbox_width`
            )
        ) AS centroid_x,
        arrayJoin(
            arrayMap(
                (t, h) -> toUInt16(t + h / 2),
                `detections.bbox_top`,
                `detections.bbox_height`
            )
        ) AS centroid_y
    FROM camera_events
    WHERE site_name = %(site_name)s
      AND cam_name = %(cam_name)s
      AND event_timestamp >= %(start_ts)s
      AND event_timestamp < %(end_ts)s
      AND has(`detections.label`, 'person')
)
GROUP BY grid_x, grid_y
ORDER BY density DESC
```

---

## Materialized View for Performance

```sql
-- Pre-compute density grid per camera per hour
CREATE MATERIALIZED VIEW mv_spatial_density
ENGINE = SummingMergeTree()
ORDER BY (site_name, cam_name, hour, grid_x, grid_y)
AS
WITH 
    1920 AS frame_width,
    1080 AS frame_height,
    100 AS grid_size
SELECT
    site_name,
    cam_name,
    toStartOfHour(toDateTime(event_timestamp)) AS hour,
    intDiv(centroid_x * grid_size, frame_width) AS grid_x,
    intDiv(centroid_y * grid_size, frame_height) AS grid_y,
    count() AS density
FROM (
    SELECT
        site_name,
        cam_name,
        event_timestamp,
        arrayJoin(
            arrayMap((l, w) -> l + w / 2, `detections.bbox_left`, `detections.bbox_width`)
        ) AS centroid_x,
        arrayJoin(
            arrayMap((t, h) -> t + h / 2, `detections.bbox_top`, `detections.bbox_height`)
        ) AS centroid_y
    FROM camera_events
    WHERE has(`detections.label`, 'person')
)
GROUP BY site_name, cam_name, hour, grid_x, grid_y;
```

---

## API Endpoint

```python
@app.get("/api/v1/heatmap/{site_name}/{cam_name}")
async def get_spatial_heatmap(
    site_name: str,
    cam_name: str,
    date: str = Query(...),
    resolution: int = Query(100, ge=20, le=200)
):
    """
    Get spatial density grid for a camera.
    Returns sparse matrix of (x, y, intensity) for canvas rendering.
    """
    target_date = datetime.strptime(date, '%Y-%m-%d')
    start_ts = int(target_date.timestamp())
    end_ts = start_ts + 86400
    
    query = """
        SELECT grid_x, grid_y, sum(density) AS total
        FROM mv_spatial_density
        WHERE site_name = %(site_name)s
          AND cam_name = %(cam_name)s
          AND hour >= toDateTime(%(start_ts)s)
          AND hour < toDateTime(%(end_ts)s)
        GROUP BY grid_x, grid_y
        ORDER BY total DESC
        LIMIT 5000
    """
    
    rows = ch.execute(query, {
        'site_name': site_name, 'cam_name': cam_name,
        'start_ts': start_ts, 'end_ts': end_ts
    })
    
    # Normalize to 0-1 intensity
    max_density = max((r[2] for r in rows), default=1)
    
    cells = [
        {"x": r[0], "y": r[1], "v": round(r[2] / max_density, 3)}
        for r in rows
    ]
    
    return {
        "site_name": site_name,
        "cam_name": cam_name,
        "date": date,
        "resolution": 100,
        "frame_size": {"width": 1920, "height": 1080},
        "data": {"format": "sparse", "cells": cells}
    }
```

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

---

## Frontend Canvas Renderer

```javascript
class SpatialHeatmapRenderer {
    constructor(canvas, backgroundImage) {
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d');
        this.backgroundImage = backgroundImage;
        
        // Gradient: blue (cold) → green → yellow → red (hot)
        this.palette = this.createPalette();
    }

    createPalette() {
        const canvas = document.createElement('canvas');
        canvas.width = 256; canvas.height = 1;
        const ctx = canvas.getContext('2d');
        const gradient = ctx.createLinearGradient(0, 0, 256, 0);
        gradient.addColorStop(0, 'rgba(0, 0, 255, 0)');
        gradient.addColorStop(0.25, 'rgba(0, 255, 255, 0.5)');
        gradient.addColorStop(0.5, 'rgba(0, 255, 0, 0.7)');
        gradient.addColorStop(0.75, 'rgba(255, 255, 0, 0.85)');
        gradient.addColorStop(1, 'rgba(255, 0, 0, 1)');
        ctx.fillStyle = gradient;
        ctx.fillRect(0, 0, 256, 1);
        return ctx.getImageData(0, 0, 256, 1).data;
    }

    async render(apiUrl) {
        // Draw background (camera snapshot or floorplan)
        if (this.backgroundImage) {
            this.ctx.drawImage(this.backgroundImage, 0, 0, 
                this.canvas.width, this.canvas.height);
        }
        
        // Fetch heatmap data
        const response = await fetch(apiUrl);
        const data = await response.json();
        
        // Create alpha layer for heatmap
        const alphaCanvas = document.createElement('canvas');
        alphaCanvas.width = this.canvas.width;
        alphaCanvas.height = this.canvas.height;
        const alphaCtx = alphaCanvas.getContext('2d');
        
        const cellWidth = this.canvas.width / data.resolution;
        const cellHeight = this.canvas.height / data.resolution;
        const radius = Math.max(cellWidth, cellHeight) * 1.5;
        
        // Draw each point as radial gradient
        for (const cell of data.data.cells) {
            const x = (cell.x + 0.5) * cellWidth;
            const y = (cell.y + 0.5) * cellHeight;
            
            const gradient = alphaCtx.createRadialGradient(x, y, 0, x, y, radius);
            gradient.addColorStop(0, `rgba(0, 0, 0, ${cell.v})`);
            gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
            
            alphaCtx.fillStyle = gradient;
            alphaCtx.beginPath();
            alphaCtx.arc(x, y, radius, 0, Math.PI * 2);
            alphaCtx.fill();
        }
        
        // Colorize using palette
        this.colorize(alphaCtx);
        
        // Overlay on main canvas with transparency
        this.ctx.globalAlpha = 0.7;
        this.ctx.drawImage(alphaCanvas, 0, 0);
        this.ctx.globalAlpha = 1.0;
    }

    colorize(ctx) {
        const imageData = ctx.getImageData(0, 0, ctx.canvas.width, ctx.canvas.height);
        const pixels = imageData.data;
        
        for (let i = 0; i < pixels.length; i += 4) {
            const alpha = pixels[i + 3];
            if (alpha > 0) {
                const offset = alpha * 4;
                pixels[i] = this.palette[offset];
                pixels[i + 1] = this.palette[offset + 1];
                pixels[i + 2] = this.palette[offset + 2];
                pixels[i + 3] = Math.min(255, alpha * 2);
            }
        }
        
        ctx.putImageData(imageData, 0, 0);
    }
}

// Usage
const canvas = document.getElementById('heatmap-canvas');
const bgImage = document.getElementById('camera-snapshot');
const renderer = new SpatialHeatmapRenderer(canvas, bgImage);
renderer.render('/api/v1/heatmap/HEAD_OFFICE/EMPLOYEE_AREA?date=2026-01-05');
```

---

## Complete Data Flow

```
Camera Frame → YOLO Detection → BBox (left, top, width, height)
                    ↓
           Centroid Calculation
           x = left + width/2
           y = top + height/2
                    ↓
           Store in camera_events
           (detections.bbox_* arrays)
                    ↓
           Materialized View: mv_spatial_density
           Grid (100x100) aggregation
                    ↓
           API: GET /heatmap/{site}/{cam}?date=...
           Returns sparse matrix
                    ↓
           Canvas Renderer
           Overlay on camera view / floorplan
```

---

## Key Formulas

| Calculation | Formula |
|-------------|---------|
| **Centroid X** | `bbox_left + bbox_width / 2` |
| **Centroid Y** | `bbox_top + bbox_height / 2` |
| **Grid X** | `(centroid_x / frame_width) * grid_size` |
| **Grid Y** | `(centroid_y / frame_height) * grid_size` |
| **Intensity** | `cell_count / max_cell_count` |

---

# Person Tracking & Deduplication (Future)

## Problem

Same person detected in multiple frames = duplicate counts in heatmap.

## Solution: Unique Person ID Tracking

### Concept

```
Frame 1: Person at (100, 200) → Assign ID: P001
Frame 2: Person at (105, 203) → Match to P001 (same person moved)
Frame 3: New person at (500, 300) → Assign ID: P002
```

### Schema Addition

Your schema already has `detections.object_id`:
```sql
`detections.object_id` Array(UInt16)
```

Use this for tracking continuity within a video segment.

### Deduplication Query

```sql
-- Count unique persons, not detections
SELECT
    grid_x, grid_y,
    uniqExact(object_id) AS unique_persons  -- Instead of count()
FROM (
    SELECT
        intDiv(centroid_x * 100, 1920) AS grid_x,
        intDiv(centroid_y * 100, 1080) AS grid_y,
        arrayJoin(`detections.object_id`) AS object_id,
        arrayJoin(arrayMap((l, w) -> l + w/2, ...)) AS centroid_x,
        ...
)
GROUP BY grid_x, grid_y
```

### Cross-Camera Person Re-ID (Advanced)

For tracking same person across cameras:

1. **Face Embedding** → Store 512-dim vector per person
2. **Re-ID Model** → Compare embeddings across cameras
3. **Global Person ID** → Assign consistent ID across all cameras

Your schema has:
```sql
`recognition.identity_id` Array(UInt32)
```

Use this for cross-camera tracking when face is recognized.

---

## Developer Checklist

- [ ] Create `mv_spatial_density` materialized view
- [ ] Implement `GET /api/v1/heatmap/{site}/{cam}` endpoint
- [ ] Add canvas renderer to frontend
- [ ] (Future) Implement object_id deduplication
- [ ] (Future) Cross-camera Re-ID integration
