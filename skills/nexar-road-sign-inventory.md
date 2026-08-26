---
name: Build a road-sign inventory from Nexar Road Inventory
description: Search curated traffic-sign and road-asset detections for an area and pull the full record, including evidence imagery and the OSM road segment, for each one.
api: openapi/nexar-roadinventory-openapi.yml
operations: [NexarPlatformAPI_FindDetections, NexarPlatformAPI_GetDetection]
---

# Build a road-sign inventory

Use this to audit signage along a corridor, find missing or damaged assets, or keep a
map's sign layer fresh.

## Before you start

- `https://external.getnexar.com`, `Authorization: Bearer {token}`, scopes
  `detection:find`, `detection:get`, `image:get`. POST + JSON body for every call.

## Steps

1. **Search the corridor.** `NexarPlatformAPI_FindDetections`
   (`POST /api/roadinventory/v3/detections`) with a required `bounding_box`. Filter on
   `detection_types` for traffic signs, on `osm_road_types` to stay on the road class you
   care about, and on `detection_status` to pick up only what is `NEW` since your last run.
2. **Cap density.** Set `max_detections_per_h3` so an intersection with fifty signs does
   not consume the whole response budget. Nexar indexes on H3 at resolution 12.
3. **Fetch the full record.** `NexarPlatformAPI_GetDetection`
   (`POST /api/roadinventory/v3/detections/single`) for the detail of a single sign,
   including its pixel bounding box within the evidence frame.
4. **Join to your basemap** on the `apiOsmRoadSegmentApiElement` the detection carries —
   detections are already bound to OpenStreetMap road segments and road types, so you do
   not need to conflate geometry yourself.

## Rules

- **Sort with `sort_by` + `sort_direction`, size with `limit`.** No cursor is published;
  window your queries by area and time.
- **Errors:** 403 for insufficient permissions, 500 for a transient server problem, and
  the `default` response carries `runtimeError` `{error, code, message, details[]}`.
  Not RFC 9457.
- **Read-only.** Nothing here mutates provider state.
