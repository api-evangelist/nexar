---
name: Fetch road imagery for an area (Nexar CityStream VirtualCam)
description: Check whether Nexar has camera coverage for an area, then pull anonymized road frames for it, filtered by quality, road type, time of day and heading.
api: openapi/nexar-virtualcam-openapi.yml
operations: [VcamService_GetH3Coverage, VcamService_FindRawFrames, RoadItemService_FindRawFrames]
---

# Fetch road imagery for an area

Use this when you need real photographs of a road, an intersection or a corridor —
for asset inspection, change detection, or building a training set.

## Before you start

- Base URL is `https://external.getnexar.com`. Every call is a **POST with a JSON body**,
  including the searches — the filter object is the request.
- Authorize with `Authorization: Bearer {token}`. The token is minted by Nexar's Okta
  authorization server (`https://nexar.okta.com/oauth2/aus3qkg89t55hJZsT4x7`) through the
  Developers Portal. VirtualCam needs the `frame:get` and `image:get` scopes.
- An unauthorized call returns **403** with the plain-text body `RBAC: access denied` —
  not JSON. Do not try to parse it.
- Access is granted by Okta **group** membership, negotiated through a sales conversation.
  There is no self-serve tier. If you get 403 on a request that looks correct, the account
  is very likely not entitled to that product or that geography.

## Steps

1. **Check coverage first.** Call `VcamService_GetH3Coverage`
   (`POST /api/virtualcam/v4/coverage`) with the H3 indices or bounding box for your area.
   This tells you whether imagery exists before you spend a frame query. Note that a
   **403 here can also mean "the requested area is not authorized"**, which the contract
   states explicitly — treat it as an entitlement answer, not only an auth failure.
2. **Pull frames.** Call `VcamService_FindRawFrames` (`POST /api/virtualcam/v5/frames`).
   The request body (`FindRawFramesApiRequestParams`) takes:
   - `bounding_box` — `north_east` / `south_west`, each `{latitude, longitude}`
   - `h3_location` — H3 indices, as an alternative to a bounding box
   - `filters` — minimum quality, OSM road type, daylight state, heading, date range
   - `sort_by`
3. **Do not use the v4 frames operation for new work.** `RoadItemService_FindRawFrames`
   (`POST /api/virtualcam/v4/frames`) is the previous generation. Nexar announced in the
   contract itself that Frames V4 and Coverage V3 were deprecated on **2025-03-31**.
   It is still described in the published spec — prefer v5.

## Rules

- **Paginate with `limit`, and expect no cursor.** The published contract declares no page
  token, so plan your queries by area and time window rather than by paging deeply.
- **429 means rate limited.** VirtualCam is the only CityStream product that declares a
  429 ("API rate limit exceeded"), and Nexar publishes **no limit, no window and no
  `Retry-After` header**. Back off exponentially and treat throughput as unknown.
- **Errors are not RFC 9457.** A 400 returns `nexarBadRequest` `{status, error}`. Other
  failures may return the grpc-gateway `Status` shape `{code, message, details[]}`.
  Nothing returns `application/problem+json`.
- **This API is read-only.** Nothing you call here creates or changes state, so there is
  nothing to undo and no idempotency key to send.
