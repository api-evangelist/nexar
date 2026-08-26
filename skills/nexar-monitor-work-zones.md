---
name: Monitor work zones and consume them as WZDx
description: Find curated Nexar work-zone detections for an area, read one in full, and take the same data as a USDOT WZDx feed or as GeoJSON.
api: openapi/nexar-workzones-openapi.yml
operations: [NexarPlatformAPI_FindDetections, NexarPlatformAPI_GetDetection, CityStreamAPI_FindRealtimeDetections]
---

# Monitor work zones

Use this to keep a map, a routing engine or a DOT feed current with lane closures and
construction activity that Nexar's camera network detects from the road.

## Before you start

- Base URL `https://external.getnexar.com`; `Authorization: Bearer {token}`; scopes
  `detection:find`, `detection:get` and `image:get` for evidence imagery.
- All operations are POST with a JSON body.

## Steps

1. **Search.** `NexarPlatformAPI_FindDetections` (`POST /api/workzones/v3/detections`).
   `bounding_box` is **required** (`north_east` / `south_west`). Optional `filter` narrows by:
   - `detection_types` — e.g. construction zones vs road signs
   - `osm_road_types` — MOTORWAY, PRIMARY, …
   - `detection_status` — `NEW`, `EXISTING`, `REMOVED`
   - `detections_with_evidence` — only detections that carry a visual evidence frame
   Also useful: `detections_from_millis` (epoch cutoff) and `max_detections_per_h3`
   (cap per resolution-12 hexagon, so a dense downtown does not swamp the response).
2. **Drill in.** `NexarPlatformAPI_GetDetection`
   (`POST /api/workzones/v3/detections/single`) returns the full record for one work zone,
   including its evidence frame and its OSM road segment.
3. **Poll the live surface if you need minutes, not hours.**
   `CityStreamAPI_FindRealtimeDetections` (`POST /api/livefeed/v4/detections`,
   `openapi/nexar-livefeed-openapi.yml`) is the real-time projection of the same core.

## Take it in the format you already speak

The Live Feed operation declares **three** response media types on one operation. Set `Accept`:

- `application/json` — Nexar's native detection payload
- `application/geo+json` — an RFC 7946 `FeatureCollection`, drops straight into a GIS
- `application/vnd.wzdx` — a **USDOT Work Zone Data Exchange** feed

If your consumer already ingests WZDx, ask for `application/vnd.wzdx` and skip writing a
connector entirely. The WZDx object model is first-class in all four contracts
(`apiWZDxDetection`, `apiWZDxDetectionCollection`, `apiWZDxFeedInfo`, `apiWZDxDataSource`).

## Rules

- **Status is a lifecycle, not a flag.** `NEW` / `EXISTING` / `REMOVED` is how you learn a
  work zone went away. Reconcile on status; do not treat an absent detection as a removal.
- **403 means insufficient permissions** — the contract's own wording. It is an
  entitlement answer as often as an auth one.
- **Unexpected failures use the `default` response**, shaped as the grpc-gateway
  `runtimeError` `{error, code, message, details[]}`. There is no `Retry-After` and no
  rate-limit header on this product; the Work Zones contract does not even declare a 429.
- **Read-only.** No writes, no reversal, no idempotency key.
